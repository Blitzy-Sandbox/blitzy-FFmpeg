# Integration Interfaces — How HLS Talks to External Systems

> **Commit Anchor:** All source references in this document are anchored to commit `566ad786` (full hash `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). Line numbers cited as `[<path>:L<start>-L<end>]` are valid at this commit. See [`../README.md`](../README.md) for the documentation-set-wide commit-anchor convention and citation format.

---

## Overview

### Plain-Language Summary

This document is a per-interface reference for every external touchpoint of the FFmpeg HLS muxer and demuxer. "External" here means anything outside the HLS C source files themselves: the file system, HTTP servers, libavformat's protocol layer, libavutil's AES primitives, the MPEG-TS and fMP4 sub-muxers, the sample-encryption transform, and the filename-templating helpers. The HLS code owns segmentation, playlist construction, and key install; it delegates everything else to one of the interfaces catalogued below.

The audience is engineers who need to understand or port the interface surface — what each external dependency expects, what HLS supplies, and where in `libavformat/hlsenc.c`, `libavformat/hls.c`, `libavformat/hls_sample_encryption.h`, `libavutil/aes.h`, and `libavformat/mpegts.h` the contact lives. Each section opens with a plain-language description suitable for an integrator, then descends to source-cited technical detail for an engineer planning a port.

The interfaces fall into the following groups, presented in this order:

1. File and HTTP AVIOContext writes (sections 2 and 3) — the byte-stream sinks that hold segment data and playlists.
2. Protocol handlers and HTTP detection (sections 4 and 5) — how HLS composes URLs whose scheme triggers a libavformat URL protocol handler.
3. Cryptographic interfaces (sections 6 and 7) — AES-128 full-segment encryption (muxer side) and HLS Sample Encryption (demuxer side).
4. Filename templating (sections 8 and 9) — `hls_segment_filename` placeholder expansion and `use_localtime` `strftime` expansion.
5. Sub-muxer integration (sections 10 and 11) — the meta-muxer relationship with `ff_mpegts_muxer` and `ff_mp4_muxer`.
6. HTTP DELETE channel and fMP4 init-segment resend (sections 12 and 13) — the two specialised HTTP exchanges outside the primary write path.

For internal lifecycle (when each interface is invoked) and ordering constraints, see [`pipeline-orchestration.md`](pipeline-orchestration.md) and [`process-flows.md`](process-flows.md). For struct field definitions referenced from this document (e.g., `HLSContext`, `VariantStream`, `HLSCryptoContext`), see [`data-model.md`](data-model.md). For decision logic that gates which interface is used (e.g., file vs HTTP, MPEG-TS vs fMP4), see [`codec-logic.md`](codec-logic.md).

---

## Interface — File AVIOContext Writes

### Plain-Language Summary

The HLS muxer writes every segment file, every per-variant `.m3u8` playlist, and every encryption-key file to disk through FFmpeg's `AVIOContext` abstraction. It never calls POSIX `open`/`write`/`close` directly; it goes through the framework's `s->io_open()` callback, which in turn delegates to whichever URL protocol handler matches the filename's scheme (for a bare filename with no `scheme://` prefix, that is the `file` protocol).

### Technical Detail

The wrapper that opens an output AVIOContext for any HLS-owned write is `hlsenc_io_open` at `[libavformat/hlsenc.c:L292-L311]`. It receives an `AVFormatContext *s`, the target `AVIOContext **pb` slot, a `filename`, and an option dictionary. In its first branch — entered when no AVIOContext is already open at `*pb`, or the filename is not HTTP-based, or `http_persistent` is zero — it delegates to the framework's open callback by calling `s->io_open(s, pb, filename, AVIO_FLAG_WRITE, options)` at `[libavformat/hlsenc.c:L299]`. The `io_open` function pointer is declared on `AVFormatContext` itself at `[libavformat/avformat.h:L1866-L1867]` and is assigned by the framework when the format context is created; HLS reads through this pointer rather than calling lower-level URL protocol code (`ffurl_open` or similar) directly. This indirection lets a host application substitute a custom open callback (for in-memory sinks, encrypted layers, or test doubles) without modifying any HLS code.

On error, `hlsenc_io_open` returns the error code from `s->io_open` (or from the persistent-reuse branch). Callers that set `hls->ignore_io_errors` typically treat a non-zero return as success — see `hls_delete_file` at `[libavformat/hlsenc.c:L520-L521]` for the canonical "swallow on error" pattern. Callers that demand strong durability (e.g., the initial `avformat_write_header` open) propagate the error upward.

The closure path is `hlsenc_io_close` at `[libavformat/hlsenc.c:L313-L331]`. For non-HTTP filenames, for non-persistent connections, or whenever encryption is active, it calls `ff_format_io_close(s, pb)` at `[libavformat/hlsenc.c:L321]`, which in turn invokes the framework's `io_close2` callback and flushes-and-frees the AVIOContext. The HTTP-persistent path of `hlsenc_io_close` is described in the next section.

Segment payload bytes do not flow through `hlsenc_io_open` directly. Instead, HLS opens a target AVIOContext per segment (the `vs->out` slot for the per-segment file in MPEG-TS mode, or the persistent `vs->out_single_file` slot when `HLS_SINGLE_FILE` is set), and the child sub-muxer (TS or fMP4) writes container payload bytes into that AVIOContext through `avio_write`, `avio_tell`, `avio_flush`, and similar generic AVIOContext APIs. The flush-and-publish pattern at the segment-cut boundary is visible in `hls_write_packet` where `av_write_frame(oc, NULL)` flushes any buffered data into `oc->pb` at `[libavformat/hlsenc.c:L2507]`, `avio_flush(oc->pb)` ensures the data reaches the underlying I/O at `[libavformat/hlsenc.c:L2510]`, and `hlsenc_io_close` finalises the segment file.

Three additional file-output paths use `hlsenc_io_open` directly: the AES-128 key file write inside `do_encrypt` (via the framework's `s->io_open` at `[libavformat/hlsenc.c:L702]`), the HTTP DELETE request (covered in a later section), and the fMP4 init segment write through `hls_init_file_resend` at `[libavformat/hlsenc.c:L2362-L2377]`.

### AVIOContext Slot Inventory

The HLS muxer holds AVIOContext pointers in several dedicated slots, each serving a distinct write target:

| Slot | Field | Declared at | Purpose |
|---|---|---|---|
| Master playlist writer | `HLSContext::m3u8_out` | `[libavformat/hlsenc.c:L259]` | The top-level `.m3u8` (master playlist) referencing each variant's per-variant playlist; opened by `hlsenc_io_open` at `[libavformat/hlsenc.c:L1388]` and closed at `[libavformat/hlsenc.c:L1524]` once per master-playlist publish. In byterange mode, the per-variant playlist also re-uses `hls->m3u8_out` (see `[libavformat/hlsenc.c:L1578]`). |
| Subtitle playlist writer | `HLSContext::sub_m3u8_out` | `[libavformat/hlsenc.c:L260]` | Writer for `.vtt`-bearing variant playlists when subtitle muxing is active. |
| Per-variant segment/playlist writer | `VariantStream::out` | `[libavformat/hlsenc.c]` (VariantStream struct) | Holds either the per-segment file AVIOContext (during a segment write) or the per-variant playlist AVIOContext (during playlist publish). Re-purposed across the segment-write/playlist-publish cycle. |
| Single-file output | `VariantStream::out_single_file` | `[libavformat/hlsenc.c:L127]` | Held open for the lifetime of the format context when `HLS_SINGLE_FILE` is set; receives all segment bytes concatenated into one file with `EXT-X-BYTERANGE` ranges in the playlist. |
| HTTP DELETE channel | `HLSContext::http_delete` | `[libavformat/hlsenc.c:L261]` | Reused dedicated context for issuing HTTP DELETE on expired segments. |

Each slot has its own open/close lifecycle managed by `hlsenc_io_open`/`hlsenc_io_close`. Reuse rules differ per slot: `out_single_file` is opened once and held; `m3u8_out` is opened-and-closed per master-playlist publish; `vs->out` is opened-and-closed per segment write and again per per-variant-playlist write; `http_delete` is opened-and-closed per DELETE request (or reused via HTTP persistence).

---

## Interface — HTTP AVIOContext Writes (with Persistent-Connection Reuse)

### Plain-Language Summary

When the output URL uses `http://` or `https://`, the HLS muxer can reuse a single TCP connection across many playlist and segment writes via the `http_persistent` option. Reuse converts the per-write open-connect-write-close cycle into a single long-lived connection over which each segment and playlist update is sent as a new HTTP request — significantly lower latency for live broadcasts and CDN ingest endpoints that expect chunked uploads.

### Technical Detail

The persistent-connection branch lives inside the same `hlsenc_io_open`/`hlsenc_io_close` wrappers used for plain file I/O. The branching condition in `hlsenc_io_open` is at `[libavformat/hlsenc.c:L298]`: when an AVIOContext already exists at `*pb`, the filename is HTTP-based, and `hls->http_persistent` is non-zero, the function skips a fresh `io_open` and enters the HTTP-reuse branch instead. Inside the branch, it retrieves the underlying URL context via `ffio_geturlcontext(*pb)` at `[libavformat/hlsenc.c:L302]` and calls `ff_http_do_new_request(http_url_context, filename)` at `[libavformat/hlsenc.c:L304]`. That function issues a new HTTP request on the existing TCP connection without tearing it down; the existing AVIOContext is repointed at the new request body. On request-setup failure the code falls back to a fresh open via `ff_format_io_close(s, pb)` at `[libavformat/hlsenc.c:L306]`, releasing the connection.

The closure half of the persistent path is in `hlsenc_io_close` at `[libavformat/hlsenc.c:L313-L331]`. The guard at `[libavformat/hlsenc.c:L320]` reads: when the filename is HTTP-based AND `http_persistent` is set AND neither `key_info_file` nor `encrypt` is active, take the persistent branch; otherwise fully close the context with `ff_format_io_close`. The persistent branch retrieves the URL context, flushes the AVIOContext with `avio_flush(*pb)` at `[libavformat/hlsenc.c:L326]`, and then calls `ffurl_shutdown(http_url_context, AVIO_FLAG_WRITE)` at `[libavformat/hlsenc.c:L327]`. `ffurl_shutdown` ends the request body half-stream so the server can complete its response, but does not close the TCP socket; the AVIOContext stays alive for the next call to `hlsenc_io_open` on the same `*pb` slot.

The explicit exclusion of key-info and encryption modes from the persistent branch reflects the `crypto:` pseudo-protocol's incompatibility with HTTP connection reuse: a `crypto:` URL wraps an inner HTTP URL with an encryption transform, and the wrapping layer holds the connection state. Reusing the underlying HTTP socket across encrypted segments without re-initialising the AES context would corrupt the cipher stream.

The `set_http_options` helper at `[libavformat/hlsenc.c:L333-L350]` adds `multiple_requests=1` to the option dictionary when `http_persistent` is set (`av_dict_set_int(options, "multiple_requests", 1, 0)` at `[libavformat/hlsenc.c:L345]`); this tells the HTTP protocol driver to keep the connection alive after the current request body completes.

### Failure Recovery on Reuse

When `ff_http_do_new_request` fails (e.g., the server closed the connection mid-stream, the previous request received a 5xx response that invalidated the transport, or the request body was rejected), the code at `[libavformat/hlsenc.c:L305-L306]` falls back to `ff_format_io_close(s, pb)` — the AVIOContext is torn down and the slot is cleared. The next `hlsenc_io_open` for that slot will take the cold-open path (the `!*pb` branch at `[libavformat/hlsenc.c:L298]`), opening a fresh HTTP connection.

The non-fall-through return at `[libavformat/hlsenc.c:L310]` propagates the error code (negative `AVERROR`) to the caller. Callers respect this error subject to `hls->ignore_io_errors`: when set, errors are swallowed and treated as success at the call site; when unset, errors propagate up through `hls_write_packet` (or whichever lifecycle hook is active) and abort the encoding.

### Use Cases for Persistent Connections

Persistent HTTP is the recommended configuration for two production scenarios:

- **Live CDN ingest** — A long-running live broadcast produces dozens of segments per minute (with `hls_time=2` and a multi-bitrate ladder, easily a hundred PUTs per minute). Each segment PUT plus each playlist PUT plus each old-segment DELETE is a separate HTTP transaction over the same TCP connection. Tearing down and re-establishing the TCP+TLS handshake on every transaction would dominate latency and CPU.
- **Authenticated origin push** — When the upstream ingest endpoint requires Basic/Bearer authentication or mutual TLS, the per-request handshake cost compounds with the cryptographic handshake cost. Connection reuse amortises the handshake cost over the entire publish window.

For VOD encoding into a local filesystem, `http_persistent` has no effect (the file protocol does not implement the persistence semantics). For HTTPS endpoints, the same persistence applies — the underlying TLS protocol layer keeps the TLS session alive alongside the TCP connection.

---

## Interface — Protocol Handlers (`file://`, `http://`, `https://`, `crypto:`)

### Plain-Language Summary

HLS does not implement these protocols itself — it composes URLs whose scheme triggers the corresponding libavformat URL protocol handler. The HLS code's job is to construct the right URL string for the right write, set any handler-specific dictionary options (HTTP method, headers, encryption key), and pass the URL through `s->io_open`. The protocol layer handles the wire bytes.

### Technical Detail

Four protocol scheme families are in use:

- **`file://` (or no scheme prefix)**. A bare filename like `out0.ts` is interpreted by the `file` URL protocol. This is the default path for segment files, per-variant playlists, the master playlist, the encryption key file, the fMP4 init segment, and the `.tmp` temp files used by `HLS_TEMP_FILE` mode. The temp-file rename is implemented by `ff_rename(oc->url, final_filename, s)` at `[libavformat/hlsenc.c:L1309]` inside `hls_rename_temp_file` at `[libavformat/hlsenc.c:L1300]`; the atomic rename relies on POSIX filesystem semantics provided by the `file` protocol handler.

- **`http://` and `https://`**. HLS detects these via `ff_is_http_proto` (see the next section) and switches behaviour in three places: connection reuse in `hlsenc_io_open`/`_close`, default method PUT in `set_http_options` at `[libavformat/hlsenc.c:L340]`, and HTTP DELETE method override in `hls_delete_file` at `[libavformat/hlsenc.c:L515]`. The HLS muxer never sends raw HTTP — it only composes the URL and sets the dictionary options that the libavformat HTTP protocol driver consumes.

- **`crypto:`** is not a network protocol — it is a libavformat URL prefix that wraps an inner URL with an AES-128 encryption transform applied by `libavutil`'s AES API. The HLS muxer composes `crypto:<inner-url>` strings at three sites: the single-file temp filename in `hls_start` at `[libavformat/hlsenc.c:L1813]` (`vs->basename_tmp = av_asprintf("crypto:%s.tmp", oc->url)`), the per-segment filename in `hls_write_packet` at `[libavformat/hlsenc.c:L2556]` (`filename = av_asprintf("crypto:%s", oc->url)`), and the trailer-time write in `hls_write_trailer` at `[libavformat/hlsenc.c:L2755]`. Before the open, HLS adds two dictionary entries that the `crypto:` protocol driver reads to configure the cipher: `av_dict_set(&options, "encryption_key", vs->key_string, 0)` at `[libavformat/hlsenc.c:L2554]` and `av_dict_set(&options, "encryption_iv", vs->iv_string, 0)` at `[libavformat/hlsenc.c:L2555]` for per-segment cuts; the trailer write performs the same setup at `[libavformat/hlsenc.c:L2753-L2754]`.

The HLS muxer composes the URL; libavformat's protocol layer resolves the scheme prefix and dispatches to the correct handler.

### Protocol Discovery in `hls_delete_old_segments`

For the old-segment cleanup path that may issue HTTP DELETE, the muxer cannot rely on the original output URL's scheme — old segments may have been written to a different absolute URL than what the playlist URL implies. The cleanup driver `hls_delete_old_segments` calls `avio_find_protocol_name(segment->filename)` to determine the scheme of each expired segment URL, then passes the scheme name as the `proto` argument to `hls_delete_file` at `[libavformat/hlsenc.c:L507-L529]`. The `proto` argument gates the HTTP vs `unlink` branch at `[libavformat/hlsenc.c:L510]`.

### URL Composition Concerns

The HLS muxer composes URLs by string concatenation (`av_asprintf`, `av_strlcat`, `snprintf`), not by URL-aware composition. Three caveats follow:

- Percent-encoding is the caller's responsibility. If `hls_segment_filename` produces an unencoded URL component (e.g., a basename containing spaces), the protocol handler may reject it.
- `crypto:` URLs are layered prefixes — `crypto:http://host/seg.ts` is valid and routes through the `crypto:` driver, which then opens `http://host/seg.ts` as the inner URL. The HLS muxer composes this layering by prepending `crypto:` to the already-composed inner URL.
- Filename templating runs before URL composition. The `%v`, `%d`, `%s`, `%t` substitutions happen on the raw template; the result is then passed to `hlsenc_io_open` as a fully-formed URL.

---

## Interface — `ff_is_http_proto` Detection

### Plain-Language Summary

A single libavformat helper, `ff_is_http_proto`, decides whether a URL is HTTP-based. Three places in the HLS muxer call it to switch behaviour: opening a write, closing a write, and computing HTTP option dictionary entries.

### Technical Detail

`ff_is_http_proto(filename)` returns non-zero when the URL begins with `http://` or `https://` (and zero otherwise). It is invoked at exactly four sites in the muxer:

1. `[libavformat/hlsenc.c:L296]` inside `hlsenc_io_open` — used to compute `http_base_proto = filename ? ff_is_http_proto(filename) : 0`. The result gates the persistent-connection branch at `[libavformat/hlsenc.c:L298]`.
2. `[libavformat/hlsenc.c:L316]` inside `hlsenc_io_close` — same computation, gates the persistent-shutdown branch at `[libavformat/hlsenc.c:L320]`.
3. `[libavformat/hlsenc.c:L335]` inside `set_http_options` — gates the default-PUT branch at `[libavformat/hlsenc.c:L339-L341]`.
4. `[libavformat/hlsenc.c:L2875]` inside `hls_init` — pre-computes `http_base_proto` for downstream HTTP-aware initialisation (e.g., the `master_pl_name` write).

`set_http_options` at `[libavformat/hlsenc.c:L333-L350]` is the only site that adds option-dictionary entries (as opposed to flipping internal flow):

- When `c->method` is non-NULL (user override), `av_dict_set(options, "method", c->method, 0)` at `[libavformat/hlsenc.c:L338]` propagates the user-chosen HTTP verb.
- Else when `http_base_proto` is true, `av_dict_set(options, "method", "PUT", 0)` at `[libavformat/hlsenc.c:L340]` sets the default PUT verb expected by HLS-aware ingest endpoints.
- `c->user_agent` (if set) becomes the `user_agent` entry at `[libavformat/hlsenc.c:L343]`.
- `c->http_persistent` (if set) enables `multiple_requests=1` at `[libavformat/hlsenc.c:L345]` so the HTTP protocol driver keeps the TCP connection alive.
- `c->timeout` (if non-negative) becomes the `timeout` entry at `[libavformat/hlsenc.c:L347]`.
- `c->headers` (if set) becomes the `headers` entry at `[libavformat/hlsenc.c:L349]`, supplying user-specified HTTP request headers verbatim.

The function is void-returning; the option dictionary is mutated in place. The dictionary is consumed by the HTTP protocol driver inside `s->io_open` when the URL is opened.

---

## Interface — AES-128 Crypto Pipeline

### Plain-Language Summary

When `hls_enc=1` or `hls_key_info_file=<path>` is set, the HLS muxer arranges for each segment to be AES-128-encrypted. The actual block cipher runs inside libavformat's `crypto:` URL protocol; the HLS muxer's responsibilities are limited to generating or fetching the 16-byte key, deriving or accepting the 16-byte IV, writing the key file to disk (in `hls_enc` mode), composing the per-segment `crypto:<inner-url>` filename, and emitting the `EXT-X-KEY` line in the playlist.

Encrypted fMP4 mode is not supported: `hls_start` returns `AVERROR_PATCHWELCOME` at `[libavformat/hlsenc.c:L1770-L1771]` when both `segment_type=fmp4` and encryption is requested.

### Technical Detail

Two key-install code paths exist, selected by which option the user supplied:

**Inline key generation — `do_encrypt`** at `[libavformat/hlsenc.c:L641-L711]`. Activated by `hls_enc=1`. Constructs the on-disk key URI by appending `.key` to either `hls->master_m3u8_url` or `s->url` (the basename source at `[libavformat/hlsenc.c:L648]`). If the user supplied `hls_enc_key_url`, `hls->key_file` and `hls->key_uri` are copied from it; otherwise both default to the `.key` basename. When `hls->iv_string` is empty, an IV defaults to 16 zero bytes with the 64-bit sequence number written into the low half via `AV_WB64(iv + 8, vs->sequence)` at `[libavformat/hlsenc.c:L671]`; if the user supplied `hls_enc_iv`, that 16-byte value is used directly. The key itself is taken from `hls->key` when supplied, or generated by `av_random_bytes(key, sizeof(key))` at `[libavformat/hlsenc.c:L692]`. The key is written to disk via `s->io_open(s, &pb, hls->key_file, AVIO_FLAG_WRITE, &options)` at `[libavformat/hlsenc.c:L702]`, an `avio_seek(pb, 0, SEEK_CUR)` at `[libavformat/hlsenc.c:L706]`, and `avio_write(pb, key, KEYSIZE)` at `[libavformat/hlsenc.c:L707]`, then closed with `avio_close(pb)` at `[libavformat/hlsenc.c:L708]`.

**External key info file — `hls_encryption_start`** at `[libavformat/hlsenc.c:L714-L771]`. Activated by `hls_key_info_file=<path>`. Opens the key info file via `s->io_open` with `AVIO_FLAG_READ`, then reads three line-delimited fields using `ff_get_line` at `[libavformat/hlsenc.c:L731]` (key URI for the playlist `EXT-X-KEY:URI=` value), `[libavformat/hlsenc.c:L734]` (key file path on disk), and `[libavformat/hlsenc.c:L737]` (optional IV hex string). After validating the URI and file are non-empty, it re-opens the key file path with `AVIO_FLAG_READ` and reads exactly 16 raw bytes via `avio_read(pb, key, sizeof(key))` at `[libavformat/hlsenc.c:L760]`. The 16 bytes are converted to a 32-char hex string by `ff_data_to_hex(vs->key_string, key, sizeof(key), 0)` for use in the playlist line and as the `encryption_key` dictionary value.

Both paths converge on the same downstream wiring. Two sites compose the `crypto:` URL: the per-segment cut inside `hls_write_packet` at `[libavformat/hlsenc.c:L2553-L2556]` (`encryption_key`/`encryption_iv` dictionary entries at `[libavformat/hlsenc.c:L2554-L2555]`, `crypto:` URL at `[libavformat/hlsenc.c:L2556]`) and the trailer-time write inside `hls_write_trailer` at `[libavformat/hlsenc.c:L2752-L2755]`. When `hlsenc_io_open` opens the composed filename, the libavformat protocol resolver dispatches to the `crypto:` URL protocol driver, which strips the prefix to recover the inner URL, opens the inner URL through its own protocol handler, and wraps the resulting AVIOContext with an AES-128-CBC transform.

The block cipher itself is libavutil's AES API. The `crypto:` URL protocol driver (not the HLS code itself) calls `av_aes_alloc` at `[libavutil/aes.h:L41]` to allocate a context, `av_aes_init(struct AVAES *a, const uint8_t *key, int key_bits, int decrypt)` at `[libavutil/aes.h:L51]` with `key_bits=128` and `decrypt=0` (encrypt mode), and `av_aes_crypt(struct AVAES *a, uint8_t *dst, const uint8_t *src, int count, uint8_t *iv, int decrypt)` at `[libavutil/aes.h:L63]` for each 16-byte block of segment payload with the IV passed through to chain CBC blocks across the segment. The HLS source file does not include `libavutil/aes.h` directly — the cipher contract is mediated entirely by the `crypto:` URL prefix and the two dictionary entries.

`KEYSIZE` is `16` (declared in `libavformat/hlsenc.c`); the AES-128 key length and IV length are both 16 bytes, and the 32-char hex string stored in `vs->key_string` and `vs->iv_string` is the textual form emitted into `EXT-X-KEY` lines.

### Periodic Rekey

When the `HLS_PERIODIC_REKEY` flag (bit 12, at `[libavformat/hlsenc.c:L110]`) is set in `hls->flags`, the key-info file is re-read at the start of every segment. The check is at `[libavformat/hlsenc.c:L1779]`:

```c
if (!vs->encrypt_started || (c->flags & HLS_PERIODIC_REKEY)) {
```

When this condition holds, `hls_encryption_start` is invoked again at `[libavformat/hlsenc.c:L1781]`, re-parsing the three-line key info file. If the operator has updated the on-disk key info (new URI, new key file path, new IV), the next segment will be encrypted with the new key and `hls_window` will emit a fresh `EXT-X-KEY` line in the playlist before the affected segment's `EXTINF`. Without `HLS_PERIODIC_REKEY`, `hls_encryption_start` runs only once on first segment start, and the same key is used for every subsequent segment.

### `EXT-X-KEY` Playlist Emission

The playlist emission for the AES-128 key info happens inside `hls_window` at `[libavformat/hlsenc.c:L1601-L1609]`. For each segment in `vs->segments`, the muxer compares the segment's stored key URI and IV against the running per-iteration key URI and IV. When they differ (either initial install or a rekey), it emits a fresh `EXT-X-KEY:METHOD=AES-128,URI="..."` line at `[libavformat/hlsenc.c:L1603]`, optionally appending `,IV=0x<hex>` at `[libavformat/hlsenc.c:L1604-L1605]`. The IV is omitted from the playlist line when `vs->iv_string` is empty (which happens only when the user did not supply `hls_enc_iv` and the IV defaulted to the sequence-number-based value).

Each `HLSSegment` records its own key URI and IV (`en->key_uri`, `en->iv_string`); the playlist iterator emits a new `EXT-X-KEY` line only when those values change between adjacent segments. This minimises the playlist size for steady-state encryption while still supporting per-segment rekey when `HLS_PERIODIC_REKEY` rotates the key on every cut.

### CBC Mode and IV Chaining

The `crypto:` URL protocol driver runs AES-128-CBC. CBC mode requires an IV for each segment; within a segment, CBC chains 16-byte blocks together (the cipher block N XORs with the plaintext of block N+1 before encryption). The IV resets to the per-segment IV at the start of each segment write — there is no cross-segment IV chaining. PKCS#7 padding is applied by the `crypto:` driver internally; the HLS muxer's role is limited to supplying the key and IV via the dictionary entries.

---

## Interface — HLS Sample Encryption Pipeline

### Plain-Language Summary

HLS Sample Encryption is a different scheme from segment-level AES-128: individual samples (audio or video frames) inside an otherwise-cleartext MPEG-TS payload are encrypted in place. This is demuxer-side functionality only — the FFmpeg HLS muxer does not produce Sample-Encrypted output. The demuxer recognises Sample-Encrypted streams by their MPEG-TS stream-type values and applies the per-frame decryption transform before delivering AVPackets upstream.

### Technical Detail

The decrypt entry point is `ff_hls_senc_decrypt_frame(enum AVCodecID codec_id, HLSCryptoContext *crypto_ctx, AVPacket *pkt)` declared at `[libavformat/hls_sample_encryption.h:L63]`. It is given an `HLSCryptoContext` containing the AES key and IV plus an allocated `AVAES *aes_ctx`, and an `AVPacket` whose payload is mutated in place after decryption.

The crypto context struct is at `[libavformat/hls_sample_encryption.h:L43-L47]`:

```c
typedef struct HLSCryptoContext {
    struct AVAES    *aes_ctx;
    uint8_t         key[16];
    uint8_t         iv[16];
} HLSCryptoContext;
```

For audio streams, a per-stream audio setup info block conveys the codec tag, priming sample count, version, and a short (≤ 10-byte) codec-specific configuration payload. The setup-info struct is at `[libavformat/hls_sample_encryption.h:L49-L56]`:

```c
typedef struct HLSAudioSetupInfo {
    enum AVCodecID      codec_id;
    uint32_t            codec_tag;
    uint16_t            priming;
    uint8_t             version;
    uint8_t             setup_data_length;
    uint8_t             setup_data[HLS_MAX_AUDIO_SETUP_DATA_LEN + AV_INPUT_BUFFER_PADDING_SIZE];
} HLSAudioSetupInfo;
```

Two helpers populate and apply the setup info: `ff_hls_senc_read_audio_setup_info(HLSAudioSetupInfo *info, const uint8_t *buf, size_t size)` at `[libavformat/hls_sample_encryption.h:L59]` parses a raw byte buffer into the struct, and `ff_hls_senc_parse_audio_setup_info(AVStream *st, HLSAudioSetupInfo *info)` at `[libavformat/hls_sample_encryption.h:L61]` propagates the parsed values into the destination `AVStream` (codec tag, codec extradata, AAC priming, etc.).

Two constants bound the buffers: `HLS_MAX_ID3_TAGS_DATA_LEN = 138` at `[libavformat/hls_sample_encryption.h:L40]` (maximum length of the ID3-priv-tag payload carrying the setup info inside the TS) and `HLS_MAX_AUDIO_SETUP_DATA_LEN = 10` at `[libavformat/hls_sample_encryption.h:L41]` (maximum length of the codec-specific tail). The `setup_data` array in `HLSAudioSetupInfo` adds `AV_INPUT_BUFFER_PADDING_SIZE` bytes of trailing padding so libavcodec parsers can safely over-read.

Stream-type recognition lives in the MPEG-TS demuxer. Four stream-type values are declared in `libavformat/mpegts.h` at `[libavformat/mpegts.h:L177-L180]`:

| Constant | Value | Codec |
|---|---|---|
| `STREAM_TYPE_HLS_SE_VIDEO_H264` | `0xdb` | H.264 video, sample-encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_AAC`  | `0xcf` | AAC audio, sample-encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_AC3`  | `0xc1` | AC3 audio, sample-encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_EAC3` | `0xc2` | E-AC3 audio, sample-encrypted |

When the MPEG-TS sub-demuxer encounters one of these values in a PMT descriptor, it routes the stream's elementary-stream payload through `ff_hls_senc_decrypt_frame` before passing the resulting cleartext packet up to the HLS demuxer. The HLS demuxer in turn populates `crypto_ctx->key` and `crypto_ctx->iv` from the `EXT-X-KEY` line at playlist-parse time and lazily allocates `crypto_ctx->aes_ctx` on first use.

### Demuxer-Side Integration Points

The HLS demuxer integrates Sample Encryption at four code sites:

- **Audio-setup-info population during ID3 parsing.** The MPEG-TS PES payload of a Sample-Encrypted audio stream begins with an ID3 `priv` tag carrying the codec setup info. The HLS demuxer's ID3 parser at `[libavformat/hls.c:L1161]` accepts an `HLSAudioSetupInfo *` out-parameter and invokes `ff_hls_senc_read_audio_setup_info(audio_setup_info, priv->data, priv->datasize)` at `[libavformat/hls.c:L1181]` when a setup-info `priv` tag is found.
- **AVStream extradata propagation.** Once the playlist has identified a Sample-Encrypted audio stream and the codec_id is determined, the demuxer invokes `ff_hls_senc_parse_audio_setup_info(pls->ctx->streams[0], &pls->audio_setup_info)` at `[libavformat/hls.c:L2423]` to inject the parsed codec_tag, priming sample count, and codec-specific extradata into the destination AVStream.
- **AES context allocation.** First-use allocation of the AES key schedule lives at `[libavformat/hls.c:L2380]`: `c->crypto_ctx.aes_ctx = av_aes_alloc();`. The schedule is initialised lazily and reused across many frames.
- **Per-frame decrypt.** The hot path that decrypts each Sample-Encrypted frame is at `[libavformat/hls.c:L2600-L2604]`: when the active segment's `seg->key_type == KEY_SAMPLE_AES` and the inner format is not the MOV demuxer, the demuxer copies the per-segment IV from `seg->iv` and the key from `pls->key` into the shared crypto context, then calls `ff_hls_senc_decrypt_frame(codec_id, &c->crypto_ctx, pls->pkt)` at `[libavformat/hls.c:L2604]`. The packet payload is mutated in place; the caller's stream-index and PTS are preserved.

### Sample-Encrypted Codec Support

`ff_hls_senc_decrypt_frame` is codec-aware: the `codec_id` argument selects the per-codec framing rules (e.g., AAC has access-unit boundaries inside an ADTS frame; H.264 has NAL-unit framing inside a Annex B byte stream). The current implementation supports AAC, AC3, EAC3 (audio) and H.264 (video) — matching the four `STREAM_TYPE_HLS_SE_*` constants. Other codec_id values are rejected; the demuxer falls back to passing the cleartext packet through unmodified.

The AC3 and EAC3 audio cases share the AAC code path with codec-specific framing offsets. The assertion at `[libavformat/hls.c:L2342-L2344]` enforces that only these three audio codec_ids reach the Sample-Encrypted audio AVStream wiring.

---

## Interface — `hls_segment_filename` Templating

### Plain-Language Summary

The `hls_segment_filename` option supports printf-style placeholders that the muxer substitutes at segment-cut time. The supported placeholders are `%d` (sequence number), `%v` (variant index or variant name), `%s` (segment size in bytes), and `%t` (segment duration in microseconds). The `%s` and `%t` substitutions are gated by `hls_flags=second_level_segment_size` and `=second_level_segment_duration` and require `use_localtime=1`.

### Technical Detail

Two substitution helpers implement the placeholder expansion:

- **String substitution** — `replace_str_data_in_filename(char **s, const char *filename, char placeholder, const char *datastring)` at `[libavformat/hlsenc.c:L382-L420]`. Scans the source filename byte by byte, copies non-`%` characters verbatim, recognises `%%` as a literal `%`, and replaces `%<placeholder>` with the supplied data string. Returns the number of substitutions made (≥ 1 for a valid template; 0 indicates a missing placeholder).

- **Integer substitution** — `replace_int_data_in_filename(char **s, const char *filename, char placeholder, int64_t number)` at `[libavformat/hlsenc.c:L422-L468]`. Same scan strategy, but recognises optional decimal width digits between `%` and the placeholder (e.g., `%03d` for zero-padded three-digit output). Used for sequence numbers, variant indices, size, and duration.

The single-pass orchestrator that selects which substitution to apply is `sls_flags_filename_process(struct AVFormatContext *s, HLSContext *hls, VariantStream *vs, double duration, int64_t pos, int64_t size)` at `[libavformat/hlsenc.c:L908]`. It is invoked from `hls_append_segment` at `[libavformat/hlsenc.c:L1066]` after a segment is finalised but before its filename is added to the playlist. The activation guard at `[libavformat/hlsenc.c:L912]` requires `HLS_SECOND_LEVEL_SEGMENT_SIZE` or `HLS_SECOND_LEVEL_SEGMENT_DURATION` to be set in `hls->flags`. When `HLS_SECOND_LEVEL_SEGMENT_SIZE` is set, `replace_int_data_in_filename` substitutes `%s` with `pos + size` (cumulative byte position) at `[libavformat/hlsenc.c:L921]`. When `HLS_SECOND_LEVEL_SEGMENT_DURATION` is set, `replace_int_data_in_filename` substitutes `%t` with `(int64_t)round(duration * HLS_MICROSECOND_UNIT)` at `[libavformat/hlsenc.c:L933]`.

The sequence-number placeholder `%d` is handled separately — not by `sls_flags_filename_process` but by `sls_flag_use_localtime_filename(AVFormatContext *oc, HLSContext *c, VariantStream *vs)` at `[libavformat/hlsenc.c:L998]`, which is invoked from `hls_start` at `[libavformat/hlsenc.c:L1718]`. Inside, `replace_int_data_in_filename` substitutes `%d` with `vs->sequence` at `[libavformat/hlsenc.c:L1002]`. The flag `HLS_SECOND_LEVEL_SEGMENT_INDEX` gates this path.

The three `HLS_SECOND_LEVEL_*` flag values are declared at `[libavformat/hlsenc.c:L106-L108]`:

- `HLS_SECOND_LEVEL_SEGMENT_INDEX = (1 << 8)` — enables `%d` index substitution.
- `HLS_SECOND_LEVEL_SEGMENT_DURATION = (1 << 9)` — enables `%t` duration substitution.
- `HLS_SECOND_LEVEL_SEGMENT_SIZE = (1 << 10)` — enables `%s` size substitution.

The variant placeholder `%v` is handled at variant-list parse time by `format_name` (using either the string or integer helper depending on whether the user supplied a `varname` in `var_stream_map`) at `[libavformat/hlsenc.c:L1950-L1955]`. This is fixed-up-front substitution: each variant's `basename` is materialised once during `update_variant_stream_info`, not at every segment cut.

### Validation and Error Handling

When a placeholder is required but not present in the template, the substitution helpers return zero (no substitutions made), and the calling code treats this as `AVERROR(EINVAL)`. The diagnostic messages cite the offending template, e.g., `[libavformat/hlsenc.c:L922-L926]` for the size-substitution failure ("Invalid second level segment filename template '%s', you can try to remove second_level_segment_size flag\n"). The duration-substitution failure path at `[libavformat/hlsenc.c:L936-L940]` produces an analogous diagnostic for the `t` placeholder.

The `replace_int_data_in_filename` width parser at `[libavformat/hlsenc.c:L422-L468]` accepts decimal width digits between `%` and the placeholder, then formats the substituted value via `snprintf` with the width interpreted as a `%0<N>d` style (zero-padding with minimum field width). Negative widths and width-modifier characters other than digits are unsupported.

### `current_segment_final_filename_fmt` State

The second-level filename substitution is gated by `vs->current_segment_final_filename_fmt` (non-empty), checked at `[libavformat/hlsenc.c:L913]`. This field is populated at segment-cut time by the `hls_start` path and captures the post-strftime, post-variant-substitution filename template that still contains the unsubstituted `%s` or `%t` placeholders. The second-level processing runs after the segment file is written (and the size and duration are known), then the playlist entry is built from the post-substitution filename.

The presence of a placeholder in the template is not sufficient — the corresponding flag (`HLS_SECOND_LEVEL_SEGMENT_SIZE` or `HLS_SECOND_LEVEL_SEGMENT_DURATION`) must also be set. The template is permitted to contain `%s` without the size flag (in which case `%s` is left literal); a stricter behaviour is enforced at flag-setup-time validation, not here.

---

## Interface — `use_localtime` `strftime` Expansion

### Plain-Language Summary

When `strftime=1` (the AVOption name for `use_localtime=1`), the segment filename is treated as a `strftime(3)` format string and expanded with the wall-clock time at the moment of segment creation. The companion option `strftime_mkdir=1` (the AVOption name for `use_localtime_mkdir=1`) auto-creates intermediate directory components.

### Technical Detail

The expansion helper is `strftime_expand(const char *fmt, char **dest)` at `[libavformat/hlsenc.c:L269-L290]`. It allocates a `MAX_URL_SIZE` buffer with `av_mallocz`, captures the current time via `time(&now0)`, converts to local time with `localtime_r(&now0, &tmpbuf)`, and applies the supplied `fmt` via `strftime(buf, MAX_URL_SIZE, fmt, tm)`. A return value of zero from `strftime` indicates the format produced an empty string (treated as `AVERROR(EINVAL)`); otherwise the buffer pointer is returned via `*dest`.

The invocation site is in `hls_start` at `[libavformat/hlsenc.c:L1711]`:

```c
r = strftime_expand(vs->basename, &expanded);
```

It runs only when `c->use_localtime` is true (`[libavformat/hlsenc.c:L1707]`). The expanded URL is set on the child `oc` via `ff_format_set_url(oc, expanded)` at `[libavformat/hlsenc.c:L1716]`, replacing the templated basename with its time-expanded form.

When `c->use_localtime_mkdir` is also set, the directory path is extracted via `av_dirname` and passed to `ff_mkdir_p(dir)` at `[libavformat/hlsenc.c:L1729]`; failures other than `EEXIST` propagate as `AVERROR(errno)`. This lets a `strftime` template that expands into nested date-based directories (`%Y/%m/%d/%H/segment_%d.ts`) work out of the box without a separate `mkdir -p` step.

When the user does not supply `hls_segment_filename`, `hls_init` falls back to a default `strftime` pattern via `get_default_pattern_localtime_fmt(s)` (declared at `[libavformat/hlsenc.c:L1860]`, invoked at `[libavformat/hlsenc.c:L2880]`). The default pattern includes wall-clock components plus a sequence-number placeholder so unique filenames are guaranteed even without user templating.

The fMP4 init filename can itself be a `strftime` template — `strftime_expand` is invoked a second time at `[libavformat/hlsenc.c:L3044]` to expand `hls->fmp4_init_filename` when the init segment is named for write.

### Buffer Size and Thread Safety

The expansion buffer is sized to `MAX_URL_SIZE` (a libavformat-wide constant for URL lengths). If the user-supplied `strftime` pattern expands to more than `MAX_URL_SIZE - 1` bytes, `strftime` returns 0 (per POSIX semantics) and the helper returns `AVERROR(EINVAL)`. There is no automatic resize-and-retry; pathological format strings (e.g., enormous numeric width specifiers) fail loudly rather than silently truncate.

The helper uses `localtime_r` rather than `localtime` so the conversion is thread-safe even though the HLS muxer itself is single-threaded per `AVFormatContext`. The time zone applied is whatever `localtime_r` returns for the current process locale (typically `TZ`-derived or the system default). UTC expansion requires the user to supply a UTC format like `%Y-%m-%dT%H:%M:%SZ` with a `TZ=UTC` environment override at process launch — there is no HLS option to force UTC expansion.

### Specifier Availability Caveat

Available `strftime` specifiers are determined by the host libc. POSIX guarantees the conventional set (`%Y`, `%m`, `%d`, `%H`, `%M`, `%S`, `%j`, `%U`, `%W`, etc.), but extensions like `%N` (nanoseconds, glibc-only) are non-portable. The HLS muxer does not interpose its own placeholders into the `strftime` namespace — the only HLS-specific placeholders (`%d`, `%v`, `%t`, `%s` as documented in the previous section) are processed *after* `strftime` runs, so a `strftime` pattern that produces an output containing one of those literal percent sequences would re-enter the HLS templating layer.

### Default Pattern Fallback

When the user does not supply `hls_segment_filename` at all and `use_localtime=1`, the default fallback pattern emitted by `get_default_pattern_localtime_fmt(s)` at `[libavformat/hlsenc.c:L1860]` (called from `hls_init` at `[libavformat/hlsenc.c:L2880]`) produces a pattern based on the output playlist URL's directory component plus a wall-clock-timestamped basename. The fallback guarantees uniqueness across segments and across process restarts, which is important for live re-publish scenarios where the same playlist URL is reused.

The `strftime_mkdir` flag is only useful when the expanded pattern contains directory separators — `ff_mkdir_p` at `[libavformat/hlsenc.c:L1729]` is a no-op when the result of `av_dirname` is `.`.

---

## Interface — MPEG-TS Sub-Muxer (`segment_type=mpegts`)

### Plain-Language Summary

When `hls_segment_type=mpegts` (the default), the HLS muxer creates a child `AVFormatContext` whose output format is `ff_mpegts_muxer`. HLS forwards each `AVPacket` to the child muxer; the child writes MPEG-TS payload bytes (PAT, PMT, PES) into the AVIOContext that HLS provides for the active segment file. HLS owns segmentation; the TS muxer owns container payload.

### Technical Detail

The variant's output format is assigned during `hls_init` at `[libavformat/hlsenc.c:L2995]`:

```c
vs->oformat = &ff_mpegts_muxer.p;
```

The child format context is allocated by `hls_mux_init` at `[libavformat/hlsenc.c:L773]` via `avformat_alloc_output_context2(&vs->avf, vs->oformat, NULL, NULL)` at `[libavformat/hlsenc.c:L783]`. The resulting `vs->avf` is a fully-functional `AVFormatContext` with `oformat = &ff_mpegts_muxer.p`; its `avf->pb` is set to a dynamic in-memory buffer for the duration of the first segment by `avio_open_dyn_buf(&oc->pb)` at `[libavformat/hlsenc.c:L857]`, then replaced with the real per-segment AVIOContext at segment-cut time inside `hls_write_packet`.

The child's container header (PAT plus PMT plus any service info) is written by `avformat_write_header(vs->avf, NULL)` at `[libavformat/hlsenc.c:L2311]` inside `hls_write_header`. Note that this call goes into the *child* format context — the HLS muxer's own header writing is a no-op from the framework's standpoint because `hls_write_header` does its real work by initialising children.

Per-packet forwarding uses the chained-write helper `ff_write_chained(oc, stream_index, pkt, s, 0)` at `[libavformat/hlsenc.c:L2679]` inside `hls_write_packet`. This call forwards the AVPacket into the child muxer, mapping HLS-side stream indices to child-side stream indices. The chained helper invokes `av_write_frame` on the child internally; the HLS muxer never calls `av_write_frame` on its own AVPacket. This is the meta-muxer pattern: HLS owns segmentation logic, the child owns container payload encoding.

To force PAT/PMT re-emission at each segment boundary (so each TS segment is independently demuxable without prior context), HLS sets `mpegts_flags=resend_headers` on the child's private data at two sites: `av_opt_set(oc->priv_data, "mpegts_flags", "resend_headers", 0)` at `[libavformat/hlsenc.c:L1804]` inside `hls_start` (initial segment) and again at `[libavformat/hlsenc.c:L2651]` after each segment publish inside `hls_write_packet`. The MPEG-TS muxer's `resend_headers` flag triggers a fresh PAT and PMT on the next packet write, so every segment begins with a self-contained signal pair.

For per-segment options propagated to the child (e.g., user-supplied `mpegts_flags`, PES packet size, service IDs), HLS reads from its own `hls_segment_options` dictionary option (`AV_OPT_TYPE_DICT`) and applies it before the child's `avformat_write_header`. This is the standard libavformat option-pass-through pattern.

### Sub-Muxer Registration

`ff_mpegts_muxer` is the public registration symbol for the MPEG-TS muxer; its declaration is in `[libavformat/mpegtsenc.c:L2408]` (`FFOutputFormat ff_mpegts_muxer = { ... }`). The HLS muxer references it via `vs->oformat = &ff_mpegts_muxer.p` at `[libavformat/hlsenc.c:L2995]`. The `.p` member is the public `AVOutputFormat` sub-struct that the framework reads when dispatching `init`/`write_header`/`write_packet`/`write_trailer` callbacks; the surrounding `FFOutputFormat` struct holds implementation-private context.

### Container Payload Layout per Segment

Each TS segment begins (after `resend_headers` triggers re-emission) with a PAT (Program Association Table) packet and one PMT (Program Map Table) packet, followed by PES (Packetised Elementary Stream) packets carrying video, audio, and any data streams. The 188-byte TS packet size is fixed by the MPEG-2 standard; HLS does not override this. The first video PES packet of each segment must start with an SPS/PPS in-band (for H.264) — this is satisfied by `AVFMT_GLOBALHEADER` at `[libavformat/hlsenc.c:L3199]` which forces the encoder to embed parameter sets in the elementary stream rather than only in extradata.

### Byterange Mode Compatibility

MPEG-TS segments can be concatenated into a single file and referenced by byterange in the playlist when `hls_flags=single_file` is active. The TS muxer's stateless segment shape (no global header that the segment depends on) makes byterange-mode trivially correct: a player seeking by `EXT-X-BYTERANGE` reads N bytes starting at offset M and obtains a self-contained TS substream.

### Timestamp Forwarding

The HLS muxer does not rewrite timestamps; PTS and DTS on the forwarded `AVPacket` are written into the child TS muxer's PES headers as-is. The child applies the standard 33-bit PTS/DTS encoding at the 90 kHz MPEG clock. This passthrough is one of the functional invariants documented in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).

---

## Interface — fMP4 Sub-Muxer (`segment_type=fmp4`)

### Plain-Language Summary

When `hls_segment_type=fmp4`, the child sub-muxer is `ff_mp4_muxer` configured for fragmented output. The HLS muxer produces a single initialisation segment (`init.mp4` by default, referenced from the playlist by `EXT-X-MAP`) containing the `ftyp` and `moov` boxes, followed by one media segment per fragment — each prefixed with an `styp` segment-type box. fMP4 mode forces playlist `EXT-X-VERSION=7`.

### Technical Detail

The variant's output format is assigned during `hls_init` at `[libavformat/hlsenc.c:L2989]`:

```c
vs->oformat = &ff_mp4_muxer.p;
```

`hls_mux_init` configures the `movflags` option on the child so the MP4 muxer produces fragmented MP4 output (rather than a single seekable moov-at-end MP4). The configured movflags include `empty_moov`, `default_base_moof`, and `frag_custom` (the canonical fMP4-for-HLS combination); the exact set is computed by `hls_mux_init` based on whether single-file mode is active.

**Segment-type box (`styp`).** Each fragment begins with an `styp` box written by `write_styp(vs->out)` at two sites: `[libavformat/hlsenc.c:L2580]` inside `hls_write_packet` (per-segment cut) and `[libavformat/hlsenc.c:L2792]` inside `hls_write_trailer` (final segment). `write_styp` itself is at `[libavformat/hlsenc.c:L470]`; it writes a 24-byte `styp` box with major brand `msdh` and a compatible-brand list including `msdh` and `msix`.

**Initialisation segment capture.** When the first segment cut occurs and `vs->init_range_length` is still zero, the dynamically-buffered child writes (the `ftyp` and `moov` boxes the mp4 muxer wrote into the dynamic AVIOContext during `avformat_write_header`) are captured by `range_length = avio_close_dyn_buf(oc->pb, &vs->init_buffer)` at `[libavformat/hlsenc.c:L2513]`. The captured buffer is then written to the actual init segment file via `avio_write(vs->out, vs->init_buffer, range_length)` at `[libavformat/hlsenc.c:L2517]`. Whether `vs->init_buffer` is freed immediately or retained for future resends is gated by `hls->resend_init_file` at `[libavformat/hlsenc.c:L2518-L2519]`: when `resend_init_file=0`, `av_freep(&vs->init_buffer)` runs immediately; when `resend_init_file=1`, the buffer stays alive for the lifetime of the format context. After the init write, a fresh dynamic buffer is opened on the child via `avio_open_dyn_buf(&oc->pb)` at `[libavformat/hlsenc.c:L2520]` so subsequent fragment writes can be buffered the same way.

**Version pin.** fMP4 mode forces `EXT-X-VERSION=7` in the playlist; see [`codec-logic.md`](codec-logic.md) for the version-negotiation decision table and the source citation at `[libavformat/hlsenc.c:L1569-L1571]`.

**Encryption.** As noted in the AES-128 Crypto Pipeline section, encrypted fMP4 is unsupported and returns `AVERROR_PATCHWELCOME` from `hls_start` at `[libavformat/hlsenc.c:L1770-L1771]`. The `crypto:` pseudo-protocol cannot wrap an fMP4 fragment stream because the fMP4 muxer's seek-and-rewrite behaviour is incompatible with stream-mode AES-CBC.

### `EXT-X-MAP` Playlist Line

Because fMP4 splits stream-level metadata (the `moov` box) into a separate initialisation segment, the playlist must reference that init segment via `EXT-X-MAP:URI="init.mp4"` before any media-segment entry. The line is written by `ff_hls_write_init_file` (in `[libavformat/hlsplaylist.c]`, invoked from `[libavformat/hlsenc.c:L1612]`). Players that do not understand `EXT-X-MAP` cannot play fMP4 segments — this is one of the reasons fMP4 mode forces `EXT-X-VERSION=7`.

### Byterange Mode and Single-File fMP4

When `hls_flags=single_file` is combined with fMP4, the init segment is concatenated to the head of the single output file, and every fragment is a byterange suffix referenced by `EXT-X-BYTERANGE`. The init segment's byterange (`0`-`init_range_length`) is implicit in `EXT-X-MAP`; fragments are explicit. The `vs->out_single_file` AVIOContext (declared at `[libavformat/hlsenc.c:L127]`) is the persistent handle in this mode.

### Init Segment Filename Resolution

`hls_fmp4_init_filename` defaults to `"init.mp4"` at `[libavformat/hlsenc.c:L3142]`. The user can override it with a `strftime` pattern or `%v` placeholder; resolution happens in `hls_init` at `[libavformat/hlsenc.c:L3044]`. Each variant ends up with its own init segment filename in `vs->fmp4_init_filename`. The full URL (basename plus filename) is stored in `vs->base_output_dirname` for use by `hls_init_file_resend`.

### Movflags Composition

The exact movflags string is composed by `hls_mux_init` and depends on whether single-file mode is active. The canonical set for streaming-mode fMP4 is `empty_moov+default_base_moof+frag_custom`: `empty_moov` suppresses sample data in the initial `moov`, `default_base_moof` shrinks per-fragment headers by hoisting common values into `tfhd`/`trex`, and `frag_custom` lets the application (the HLS muxer) decide fragment boundaries rather than the mp4 muxer's automatic flushing. For single-file mode the movflag set adjusts to keep all fragments addressable in one container.

---

## Interface — HTTP DELETE for `hls_flags=delete_segments`

### Plain-Language Summary

When `hls_flags=delete_segments` is set and the output URL is HTTP-based, the HLS muxer issues HTTP DELETE requests against the URLs of expired segments. A dedicated AVIOContext slot — `HLSContext::http_delete` — holds the DELETE channel separate from the segment-write and playlist-write channels.

### Technical Detail

The dedicated AVIOContext field is declared at `[libavformat/hlsenc.c:L261]`:

```c
AVIOContext *http_delete;
```

The cleanup routine is `hls_delete_file(HLSContext *hls, AVFormatContext *avf, char *path, const char *proto)` at `[libavformat/hlsenc.c:L507-L529]`. It branches on whether the protocol is `http` (the proto argument, derived from `avio_find_protocol_name(...)` in the caller `hls_delete_old_segments`) or whether `hls->method` is set:

- **HTTP path** — at `[libavformat/hlsenc.c:L510-L522]`. The function builds an option dictionary, calls `set_http_options(avf, &opt, hls)` to populate the user-agent, timeout, and headers, then overrides the method with `av_dict_set(&opt, "method", "DELETE", 0)` at `[libavformat/hlsenc.c:L515]`. The dedicated `hls->http_delete` AVIOContext is opened via `hlsenc_io_open(avf, &hls->http_delete, path, &opt)` at `[libavformat/hlsenc.c:L517]` with `AVIO_FLAG_WRITE`. No body is written; the DELETE request is dispatched by the HTTP protocol driver on close. `hlsenc_io_close(avf, &hls->http_delete, path)` at `[libavformat/hlsenc.c:L523]` finalises the request.

- **Local file path** — at `[libavformat/hlsenc.c:L524-L527]`. When the URL is not HTTP-based, the function falls back to `unlink(path)` at `[libavformat/hlsenc.c:L524]`. Failures are logged but not propagated as errors.

Final cleanup of the DELETE channel happens in `hls_deinit` via `ff_format_io_close(s, &hls->http_delete)` at `[libavformat/hlsenc.c:L2720]`. The DELETE channel is reused across many requests when `http_persistent` is set; otherwise each DELETE opens and closes its own TCP connection.

The DELETE-emit decision is taken by `hls_delete_old_segments(s, hls, vs)` at `[libavformat/hlsenc.c:L531-L640]` (the sliding-window cleanup driver), invoked from `hls_window` at `[libavformat/hlsenc.c:L1131]` once per playlist publish. The driver walks `vs->segments` to compute `playlist_duration` then walks `vs->old_segments` (the FIFO of segments that have already aged out of the live window) at `[libavformat/hlsenc.c:L555-L570]` accumulating segments to delete until `segment_cnt >= hls->hls_delete_threshold` is hit at `[libavformat/hlsenc.c:L566]` — this is how `hls_delete_threshold` (default `1`, declared at `[libavformat/hlsenc.c:L3126]`) controls retention.

The protocol detection for each deletion call is shared across the whole batch: `proto = avio_find_protocol_name(s->url)` at `[libavformat/hlsenc.c:L607]` is computed once from the playlist URL (not the segment URL), and the same `proto` is passed to every `hls_delete_file(hls, s, path.str, proto)` invocation at `[libavformat/hlsenc.c:L608]`. The implicit assumption is that segments share the playlist's protocol — which is true for the canonical HLS deployment (playlist and segments on the same origin), but breaks if `hls_base_url` redirects segments to a different host with a different scheme.

### `ignore_io_errors` Interaction

When `hlsenc_io_open` for the DELETE fails (e.g., 404 because the segment was already cleaned by a separate process), the branch at `[libavformat/hlsenc.c:L519-L520]` checks `hls->ignore_io_errors`: if set, `hls_delete_file` returns `1` (warning, continue with the next segment); if unset, it returns the underlying error and aborts the batch. The DELETE channel itself does not propagate response bodies — HLS issues the request and treats any successful TCP-level close as success, regardless of HTTP response code (204, 404, 410 all look identical from the AVIOContext perspective unless the protocol driver translates HTTP status into an `AVERROR` code).

### Subtitle Companion Deletes

When a segment has a companion WebVTT subtitle file (`segment->sub_filename` is non-empty), the cleanup also issues a second DELETE against the subtitle URL at `[libavformat/hlsenc.c:L611-L625]`. The same `proto` and `set_http_options` configuration apply.

### Variant-Specific Path Resolution

The directory portion of the segment URL is computed once before the deletion loop at `[libavformat/hlsenc.c:L572-L593]`, with `%v` variant placeholder substitution applied via `replace_int_data_in_filename` or `replace_str_data_in_filename` (using `vs->varname` when set). This means a `hls_segment_filename` template with `%v` produces correct DELETE URLs even when the directory portion is variant-templated.

---

## Interface — fMP4 Initialization Segment Resend

### Plain-Language Summary

In fMP4 mode with `hls_fmp4_init_resend=1`, the initialisation segment (`init.mp4` by default) is re-uploaded after every playlist publish, ensuring CDN cache freshness when the upstream CDN purges or revalidates by m3u8 mtime. The init buffer is retained in memory for the lifetime of the format context to support this resend.

### Technical Detail

The implementation is `hls_init_file_resend(AVFormatContext *s, VariantStream *vs)` at `[libavformat/hlsenc.c:L2362-L2377]`:

```c
ret = hlsenc_io_open(s, &vs->out, vs->base_output_dirname, &options);
...
avio_write(vs->out, vs->init_buffer, vs->init_range_length);
hlsenc_io_close(s, &vs->out, hls->fmp4_init_filename);
```

It calls `set_http_options(s, &options, hls)` at `[libavformat/hlsenc.c:L2368]`, opens the init segment URL stored in `vs->base_output_dirname` via `hlsenc_io_open(s, &vs->out, vs->base_output_dirname, &options)` at `[libavformat/hlsenc.c:L2369]`, writes the retained init buffer via `avio_write(vs->out, vs->init_buffer, vs->init_range_length)` at `[libavformat/hlsenc.c:L2373]`, and finalises with `hlsenc_io_close(s, &vs->out, hls->fmp4_init_filename)` at `[libavformat/hlsenc.c:L2374]`.

The activation gate is in `hls_write_packet` at `[libavformat/hlsenc.c:L2638-L2644]`:

```c
if (hls->resend_init_file && hls->segment_type == SEGMENT_TYPE_FMP4) {
    ret = hls_init_file_resend(s, vs);
    ...
}
```

The resend fires after every per-variant playlist update (the `hls_window` call immediately above) and before the next segment is written. Both conditions must hold: `resend_init_file=1` AND `segment_type=fmp4`. In MPEG-TS mode there is no init segment, so the flag is silently ignored.

The init buffer's retention is gated by the same `resend_init_file` flag. At first segment cut, the `if (!hls->resend_init_file) av_freep(&vs->init_buffer)` branch at `[libavformat/hlsenc.c:L2518-L2519]` frees the buffer immediately when resend is off; when resend is on, the buffer stays alive in `vs->init_buffer` until `hls_deinit` frees it via `if (hls->resend_init_file) av_freep(&vs->init_buffer)` at `[libavformat/hlsenc.c:L2710-L2711]`.

The init segment URL itself is determined by `hls_fmp4_init_filename` (default `"init.mp4"`, declared at `[libavformat/hlsenc.c:L3142]`); when an explicit path is supplied, it can include `strftime` placeholders, expanded once at `hls_init` time via `strftime_expand` at `[libavformat/hlsenc.c:L3044]`. The URL written by the resend is whatever was computed and stored in `vs->base_output_dirname`; the resend overwrites the same URL on every call.

### Timing Relative to Playlist Publish

The activation gate at `[libavformat/hlsenc.c:L2638-L2644]` is positioned after `hls_window` returns successfully. This means the sequence per segment-cut is: (1) finalise the current media segment, (2) update the playlist via `hls_window`, (3) resend the init segment if configured. The init resend therefore happens *after* the playlist has been published referencing it via `EXT-X-MAP`, so a player that fetches the new playlist immediately on publish may briefly fetch an older init segment from the CDN cache before the resend lands. For most CDN configurations the freshness gap is well below typical player buffering thresholds, but strict synchronisation across CDN edges is not guaranteed by this design.

### Cache Freshness Rationale

The resend exists because CDN caches typically key on URL + mtime, and the init segment URL is static across an entire publish session (it does not include a sequence number or wall-clock timestamp by default). Without `resend_init_file=1`, an init segment that is uploaded once at format-context start and never re-uploaded looks stale to CDNs that bind cache entries to the response's `Last-Modified` header. Re-uploading on every playlist publish refreshes `Last-Modified` (and any other origin-side ETag) so the CDN keeps the init in active cache.

For deployments where the init segment is served from a CDN that supports cache-bust query strings or where the upstream `hls_base_url` includes a unique session token, `resend_init_file=0` is the more efficient configuration — it avoids the extra HTTP request per segment-cut.

### `init_range_length` Tracking

The captured init buffer length is stored in `vs->init_range_length` (an `int64_t` field on `VariantStream` — see [`data-model.md`](data-model.md) for the full struct dictionary). It is set at `[libavformat/hlsenc.c:L2513]` from the `avio_close_dyn_buf` return value and read by `hls_init_file_resend` at `[libavformat/hlsenc.c:L2373]` (`avio_write(vs->out, vs->init_buffer, vs->init_range_length)`). The value never changes after first segment cut — the init segment is fixed for the lifetime of the format context.

In single-file mode where the init bytes are prefixed to the first media segment within the same file, `vs->init_range_length` also serves as the `EXT-X-MAP:BYTERANGE` upper bound emitted in the playlist; this dual use is the reason for the `int64_t` width rather than `size_t`.

### Distinction: `base_output_dirname` vs `fmp4_init_filename`

`vs->base_output_dirname` holds the fully-qualified URL the init segment is written to (directory plus filename, with all templates expanded). `hls->fmp4_init_filename` holds only the filename component (post-template-expansion) that appears in the playlist's `EXT-X-MAP:URI` line. The two are kept distinct because the playlist URI is typically relative (just the filename) while the write target is absolute (full URL).

---

## Cross-Cutting Concerns

This closing section catalogues the shared idioms that recur across multiple interfaces above and the architectural patterns that govern how HLS engages with external systems. It is intentionally light on new citations — every cited line range below is also cited in one of the per-interface sections.

### Pattern — AVDictionary as Option Bag

Every HTTP-aware interface uses `AVDictionary` as the option-passing bag between HLS and the libavformat protocol layer: `hlsenc_io_open` accepts an `AVDictionary **options` parameter, `set_http_options` populates that dictionary, and the dictionary is freed via `av_dict_free(&opt)` after the open call. Examples: segment-write open chain inside `hls_start`, DELETE channel in `hls_delete_file` at `[libavformat/hlsenc.c:L511-L518]`, init segment resend in `hls_init_file_resend` at `[libavformat/hlsenc.c:L2364-L2371]`. The dictionary is consumed by the HTTP protocol driver inside `s->io_open`; once consumed, the entries that were applied are removed from the dictionary, and entries that the protocol does not recognise remain — this allows HLS to detect mis-spelled options via post-open inspection (though the HLS muxer does not currently do so).

### Pattern — AVIOContext Pass-by-Pointer-to-Pointer

Every HLS interface that opens an AVIOContext does so via a double-pointer (`AVIOContext **pb`), allowing the open helper to allocate the context and store it back into the caller's struct field. This is the libavformat-wide convention. The closure paths take the same double-pointer and set the field to NULL after closing, so a subsequent `if (vs->out)` check correctly identifies an open vs closed slot.

### Pattern — `ignore_io_errors` Soft-Failure Mode

When `hls->ignore_io_errors` is set (declared at `[libavformat/hlsenc.c:L263]` in `HLSContext`, exposed as the `ignore_io_errors` AVOption at `[libavformat/hlsenc.c:L3178]`), I/O failures that would normally abort the encode are downgraded to warnings: the DELETE channel returns `1` instead of the error at `[libavformat/hlsenc.c:L520]`, segment writes that fail with `AVERROR(EIO)` are logged and skipped (see `[libavformat/hlsenc.c:L2573-L2577]`), and the format-context-wide error counter records the failure for visibility without breaking the encode pipeline. This is the standard live-streaming resilience mode: a single dropped segment or failed DELETE is preferable to losing the entire stream.

For deployments that require strict correctness (e.g., archival VOD encodes where every segment must land), `ignore_io_errors=0` is the right choice. For live encodes facing flaky CDN ingest, `ignore_io_errors=1` keeps the encoder running across transient network failures.

### Pattern — Stream-of-Interfaces Per Segment

For each segment, HLS engages multiple interfaces in a deterministic sequence:

1. **Filename templating** (sections 8, 9) — `replace_int_data_in_filename`, `replace_str_data_in_filename`, `strftime_expand` compute the segment URL.
2. **AES key install** (section 6) — `hls_encryption_start` or `do_encrypt` runs at first segment or per-rekey; the key is written via `s->io_open`.
3. **`crypto:` URL composition** (section 4) — `av_asprintf("crypto:%s", oc->url)` wraps the segment URL when encryption is active.
4. **File or HTTP open** (sections 2, 3) — `hlsenc_io_open` opens the (possibly `crypto:`-wrapped) segment URL.
5. **Sub-muxer write** (sections 10, 11) — `ff_write_chained` forwards each `AVPacket` to the TS or fMP4 child muxer.
6. **File or HTTP close** (sections 2, 3) — `hlsenc_io_close` finalises the segment write.
7. **Playlist update** (cross-cutting, see [`pipeline-orchestration.md`](pipeline-orchestration.md)) — `hls_window` re-publishes the m3u8.
8. **DELETE** (section 12) — `hls_delete_old_segments` removes any segments past the retention window.
9. **fMP4 init resend** (section 13) — `hls_init_file_resend` re-uploads the init segment if configured.

This sequence is enforced by the order of calls inside `hls_write_packet`; see [`process-flows.md`](process-flows.md) for the visualised flow.

### Pattern — Demuxer-Side Inversions

Several interfaces have demuxer-side counterparts that invert the muxer's data flow:

- AES-128 segment encryption (muxer-side) ↔ AES-128 segment decryption (demuxer-side, handled by the `crypto:` URL protocol when the demuxer opens a segment URL starting with `crypto:` — the demuxer reads the `EXT-X-KEY` line, fetches the key URI, and prefixes each segment URL with `crypto:` for the same protocol driver to handle decrypt mode with `decrypt=1` on the `av_aes_init` call).
- HLS Sample Encryption (demuxer-side, section 7) ↔ has no muxer-side counterpart — the FFmpeg HLS muxer does not emit Sample-Encrypted streams; it only consumes them via the demuxer.
- HTTP PUT for segment writes (muxer-side) ↔ HTTP GET for segment reads (demuxer-side, handled by the HTTP protocol driver's standard GET behaviour when the demuxer opens a segment URL).

### What HLS Does *Not* Interface With Directly

For traceability when porting, the following subsystems are NOT integration interfaces of the HLS muxer or demuxer:

- **Codec encoders and decoders** — HLS forwards `AVPacket` objects whose payload is already compressed; it never invokes `avcodec_send_frame` or `avcodec_receive_packet`. The encoder lives upstream in the application using HLS as output.
- **Filter graphs** — HLS does not invoke `libavfilter`; frame transformations happen upstream.
- **Network sockets directly** — HLS never opens a socket via `socket(2)`, `bind(2)`, `connect(2)`, etc. The HTTP and HTTPS protocol drivers in libavformat own all socket lifecycle.
- **TLS context creation** — TLS is handled inside the `https://` protocol driver; HLS supplies only the URL.
- **Threading primitives** — HLS is single-threaded per `AVFormatContext`. No pthread, no atomic, no condvar interfaces are used directly.

### Summary Interface Reference

The table below provides a one-line summary per interface for quick lookup. Each row corresponds to one H2 section above.

| # | Interface | Primary Code Site | Activation Condition |
|---|-----------|-------------------|----------------------|
| 1 | File AVIOContext writes | `hlsenc_io_open` `[L292-L311]`, `hlsenc_io_close` `[L313-L331]` | Default for `file://` URLs (or when `http_persistent` is not in use) |
| 2 | HTTP AVIOContext writes (persistent reuse) | `hlsenc_io_open` `[L298-L308]`, `hlsenc_io_close` `[L320-L327]` | `http_persistent=1` AND HTTP URL |
| 3 | Protocol handlers (`file`/`http`/`https`/`crypto`) | URL scheme dispatch in `s->io_open` | URL scheme prefix |
| 4 | `ff_is_http_proto` detection | `[L296], [L316], [L335], [L2875]` | Always invoked; result gates branches |
| 5 | AES-128 crypto pipeline | `do_encrypt` `[L641-L711]`, `hls_encryption_start` `[L714-L771]` | `hls_enc=1` OR `hls_key_info_file` set |
| 6 | HLS Sample Encryption (demuxer) | `ff_hls_senc_decrypt_frame` `[hls_sample_encryption.h:L63]` | Demuxer sees `STREAM_TYPE_HLS_SE_*` in PMT |
| 7 | `hls_segment_filename` templating | `replace_int_data_in_filename` `[L422-L468]`, `replace_str_data_in_filename` `[L382-L420]`, `sls_flags_filename_process` `[L908-L946]` | Placeholder present in template AND corresponding flag set |
| 8 | `use_localtime` `strftime` expansion | `strftime_expand` `[L269-L290]` | `use_localtime=1` |
| 9 | MPEG-TS sub-muxer | `ff_write_chained` `[L2679]`, `vs->oformat = &ff_mpegts_muxer.p` `[L2995]` | `hls_segment_type=mpegts` (default) |
| 10 | fMP4 sub-muxer | `vs->oformat = &ff_mp4_muxer.p` `[L2989]`, `write_styp` `[L470]`, init capture `[L2513]` | `hls_segment_type=fmp4` |
| 11 | HTTP DELETE for expired segments | `hls_delete_file` `[L507-L529]`, `hls->http_delete` `[L261]` | `hls_flags=delete_segments` AND HTTP URL (or local fallback to `unlink`) |
| 12 | fMP4 init segment resend | `hls_init_file_resend` `[L2362-L2377]` | `hls_fmp4_init_resend=1` AND `segment_type=fmp4` |

All `[L...]` references in the table without an explicit file prefix refer to `libavformat/hlsenc.c`.

### Risk Identification for Porting

For engineers planning a port or rewrite of any subset of these interfaces, the following are the highest-complexity touchpoints — the ones with the densest state, the most non-obvious sequencing, and the most cross-interface dependencies:

- **HTTP persistent connection lifecycle (sections 2, 3)**: The `http_persistent` branch in `hlsenc_io_open`/`hlsenc_io_close` interleaves connection reuse with new-request issuance via `ff_http_do_new_request`. Bugs here manifest as stale-connection responses, partial PUT bodies, or kept-alive TCP slots leaking through the format-context lifetime. A port must reproduce both the open-time persistent fast-path at `[libavformat/hlsenc.c:L298-L308]` and the close-time `ffurl_shutdown` short-circuit at `[libavformat/hlsenc.c:L320-L327]`.
- **`crypto:` URL composition timing (sections 4, 6)**: The `crypto:` prefix must wrap the inner URL *and* the dictionary entries `encryption_key` and `encryption_iv` must be populated *before* the `s->io_open` call. The per-segment composition at `[libavformat/hlsenc.c:L2553-L2556]` and the trailer-time composition at `[libavformat/hlsenc.c:L2752-L2755]` both follow this ordering. A port that re-orders the dict population after the open call will silently disable encryption.
- **fMP4 init segment buffer lifetime (sections 11, 13)**: The init buffer at `vs->init_buffer` is captured once and either freed immediately (when `resend_init_file=0`) or retained until format context close (when `resend_init_file=1`). The decision branch at `[libavformat/hlsenc.c:L2518-L2519]` is easy to overlook; the consequences of getting it wrong are either a memory leak or a use-after-free during init resend.
- **Sliding-window DELETE batch protocol detection (section 12)**: The protocol name is computed once per batch from the *playlist* URL at `[libavformat/hlsenc.c:L607]` and reused for every DELETE. This holds for canonical deployments where playlist and segments share an origin, but breaks if `hls_base_url` redirects segments to a different scheme. A port should clarify whether to preserve this single-protocol-per-batch optimisation or compute per-segment protocols.
- **Sample Encryption priming and extradata wiring (section 7)**: The demuxer-side flow from ID3 parsing → `ff_hls_senc_read_audio_setup_info` → `ff_hls_senc_parse_audio_setup_info` → per-frame `ff_hls_senc_decrypt_frame` involves four distinct call sites with timing dependencies between them. The AVStream's extradata and codec_tag must be set before any decoder downstream of HLS sees the first packet, which means the parse must complete during `hls_read_header` rather than lazily on first `hls_read_packet`.

These are the porting hotspots. The remaining interfaces — file writes, filename templating, `strftime` expansion, MPEG-TS forwarding — are mechanically straightforward by comparison.

---
