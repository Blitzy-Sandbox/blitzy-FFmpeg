# Integration Contracts — HLS Pipeline API Contracts

> **Commit Anchor:** All source references in this document are anchored to commit `566ad786` (full hash `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). Line numbers cited as `[<path>:L<start>-L<end>]` are valid at this commit. See [`../README.md`](../README.md) for the documentation-set-wide commit-anchor convention and citation format.

---

## Overview

### Plain-Language Summary

External systems — media players, content delivery networks, key-distribution servers, and the demuxers that parse the muxer's output — depend on the FFmpeg HLS pipeline producing specific wire formats and following specific protocols. This document is the one-page-per-contract reference for every such external touchpoint. A port, rewrite, or extension that changes any wire-level emission, HTTP method, attribute order, or stream-type value documented here is a breaking change against some downstream consumer.

The document is organised as nine contract sections, each one structured identically. Section bodies enumerate **Direction** (outbound from the muxer, inbound to the demuxer, or bidirectional), **Format / Wire** (the exact bytes or text emitted or consumed, with source citations), **Default** (the option's or constant's literal default), **Optionality** (which `AVOption`, flag, or condition gates the contract), and **Failure mode if violated** (what breaks downstream). The Validation Checklist at the bottom of the document summarises all nine contracts in a single scannable list and can be used directly as a porting conformance test plan.

### Audience and Scope

The audience is mixed: engineers integrating with HLS-compliant players, CDNs, and key servers who need to confirm the wire formats their consumers will see; and engineers porting the HLS pipeline to other codebases who must preserve external-system compatibility byte-for-byte. Library integrators who only need to know *what* shape is emitted can read the plain-language statement at the top of each section and the Validation Checklist; engineers porting the pipeline should read each section's `### Technical detail` subsection top to bottom.

The contracts catalogued below are all *external-system* contracts — they describe the boundary between the HLS pipeline and code or services outside the FFmpeg HLS source tree. Three companion documents in the same `api-contracts/` directory cover the other three dimensions of the API contract surface:

- [`./functional-invariants.md`](functional-invariants.md) — the behavioural MUST/MUST NOT statements that the contracts in this document implement (header-line order, version-negotiation rules, tag placement, etc.).
- [`./data-contracts.md`](data-contracts.md) — the data-shape contracts (`AVOption` types, struct field layouts, timestamp unit conventions, binary layouts) that the wire formats in this document exchange.
- [`./timing-dependencies.md`](timing-dependencies.md) — the ordering contracts that constrain *when* each external contract is exercised (e.g., AES-128 key installed before the first encrypted segment, fMP4 init segment written before `EXT-X-MAP` references it).

Internal mechanism (the *how* of each interface, with full source narrative) is covered by [`../technical/integration-interfaces.md`](../technical/integration-interfaces.md). Recovery behaviour when a contract is violated at runtime is catalogued in [`../functionality/exception-handling.md`](../functionality/exception-handling.md).

### Two HLS Encryption Schemes — Different Contracts

This document carries two distinct cryptographic contracts because HLS defines two mutually-exclusive encryption schemes implemented in two different code paths:

- **AES-128 full-segment encryption (muxer side).** Every byte of a segment file is encrypted as one large CBC stream using a single key and IV per key-period. Implemented in `libavformat/hlsenc.c` via the `crypto:` pseudo-protocol wrap; advertised in the playlist via `#EXT-X-KEY:METHOD=AES-128`. See the **AES-128 Key URI Fetch** and **EXT-X-KEY METHOD Line Layout** contracts below.
- **Sample-AES (HLS Sample Encryption, demuxer side only).** Individual frames inside the MPEG-TS payload are encrypted in place; encrypted streams are identified by four custom stream-type values in the PMT. Implemented in `libavformat/hls_sample_encryption.c`. The FFmpeg HLS *muxer* never produces Sample-AES content; only the demuxer recognises and decrypts it. See the **HLS Sample Encryption Transport** contract below.

The build-time separation is visible in `[libavformat/Makefile:L276-L277]`: `hls_sample_encryption.o` is linked into the demuxer object set (`OBJS-$(CONFIG_HLS_DEMUXER)`) and is *not* linked into the muxer object set (`OBJS-$(CONFIG_HLS_MUXER)`). A port that merges the two encryption code paths into one library object breaks this separation and changes the muxer's emitted contract surface.

---

## Contract: AES-128 Key URI Fetch

When AES-128 full-segment encryption is enabled, the muxer advertises a key URI in every applicable `#EXT-X-KEY` playlist line, and a downstream player fetches that URI to retrieve a 16-byte raw binary AES-128 key. The muxer's own key source is either a 3-line key-info file (URI, key file path, optional IV) or an internally-generated random key written to a derived `.key` file on disk.

### Technical detail

**Direction.** Bidirectional. *Outbound* in the sense that the URI string is advertised in the playlist; *inbound* in the sense that the player follows that URI to retrieve the key bytes, and the muxer itself reads the key bytes from disk (or generates them) at session-start time.

**Format / Wire.** Two key-source paths exist in source, gated by the presence or absence of `hls->key_info_file`:

- **Key-info-file path.** When `hls_key_info_file` is set, the muxer opens the named file via `s->io_open` inside `hls_encryption_start` at `[libavformat/hlsenc.c:L723]` and reads exactly three CRLF-stripped lines: the **key URI** (advertised in `#EXT-X-KEY:URI=...`) from the first line via `ff_get_line` at `[libavformat/hlsenc.c:L731]`, the **key file path** (the muxer's local-disk read path for key bytes) from the second line at `[libavformat/hlsenc.c:L734]`, and an **optional IV** (hex-encoded) from the third line at `[libavformat/hlsenc.c:L737]`. The function then opens the named key file via `s->io_open` at `[libavformat/hlsenc.c:L753]`, reads `KEYSIZE` (16) bytes via `avio_read` at `[libavformat/hlsenc.c:L760]`, and hex-encodes the bytes into `vs->key_string` via `ff_data_to_hex` at `[libavformat/hlsenc.c:L768]`. The exact-size requirement is enforced at `[libavformat/hlsenc.c:L762-L766]`: a read of fewer than 16 bytes returns `AVERROR(EINVAL)`.
- **Auto-generated key path.** When `hls_key_info_file` is unset and `hls_enc=1`, the muxer takes the `do_encrypt` path at `[libavformat/hlsenc.c:L641-L711]`. It synthesises a default key URI from the user-supplied `hls_enc_key_url` (or, if absent, from a `<basename>.key` filename derived from the master playlist URL or `s->url`) at `[libavformat/hlsenc.c:L658-L664]`. It then generates 16 random bytes via `av_random_bytes` at `[libavformat/hlsenc.c:L692]` (or copies the user-supplied `hls_enc_key` raw bytes if non-NULL at `[libavformat/hlsenc.c:L697]`), writes those bytes to the on-disk key file with `avio_write(pb, key, KEYSIZE)` at `[libavformat/hlsenc.c:L707]`, and hex-encodes them into `hls->key_string` via `ff_data_to_hex` at `[libavformat/hlsenc.c:L700]`.

In both paths, the on-the-wire contract for the key file referenced by the URI is identical: it MUST contain **exactly 16 raw bytes** with no header, no text encoding, no base64, and no newline framing. The `KEYSIZE 16` constant is defined at `[libavformat/hlsenc.c:L70]` and is consumed by `av_aes_init` at `[libavutil/aes.h:L51]` as `key_bits=128` (16 bytes × 8 bits/byte = 128 bits).

**Default.** No encryption is engaged unless `hls_enc=1` (default 0 at `[libavformat/hlsenc.c:L3134]`) or `hls_key_info_file` is set to a non-NULL string (default NULL at `[libavformat/hlsenc.c:L3133]`). The `hls_enc_key_url` AVOption (default NULL at `[libavformat/hlsenc.c:L3136]`) populates the URI emitted in the playlist; when unset, the muxer derives a URI from the segment basename.

**Optionality.** Either the `hls_enc` boolean or the `hls_key_info_file` string activates the encryption path. The two are not mutually exclusive at the option-parser level, but only the `hls_key_info_file` path is invoked when both are set, because the `hls_key_info_file` check fires first in `hls_write_header`. The `HLS_PERIODIC_REKEY` flag (defined at `[libavformat/hlsenc.c:L110]`) augments the `hls_key_info_file` path with periodic file-reload semantics that allow a new key to be staged without restarting the muxer.

**Failure mode if violated.** Three failure modes are concrete:

1. A key file that is not exactly 16 bytes will fail the `ret != sizeof(key)` check at `[libavformat/hlsenc.c:L762]` and `hls_encryption_start` returns `AVERROR(EINVAL)`; the host call to `avformat_write_header` propagates the error and the muxing session never starts. A subtler failure occurs when the key file *is* exactly 16 bytes but contains text-encoded content (e.g., a 32-character hex string truncated to 16 chars) — the read succeeds, `av_aes_init` accepts the 16 bytes as raw key material, and produced segments decrypt successfully on the muxer side but cannot be decrypted by any compliant client (because the client expects the literal 16-byte binary key, not the truncated text).
2. A key URI advertised in the playlist that points to a resource the player cannot fetch (wrong host, missing CORS header, 403 from access-controlled key server) is undetectable at the muxer; segments stream correctly but clients fail decryption with player-specific error codes (e.g., `MEDIA_ERR_DECRYPT` in HTML5 Media Source Extensions).
3. A key info file with an empty first line (URI) or empty second line (key path) is rejected at `[libavformat/hlsenc.c:L744]` and `[libavformat/hlsenc.c:L749]` with `AVERROR(EINVAL)` — this is the explicit malformed-input contract for the 3-line file format.

---

## Contract: fMP4 Initialization Segment Delivery

When `hls_segment_type=fmp4` is selected, every fMP4 media segment is preceded by an *initialization segment* (the `init.mp4` file by default) containing the ISO BMFF `ftyp` and `moov` boxes that the player needs to parse subsequent media-segment fragments. The muxer writes this init segment once at the start of the session, advertises its filename in the playlist via the `#EXT-X-MAP` line, and optionally re-emits it on every playlist refresh when `hls_fmp4_init_resend=1`.

### Technical detail

**Direction.** Outbound. The init segment is one of two file artifacts (the other being the media segment file) that every fMP4 playlist consumer must fetch.

**Format / Wire.** Two distinct on-the-wire surfaces apply:

- **File artifact.** The init segment is an ISO BMFF file (ftyp + moov, no movie fragments) produced by the fMP4 child sub-muxer (`libavformat/movenc.c`'s `mov`/`mp4`/`fmp4` outputs). HLS does not author the box layout itself; it captures the bytes emitted by the sub-muxer into the `vs->init_buffer` dynamic buffer via `avio_close_dyn_buf` at `[libavformat/hlsenc.c:L2513]` during the first segment cut, and writes those bytes to the named init file via `avio_write(vs->out, vs->init_buffer, range_length)` at `[libavformat/hlsenc.c:L2516]`. The captured buffer is retained for resend (or freed if `hls_fmp4_init_resend=0`) at `[libavformat/hlsenc.c:L2517-L2518]`, and its byte count is stored in `vs->init_range_length` at `[libavformat/hlsenc.c:L2519]`.
- **Playlist line.** The init file is referenced in the media playlist via the `#EXT-X-MAP` line emitted by `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]`. The format string is `#EXT-X-MAP:URI="<filename>"` at `[libavformat/hlsplaylist.c:L137]`, with an optional `,BYTERANGE="<size>@<pos>"` suffix at `[libavformat/hlsplaylist.c:L139]` when single-file byterange mode is engaged. The line is emitted once per playlist publish from `hls_window` at `[libavformat/hlsenc.c:L1611-L1614]` (only when `segment_type == SEGMENT_TYPE_FMP4` AND the segment is the first segment of the playlist).

The init segment's URI in the playlist is a relative path string (typically `init.mp4` or a `%v`-expanded variant filename like `init_0.mp4` for a multi-variant publish); it is the same string that the player concatenates with the playlist's base URL to form the absolute fetch URL.

**Default.** The `hls_fmp4_init_filename` `AVOption` defaults to the literal string `"init.mp4"` at `[libavformat/hlsenc.c:L3142]`. The `hls_fmp4_init_resend` boolean defaults to `0` (no resend) at `[libavformat/hlsenc.c:L3143]`.

**Optionality.** The entire fMP4-init contract is gated by `hls_segment_type=fmp4` (the alternative `mpegts` value of the option at `[libavformat/hlsenc.c:L3139]` selects MPEG-TS segments, which do not have an initialization segment). The optional resend path is gated by `hls_fmp4_init_resend=1`; when enabled, `hls_init_file_resend` at `[libavformat/hlsenc.c:L2362-L2377]` is invoked from `hls_write_packet` at `[libavformat/hlsenc.c:L2639]` on every segment-cut boundary, re-writing the captured `vs->init_buffer` to disk via `hlsenc_io_open`/`avio_write`/`hlsenc_io_close`.

**Failure mode if violated.** Three failure modes are concrete:

1. A missing init file (the playlist references `init.mp4` but the file is not on disk or not reachable via HTTP) causes the player to fail with "unknown box" or "no moov" parser errors on the first media segment fetch. The player has no segment-format knowledge until it reads the init segment's `moov` atom.
2. An init file whose contents do not match the codec parameters of the media segments (e.g., the init was captured for AVC profile High 4.0 but the media is profile High 4.2) causes silent decode-time failures: the player accepts the init, parses sample-descriptions from it, and then mis-decodes media samples whose actual encoding parameters differ.
3. Changing the default filename from `"init.mp4"` to another string in a port is benign in isolation, but downstream tooling (CDN log aggregators, content-validation scripts) that hard-codes the `init.mp4` filename will misclassify the init segment. Renaming the default also breaks the `EXT-X-MAP:URI="init.mp4"` literal that any documentation or test fixture has baked in.

---

## Contract: HTTP Chunked Transfer (PUT default + http_persistent)

When the muxer's output URL (`s->url`) is an HTTP or HTTPS URL, every segment file, every playlist update, and every `EXT-X-KEY` key file is uploaded via HTTP `PUT` by default. With the `http_persistent` option enabled, the same TCP connection is reused across all uploads via `ff_http_do_new_request`, converting the per-write open-connect-write-close cycle into a single long-lived connection over which each upload is a new HTTP request body. This reduces handshake overhead on live publishes that produce dozens to hundreds of requests per minute.

### Technical detail

**Direction.** Outbound. The HTTP request line `PUT <path> HTTP/1.1` (or `DELETE <path> HTTP/1.1` for the cleanup channel — see the **HTTP DELETE** contract below) is the muxer's primary write channel to remote ingest endpoints.

**Format / Wire.** Three settings define the wire-format contract:

- **Request method.** The `set_http_options` helper at `[libavformat/hlsenc.c:L333-L350]` populates the dictionary that is passed to `s->io_open` for every HTTP-protocol open. When the user has set the `method` `AVOption` (declared at `[libavformat/hlsenc.c:L3165]`, default NULL), `c->method` is written verbatim into the dict at `[libavformat/hlsenc.c:L338]`. When the user has not set `method` and `s->url` resolves to an HTTP scheme via `ff_is_http_proto(s->url)` at `[libavformat/hlsenc.c:L335]`, the literal string `"PUT"` is written into the dict at `[libavformat/hlsenc.c:L340]`. This means the user-supplied `method` takes priority over the PUT default — a user that wants POST sets `-method POST` at the command line.
- **Persistent connection reuse.** When the `http_persistent` AVOption (declared at `[libavformat/hlsenc.c:L3176]`, default 0) is non-zero, `set_http_options` adds `multiple_requests=1` to the dictionary via `av_dict_set_int(options, "multiple_requests", 1, 0)` at `[libavformat/hlsenc.c:L345]`. This option is consumed by FFmpeg's HTTP protocol driver and instructs it to keep the TCP connection alive after a request body completes, so the next request on the same `AVIOContext` slot reuses the same connection.
- **Reuse mechanism.** The reuse logic lives in `hlsenc_io_open` at `[libavformat/hlsenc.c:L292-L311]`. When the function is called with `*pb` already populated (i.e., a previous open is still active), with an HTTP-protocol filename, AND with `hls->http_persistent` set, the cold-open path at `[libavformat/hlsenc.c:L299]` is skipped. Instead, the function retrieves the underlying URL context with `ffio_geturlcontext(*pb)` at `[libavformat/hlsenc.c:L302]` and calls `ff_http_do_new_request(http_url_context, filename)` at `[libavformat/hlsenc.c:L304]`. `ff_http_do_new_request` issues a new HTTP request on the existing TCP connection without tearing it down; the existing `AVIOContext` is repointed at the new request body. On failure, the code falls back to a cold open via `ff_format_io_close(s, pb)` at `[libavformat/hlsenc.c:L306]` and the next call will take the `!*pb` branch.

**Default.** `method` is unset (NULL) → the muxer emits `PUT`. `http_persistent` is `0` → each segment upload is a fresh `Connection: close` cycle. `timeout` is `-1` (no timeout) at `[libavformat/hlsenc.c:L3177]`. `http_user_agent` is NULL at `[libavformat/hlsenc.c:L3171]` (FFmpeg's built-in `User-Agent` is used).

**Optionality.** The HTTP-specific paths fire only when `s->url` is an HTTP/HTTPS scheme — detected via `ff_is_http_proto(s->url)` at `[libavformat/hlsenc.c:L335]`. For file-scheme URLs (no `scheme://` prefix or `file://...`), neither `method=PUT` nor `multiple_requests=1` is added to the dictionary, and the cold-open path of `hlsenc_io_open` at `[libavformat/hlsenc.c:L299]` is always taken because no persistent connection makes sense for local files.

**Failure mode if violated.** Three failure modes are concrete:

1. A CDN ingest endpoint that does not accept `PUT` (e.g., one expecting `POST` with multipart form data) will reject every segment upload with HTTP 405 Method Not Allowed. The workaround is to set the `method` AVOption explicitly, e.g., `-method POST`. A port that hard-codes a different default method (e.g., `POST` instead of `PUT`) shifts the burden of explicit method selection to users of the original endpoint shape.
2. With `http_persistent=1`, a server that closes the TCP connection between requests (HTTP/1.1 server with aggressive keep-alive timeout, or a connection-pool intermediary like a load balancer that rotates connections) will produce a sequence of cold-open requests despite the muxer's persistent setting — the reuse contract is honoured but the wire benefit is lost. The muxer detects each cold open as a fall-back from the failure-recovery branch at `[libavformat/hlsenc.c:L305-L306]`.
3. The exclusion of `crypto:`-wrapped writes from the persistent path (see the guard at `[libavformat/hlsenc.c:L320]` in `hlsenc_io_close`: `!http_base_proto || !hls->http_persistent || hls->key_info_file || hls->encrypt`) is deliberate. The `crypto:` pseudo-protocol holds AES-CBC state across writes; reusing the underlying HTTP socket for a fresh `crypto:` open without re-initialising the cipher would corrupt subsequent segments. A port that lifts this exclusion silently produces unreadable segments after the first key rotation.

---

## Contract: Variant-Stream BANDWIDTH Annotation

Master playlists for multi-variant publishes (adaptive bitrate ladders) list each variant via a `#EXT-X-STREAM-INF` line carrying its bandwidth in bits per second. Players use this `BANDWIDTH` value as the primary input to their adaptive-bitrate (ABR) selection logic — the highest-bandwidth variant the network can sustain is chosen and re-evaluated periodically as conditions change. The `BANDWIDTH` value is mandatory; the line also carries optional `AVERAGE-BANDWIDTH`, `RESOLUTION`, `CODECS`, `AUDIO`, `CLOSED-CAPTIONS`, and `SUBTITLES` attributes whose emission is conditional on the variant's content.

### Technical detail

**Direction.** Outbound. The line is part of the master playlist that every multi-variant player reads first.

**Format / Wire.** The entire `#EXT-X-STREAM-INF` line plus its trailing URL is emitted by `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]`. The function signature accepts `bandwidth`, `avg_bandwidth`, `filename`, `agroup`, `codecs`, `ccgroup`, and `sgroup` parameters at `[libavformat/hlsplaylist.c:L78-L82]`. The emission proceeds as a sequence of conditional `avio_printf` calls in the following order:

```text
#EXT-X-STREAM-INF:BANDWIDTH=<bps>[,AVERAGE-BANDWIDTH=<bps>][,RESOLUTION=<w>x<h>][,CODECS="..."][,AUDIO="group_<agroup>"][,CLOSED-CAPTIONS="<ccgroup>"][,SUBTITLES="<sgroup>"]
<variant-playlist-URL>
```

The literal `#EXT-X-STREAM-INF:BANDWIDTH=%d` is emitted at `[libavformat/hlsplaylist.c:L93]`. The trailing `\n%s\n\n` at `[libavformat/hlsplaylist.c:L107]` writes the variant playlist URL on its own line followed by a blank line. The conditional attributes are gated by the following predicates:

| Attribute | Predicate | Emitted at |
|---|---|---|
| `,AVERAGE-BANDWIDTH=%d` | `avg_bandwidth != 0` | `[libavformat/hlsplaylist.c:L94-L95]` |
| `,RESOLUTION=%dx%d` | `st && st->codecpar->width > 0 && st->codecpar->height > 0` | `[libavformat/hlsplaylist.c:L96-L98]` |
| `,CODECS="%s"` | `codecs && codecs[0]` (non-empty C string) | `[libavformat/hlsplaylist.c:L99-L100]` |
| `,AUDIO="group_%s"` | `agroup && agroup[0]` | `[libavformat/hlsplaylist.c:L101-L102]` |
| `,CLOSED-CAPTIONS="%s"` | `ccgroup && ccgroup[0]` | `[libavformat/hlsplaylist.c:L103-L104]` |
| `,SUBTITLES="%s"` | `sgroup && sgroup[0]` | `[libavformat/hlsplaylist.c:L105-L106]` |

The `BANDWIDTH` line is the mandatory anchor; every other attribute follows on the same physical line, comma-separated, in the order shown.

**Default.** `BANDWIDTH` is computed prior to the call by the caller — typically derived from the variant's accumulated `max_bitrate` (peak observed bitrate across all written segments) — and `AVERAGE-BANDWIDTH` from `total_size` (bytes) over `total_duration` (seconds) `[inferred — derivation occurs in hlsenc.c create_master_playlist, called prior to `ff_hls_write_stream_info`]`. `RESOLUTION` derives from the variant's reference stream's `AVCodecParameters::width` and `::height`. `CODECS` derives from per-stream `write_codec_attr` accumulation.

**Optionality.** `BANDWIDTH` is mandatory by spec and by code: a call to `ff_hls_write_stream_info` with `bandwidth == 0` logs a warning ("Bandwidth info not available, set audio and video bitrates") and returns *without emitting any line* — see the guard at `[libavformat/hlsplaylist.c:L87-L91]`. This means a variant whose bitrate could not be determined will silently disappear from the master playlist. All other attributes are conditional per the predicate table above.

**Failure mode if violated.** Three failure modes are concrete:

1. An `EXT-X-STREAM-INF` line without `BANDWIDTH` is invalid per RFC 8216 §4.4.5.2 and players reject the master playlist (typically "Invalid M3U8" or "Unparseable variant"). The code prevents this by never emitting the line at all when `bandwidth == 0`; a port that emits a `BANDWIDTH=0` placeholder, or that omits the attribute entirely, would produce an invalid playlist.
2. Reordering the attributes (e.g., placing `RESOLUTION` before `BANDWIDTH`) is technically permitted by RFC 8216 attribute-list parsing rules, but some strict-validator player implementations reject any line whose first attribute is not `BANDWIDTH`. The existing implementation always places `BANDWIDTH` first via the unconditional `avio_printf` at `[libavformat/hlsplaylist.c:L93]` — a port that re-orders would break those validators.
3. The `AUDIO="group_<agroup>"` attribute uses the literal `group_` prefix prepended to the agroup string at `[libavformat/hlsplaylist.c:L102]`. This prefix must match the same prefix used by `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L47]` (`GROUP-ID="group_%s"`). A port that drops the prefix from one but not the other breaks the player's match between `#EXT-X-STREAM-INF:AUDIO=...` and `#EXT-X-MEDIA:GROUP-ID=...`, silently disabling audio selection.

---

## Contract: HLS Sample Encryption Transport

HLS Sample Encryption (Sample-AES) is Apple's *per-frame* encryption scheme for HLS, distinct from the full-segment AES-128 mode the muxer implements. The MPEG-TS demuxer recognises Sample-AES streams via four custom `STREAM_TYPE_HLS_SE_*` constants declared in `libavformat/mpegts.h`; a Sample-AES-capable demuxer treats these stream types as encrypted variants of the corresponding standard codecs (H.264, AAC, AC-3, E-AC-3) and decrypts each frame in place. The FFmpeg HLS *muxer* does not produce Sample-AES content — only the demuxer side of this contract is implemented.

### Technical detail

**Direction.** Inbound (to the demuxer only). The demuxer recognises these stream-type values in MPEG-TS payload coming from external Sample-AES-producing encoders or live origins.

**Format / Wire.** Sample-AES streams are signalled in the MPEG-TS Program Map Table (PMT) via the following four stream-type byte values, all defined together as a contiguous block at `[libavformat/mpegts.h:L177-L180]`:

| Constant | Hex Value | Corresponding Plain Codec | Citation |
|---|---|---|---|
| `STREAM_TYPE_HLS_SE_VIDEO_H264` | `0xdb` | H.264 video (`STREAM_TYPE_VIDEO_H264 = 0x1b`) | `[libavformat/mpegts.h:L177]` |
| `STREAM_TYPE_HLS_SE_AUDIO_AAC` | `0xcf` | AAC audio (`STREAM_TYPE_AUDIO_AAC = 0x0f`) | `[libavformat/mpegts.h:L178]` |
| `STREAM_TYPE_HLS_SE_AUDIO_AC3` | `0xc1` | AC-3 audio (`STREAM_TYPE_AUDIO_AC3 = 0x81`) | `[libavformat/mpegts.h:L179]` |
| `STREAM_TYPE_HLS_SE_AUDIO_EAC3` | `0xc2` | E-AC-3 audio (`STREAM_TYPE_AUDIO_EAC3 = 0x87`) | `[libavformat/mpegts.h:L180]` |

The comment immediately preceding the constants names the Apple specification: *"HTTP Live Streaming (HLS) Sample Encryption / see 'MPEG-2 Stream Encryption Format for HTTP Live Streaming', https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/HLS_Sample_Encryption/"* at `[libavformat/mpegts.h:L174-L176]`. The Sample-AES decryption path itself is implemented in `libavformat/hls_sample_encryption.c`; the public entry points are declared at `[libavformat/hls_sample_encryption.h:L58-L62]` (`ff_hls_senc_read_audio_setup_info`, `ff_hls_senc_parse_audio_setup_info`, `ff_hls_senc_decrypt_frame`).

The encryption key for the per-frame mode is fetched the same way as the full-segment AES-128 key — via the `EXT-X-KEY` line in the playlist — but with `METHOD=SAMPLE-AES` instead of `METHOD=AES-128`. The demuxer's key-type discrimination is the `KeyType` enum at `[libavformat/hls.c:L71-L74]` with values `KEY_NONE`, `KEY_AES_128`, and `KEY_SAMPLE_AES`. See [`./functional-invariants.md`](functional-invariants.md) for the invariant that the muxer's `METHOD=AES-128` line is always literal (not `SAMPLE-AES`).

**Default.** The constants are present unconditionally in `libavformat/mpegts.h`. Detection at runtime is automatic: the MPEG-TS demuxer's stream-type switch in `libavformat/mpegts.c` maps each of the four bytes to its corresponding codec ID and marks the resulting `AVStream` as encrypted. No `AVOption` controls or disables Sample-AES recognition.

**Optionality.** The Sample-AES transport contract is exercised by the demuxer only when an inbound MPEG-TS payload's PMT carries one of the four stream-type bytes. For inbound MPEG-TS without these values, the demuxer takes the standard non-encrypted decode path. The muxer side has no Sample-AES path at all — the build-time separation `[libavformat/Makefile:L276-L277]` confirms that `hls_sample_encryption.o` is linked into `OBJS-$(CONFIG_HLS_DEMUXER)` and not into `OBJS-$(CONFIG_HLS_MUXER)`.

**Failure mode if violated.** Three failure modes are concrete:

1. A demuxer port that omits any of the four `STREAM_TYPE_HLS_SE_*` values will misclassify Sample-AES streams as "unknown stream type" — the affected streams' packets are silently dropped during PMT parsing, and the user observes a video or audio track that decoders never see.
2. A demuxer port that confuses the values (e.g., maps `0xdb` to AAC instead of H.264) silently mis-decodes encrypted frames. The decoder will produce noise or refuse to decode entirely because the bytes do not match the codec it has been told to use.
3. A muxer port that *adds* Sample-AES emission (a non-trivial change requiring frame-level encryption, audio-setup-info parsing, and AAC priming as documented in `libavformat/hls_sample_encryption.c`) crosses the muxer/demuxer build-time boundary and changes the muxer's emitted contract surface. The existing FFmpeg muxer deliberately stays out of this scheme.

---

## Contract: EXT-X-KEY METHOD Line Layout

Every encrypted segment range in a media playlist is preceded by an `#EXT-X-KEY` line that tells the player which key URI to fetch and, optionally, which initialization vector to use. The FFmpeg HLS muxer emits exactly one form of this line for its full-segment AES-128 mode: `#EXT-X-KEY:METHOD=AES-128,URI="<key-uri>"` followed optionally by `,IV=0x<32-hex-chars>`. A new `EXT-X-KEY` line is emitted only when the URI or the IV changes from the previously-emitted line, so a steady-state encrypted stream with a stable key emits this line exactly once at the start of the playlist.

### Technical detail

**Direction.** Outbound. The line is emitted inside the per-playlist segment-iteration loop in `hls_window`.

**Format / Wire.** The emission is implemented in `hls_window` at `[libavformat/hlsenc.c:L1600-L1608]`. The full assembled line is:

```text
#EXT-X-KEY:METHOD=AES-128,URI="<key-uri>"[,IV=0x<32-hex-chars>]
```

Concretely, the emission breaks into four sub-printfs gated by a single conditional:

- **Conditional gate.** The block at `[libavformat/hlsenc.c:L1601-L1602]` checks `(hls->encrypt || hls->key_info_file)` — encryption must be active — AND `(!key_uri || strcmp(en->key_uri, key_uri) || av_strcasecmp(en->iv_string, iv_string))` — this is the first segment of the playlist OR the URI changed OR the IV changed (case-insensitive comparison on the hex IV string).
- **METHOD + URI emission.** At `[libavformat/hlsenc.c:L1603]`: `avio_printf(..., "#EXT-X-KEY:METHOD=AES-128,URI=\"%s\"", en->key_uri)`. The literal `METHOD=AES-128` is hard-coded; the URI is substituted from the per-segment `HLSSegment::key_uri` field (see [`../technical/data-model.md`](../technical/data-model.md) for the field definition).
- **Optional IV emission.** At `[libavformat/hlsenc.c:L1604-L1605]`: `if (*en->iv_string) avio_printf(..., ",IV=0x%s", en->iv_string)`. The IV is emitted only when the segment's `iv_string` field is non-empty (zero-length is treated as "no IV"). The hex string is 32 characters (KEYSIZE × 2 = 16 bytes × 2 hex digits per byte = 32) produced by `ff_data_to_hex` in `do_encrypt`/`hls_encryption_start`.
- **Line terminator.** At `[libavformat/hlsenc.c:L1606]`: `avio_printf(..., "\n")`. A single LF terminator (no CRLF) consistent with the rest of the M3U8 emission.
- **Cache update.** At `[libavformat/hlsenc.c:L1607-L1608]`: `key_uri = en->key_uri; iv_string = en->iv_string;` updates the cached values for the next-segment comparison.

The `key_uri` and `iv_string` cache variables in `hls_window` start as NULL/empty for the first segment iteration, so the first encrypted segment always emits a fresh `EXT-X-KEY` line. Subsequent segments emit a new `EXT-X-KEY` line only when the URI or the IV changes — typical periodic-rekey flows produce one new `EXT-X-KEY` line per key-period boundary.

The IV format on the wire is `0x` followed by exactly 32 lowercase hexadecimal digits (`[0-9a-f]`), representing the 16 raw bytes of the AES-128 IV. The `KEYSIZE 16` constant at `[libavformat/hlsenc.c:L70]` is the source of the 16-byte size; the IV is stored as a `char[KEYSIZE*2 + 1]` (33-byte) buffer to accommodate the hex encoding plus NUL terminator. See [`./data-contracts.md`](data-contracts.md) for the IV derivation rules when the `hls_enc_iv` option is unset (sequence-number-based derivation).

**Default.** When neither `hls_enc` nor `hls_key_info_file` is set, the muxer emits no `EXT-X-KEY` lines at all. The `key_uri` value in the emitted line comes from the muxer's per-session encryption state — either the user-supplied `hls_enc_key_url` or the auto-derived `<basename>.key` URI from `do_encrypt`.

**Optionality.** Gated by `(hls->encrypt || hls->key_info_file)` at `[libavformat/hlsenc.c:L1601]`. The optional `,IV=...` portion is gated by `*en->iv_string` (non-empty per-segment IV string) at `[libavformat/hlsenc.c:L1604]`.

**Failure mode if violated.** Three failure modes are concrete:

1. A port that emits `METHOD` with anything other than `AES-128` for the muxer's full-segment encryption (e.g., `SAMPLE-AES`, `AES-256`, `NONE`) produces playlists incompatible with HLS-spec-compliant players for the full-segment mode. The literal `AES-128` is hard-coded in the printf format string at `[libavformat/hlsenc.c:L1603]` — modifying it to a parameterised value is a wire-format change.
2. A port that drops the `0x` prefix from the IV (emitting `IV=<hex>` instead of `IV=0x<hex>`) breaks RFC 8216 §4.4.4.4 IV-attribute syntax. Players reject the IV silently and use a zero-IV fallback, producing decryption noise on every segment whose actual IV was non-zero.
3. A port that always emits the `EXT-X-KEY` line per segment (omitting the URI/IV change check at `[libavformat/hlsenc.c:L1601-L1602]`) bloats the playlist with redundant lines and confuses some player implementations that expect the line to mark a *change* in key — not a per-segment reiteration. The existing single-emission-per-change behaviour is part of the wire contract.

---

## Contract: HTTP DELETE for Old Segment Cleanup

When the live sliding-window mode is engaged AND the `hls_flags=delete_segments` flag is set, the muxer issues HTTP `DELETE` requests against expired segments after they age out of the sliding window. A dedicated `AVIOContext` field on the `HLSContext` struct (`http_delete`) holds the connection used for these requests, kept separate from the per-variant playlist and segment write paths so a DELETE failure does not corrupt the active upload channel.

### Technical detail

**Direction.** Outbound. The `DELETE <path> HTTP/1.1` request line is sent to whatever CDN endpoint hosted the original `PUT <path>`.

**Format / Wire.** The cleanup flow lives in `hls_delete_old_segments` at `[libavformat/hlsenc.c:L531-L639]`. For each segment whose presence in the playlist has been superseded by newer segments and whose age exceeds the `hls_delete_threshold` retention count, the function resolves the segment's filename and proto via standard FFmpeg URL parsing. The HTTP-vs-local-file decision branch is at `[libavformat/hlsenc.c:L510-L527]`:

- **HTTP branch.** When `hls->method` is set OR the resolved protocol is `http` (case-insensitive match), the function builds an option dictionary with `method=DELETE` via `av_dict_set(&opt, "method", "DELETE", 0)` at `[libavformat/hlsenc.c:L515]`, opens (or reuses) the `http_delete` AVIOContext slot via `hlsenc_io_open(avf, &hls->http_delete, path, &opt)` at `[libavformat/hlsenc.c:L517]`, then immediately closes it via `hlsenc_io_close(avf, &hls->http_delete, path)` at `[libavformat/hlsenc.c:L523]`. No request body is written — the DELETE method is bodyless. The `http_delete` slot is the dedicated AVIOContext field on `HLSContext` at `[libavformat/hlsenc.c:L261]`; reusing it across DELETEs lets `http_persistent` benefit (when set) accrue to the cleanup channel.
- **Local-file branch.** When the protocol is not HTTP, the function calls POSIX `unlink(path)` at `[libavformat/hlsenc.c:L524]` and logs a warning on failure.

The DELETE channel is opened at session start (lazily on first DELETE) and closed in `hls_deinit` via `ff_format_io_close(s, &hls->http_delete)` at `[libavformat/hlsenc.c:L2720]`.

**Default.** Disabled. The `HLS_DELETE_SEGMENTS` flag bit (`1 << 1`) is defined at `[libavformat/hlsenc.c:L99]` and is one of the constants in the `HLSFlags` enum (`[libavformat/hlsenc.c:L96-L113]`); it must be explicitly set via `hls_flags=delete_segments` (the named constant alias is declared in the `options[]` array at `[libavformat/hlsenc.c:L3147]`).

**Optionality.** Gated by the `HLS_DELETE_SEGMENTS` flag check inside the `hls_delete_old_segments` invocation path (the flag bit defined at `[libavformat/hlsenc.c:L99]` is one of the `HLSFlags` enum values at `[libavformat/hlsenc.c:L96-L113]`). When the flag is unset, no DELETE requests are ever issued — expired segments remain on disk or on the CDN. The flag combines with `hls_delete_threshold` (default `1` at `[libavformat/hlsenc.c:L3126]`) to control how many segments past the playlist's tail are retained before deletion.

**Failure mode if violated.** Three failure modes are concrete:

1. A CDN that does not accept `DELETE` (most static-file CDNs do not — DELETE typically requires explicit origin-server cooperation) will return HTTP 405 Method Not Allowed or 403 Forbidden on every cleanup request. When `hls->ignore_io_errors=1`, the failure is swallowed and old segments accumulate indefinitely on the CDN; when `=0`, the error propagates through `hls_write_packet` and aborts the encoding mid-stream. The mitigating return-value branch at `[libavformat/hlsenc.c:L519-L520]` (`return hls->ignore_io_errors ? 1 : ret`) implements this fork.
2. A port that consolidates the `http_delete` AVIOContext slot with the per-variant `vs->out` slot would couple DELETE failures to the segment-write channel. The current isolation lets a failed DELETE not affect the next segment PUT — preserving this isolation is a porting concern.
3. A port that issues DELETE without setting `method=DELETE` (e.g., reusing the default `PUT`-bound option dictionary) sends a `PUT /path HTTP/1.1` with no body to the same path — overwriting the segment with empty content rather than deleting it. The `method=DELETE` set at `[libavformat/hlsenc.c:L515]` is the override that distinguishes cleanup from upload.

---

## Contract: EXT-X-MEDIA Audio/Subtitle Rendition Format

Master playlists for multi-track streams (multiple audio languages, multiple subtitle tracks, multiple closed-captions tracks) advertise the alternative renditions via `#EXT-X-MEDIA` lines, one per rendition. Each line carries a TYPE, GROUP-ID, NAME, DEFAULT flag, optional LANGUAGE, and a URI. The attribute order, attribute quoting, and the literal `group_` prefix on audio GROUP-IDs are part of the wire contract that match-by-GROUP-ID players parse strictly.

### Technical detail

**Direction.** Outbound. The lines are emitted into the master playlist before the `#EXT-X-STREAM-INF` lines that reference them by group ID.

**Format / Wire.** Two functions emit `#EXT-X-MEDIA` lines, one per rendition type:

- **Audio rendition** — `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L40-L56]`. The emission proceeds as a sequence of `avio_printf` calls in this order:
  ```text
  #EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="group_<agroup>",NAME="audio_<name_id>",DEFAULT=<YES|NO>,[LANGUAGE="<lang>",][CHANNELS="<n>",]URI="<filename>"
  ```
  The literal `TYPE=AUDIO` and the `group_` prefix on the GROUP-ID are at `[libavformat/hlsplaylist.c:L47]`. The NAME is constructed as `"audio_%d"` with `name_id` at `[libavformat/hlsplaylist.c:L48]`. DEFAULT renders `YES` or `NO` based on the `is_default` parameter at `[libavformat/hlsplaylist.c:L48]`. Optional `LANGUAGE="<lang>"` is emitted at `[libavformat/hlsplaylist.c:L49-L51]` when `language` is non-NULL. Optional `CHANNELS="<n>"` is emitted at `[libavformat/hlsplaylist.c:L52-L54]` when `nb_channels` is non-zero. The trailing `URI="<filename>"` is emitted at `[libavformat/hlsplaylist.c:L55]`.

- **Subtitle rendition** — `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]`. The emission proceeds as a sequence of `avio_printf` calls in this order:
  ```text
  #EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="<sgroup>",NAME="<sname-or-subtitle_id>",DEFAULT=<YES|NO>,[LANGUAGE="<lang>",]URI="<filename>"
  ```
  The literal `TYPE=SUBTITLES` and the un-prefixed GROUP-ID are at `[libavformat/hlsplaylist.c:L65]`. Note the asymmetry: the audio rendition prepends `group_` to its agroup string, but the subtitle rendition emits the sgroup verbatim. NAME defaults to `"subtitle_%d"` with `name_id` when `sname` is NULL at `[libavformat/hlsplaylist.c:L66-L70]`; otherwise the supplied `sname` is used. DEFAULT renders `YES` or `NO` at `[libavformat/hlsplaylist.c:L71]`. Optional `LANGUAGE="<lang>"` at `[libavformat/hlsplaylist.c:L72-L74]`. Trailing `URI="<filename>"` at `[libavformat/hlsplaylist.c:L75]`.

**Default.** No `#EXT-X-MEDIA` lines are emitted unless the variant-stream configuration declares an audio group (`agroup`) or subtitle group (`sgroup`). The default of `hls_subtitle_path` at `[libavformat/hlsenc.c:L3138]` is NULL; the default for variant audio/subtitle groups is determined by the `var_stream_map` option (default NULL at `[libavformat/hlsenc.c:L3172]`).

**Optionality.** Per-variant. A `var_stream_map` value of `"v:0,a:0,agroup:aud,language:eng,default:yes"` declares an audio group; subtitle groups are similarly declared via `sgroup:` attribute syntax. Closed-captions groups (`CLOSED-CAPTIONS=...` on `#EXT-X-STREAM-INF` lines) use `ccgroup:` syntax and are advertised via separate `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` lines `[inferred — closed-captions emission shape is constructed in hlsenc.c via cc_stream_map, parallel to agroup but with TYPE=CLOSED-CAPTIONS]`.

**Failure mode if violated.** Three failure modes are concrete:

1. A port that drops the `group_` prefix from the audio GROUP-ID (changing `"group_%s"` to `"%s"` at `[libavformat/hlsplaylist.c:L47]`) without also changing the corresponding `AUDIO="group_%s"` emission at `[libavformat/hlsplaylist.c:L102]` breaks the variant-to-rendition match in the master playlist. Players see `AUDIO="group_eng"` on the `EXT-X-STREAM-INF` line but `GROUP-ID="eng"` on the `EXT-X-MEDIA` line and conclude that no audio rendition matches — audio selection silently fails. The two prefixes must remain symmetric.
2. A port that adds a `group_` prefix to the subtitle GROUP-ID (mirroring the audio asymmetry) at `[libavformat/hlsplaylist.c:L65]` breaks players that match the playlist's `SUBTITLES=` attribute literally. The existing asymmetry is part of the wire contract; preserving it across a port is required for compatibility with downstream tooling that has internalised the asymmetry.
3. A port that changes the attribute order (e.g., places `URI=...` before `NAME=...`) is technically permitted by RFC 8216 attribute-list parsing, but breaks strict-validator implementations that key off attribute order for performance. The existing order is `TYPE`, `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE` (optional), `CHANNELS` (audio-only, optional), `URI` — a port should preserve this canonical order.

---

## Contract: EXT-X-PROGRAM-DATE-TIME ISO-8601 Format

When wall-clock anchoring is enabled via the `hls_flags=program_date_time` flag, each segment in the playlist is preceded by an `#EXT-X-PROGRAM-DATE-TIME` line stamping it with an ISO-8601 timestamp at millisecond precision. Players use this timestamp to correlate playlist segments with absolute wall-clock time (essential for live event streams that must sync with external time references) and to surface "live edge" indicators to viewers.

### Technical detail

**Direction.** Outbound. The line is part of every per-segment entry in the media playlist when the flag is active.

**Format / Wire.** The full line format is:

```text
#EXT-X-PROGRAM-DATE-TIME:YYYY-MM-DDTHH:MM:SS.mmm±HHMM
```

The emission is inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L167-L192]`. The construction proceeds in five steps:

- **Seconds-since-epoch extraction.** `tt = (int64_t)*prog_date_time` at `[libavformat/hlsplaylist.c:L172]` casts the `double prog_date_time` to integer seconds since the Unix epoch.
- **Millisecond extraction with clamping.** `milli = av_clip(lrint(1000*(*prog_date_time - tt)), 0, 999)` at `[libavformat/hlsplaylist.c:L173]` extracts the fractional-seconds component, multiplies by 1000, rounds to the nearest integer, and clamps to the closed interval `[0, 999]` so the printed ms field is always exactly three digits.
- **Date/time formatting.** `localtime_r(&tt, &tmpbuf)` at `[libavformat/hlsplaylist.c:L174]` produces a `struct tm` in the host's local timezone. `strftime(buf0, sizeof(buf0), "%Y-%m-%dT%H:%M:%S", tm)` at `[libavformat/hlsplaylist.c:L175]` formats the date portion in ISO-8601 extended format.
- **Timezone offset.** The primary path `strftime(buf1, sizeof(buf1), "%z", tm)` at `[libavformat/hlsplaylist.c:L179]` produces `±HHMM` via the platform's strftime support. The fallback path at `[libavformat/hlsplaylist.c:L180-L189]` is taken when `%z` is not supported or when the formatted string begins with an unexpected character; it computes the offset manually by comparing `localtime_r` with `gmtime_r` results and emits `snprintf(buf1, sizeof(buf1), "%c%02d%02d", sign, hours, minutes)` at `[libavformat/hlsplaylist.c:L185-L189]`.
- **Final emission.** `avio_printf(out, "#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s\n", buf0, milli, buf1)` at `[libavformat/hlsplaylist.c:L191]` writes the assembled line. The `%03d` format specifier guarantees exactly three digits (zero-padded on the left); `buf0` carries the date-up-to-seconds part and `buf1` carries the `±HHMM` offset.

The per-segment `*prog_date_time` value is advanced after the emission via `*prog_date_time += duration` at `[libavformat/hlsplaylist.c:L192]`, so the next segment's timestamp equals the previous segment's timestamp plus its duration in seconds. Discontinuity-induced re-anchoring is implemented separately via the per-segment `HLSSegment::discont_program_date_time` field (see [`../technical/data-model.md`](../technical/data-model.md)) — when a discontinuity is detected, `vs->last_segment->discont_program_date_time` is overwritten with a freshly-computed wall-clock time at `[libavformat/hlsenc.c:L1273-L1275]`, which then takes priority over the rolling `prog_date_time` value at the next emission.

**Default.** Disabled. The line is emitted only when the `HLS_PROGRAM_DATE_TIME` flag bit (`1 << 7`) at `[libavformat/hlsenc.c:L105]` is set on `hls->flags`. The named constant alias is declared in the `options[]` array at `[libavformat/hlsenc.c:L3153]` as `program_date_time` (i.e., `hls_flags=program_date_time` enables it).

**Optionality.** The decision to pass a non-NULL `prog_date_time` pointer into `ff_hls_write_file_entry` is made by the caller `hls_window` at `[libavformat/hlsenc.c:L1548]`: `double *prog_date_time_p = (hls->flags & HLS_PROGRAM_DATE_TIME) ? &prog_date_time : NULL`. The `prog_date_time` variable is initialised once per playlist publish from `vs->initial_prog_date_time` and accumulates segment-by-segment.

**Failure mode if violated.** Three failure modes are concrete:

1. A port that emits a format other than ISO-8601 extended (e.g., `YYYY-MM-DD HH:MM:SS` with a space separator instead of `T`, or `YYYY/MM/DD`) will be rejected by players that strictly parse RFC 8216 §4.4.5.1, which mandates ISO-8601. Players that tolerate the format silently may still mis-compute live-edge offsets, surfacing as incorrect "X seconds behind live" indicators.
2. A port that emits milliseconds without zero-padding (e.g., `.5` instead of `.500`) or with more than three digits (`.500000` for microseconds) breaks the `%03d` contract. The fractional component is required to be exactly three digits per the existing format string at `[libavformat/hlsplaylist.c:L191]`.
3. A port that emits the timezone offset in any form other than `±HHMM` (no colon — e.g., not `±HH:MM`, not `Z` for UTC) violates the platform's `%z` strftime contract. The HLS spec allows both `±HHMM` and `±HH:MM`, but downstream tooling that parses with a strict `%z`-style regex breaks if the format is changed.

---

## Cross-References

This document interacts with five other leaves of the documentation set:

- [`./functional-invariants.md`](functional-invariants.md) — for the MUST/MUST NOT statements that the contracts here implement. Most contracts in this document have a corresponding behavioural invariant; for example, the EXT-X-KEY line layout contract here corresponds to the "EXT-X-KEY METHOD=AES-128 line precedes encrypted segment entries" invariant.
- [`./data-contracts.md`](data-contracts.md) — for the `AVOption` types, defaults, bounds, and binary layouts that the contracts here exchange. The full struct field tables for `HLSContext`, `VariantStream`, and `HLSSegment` (with type, line, and effect for every field) are canonicalised there; this document cites struct fields by name but does not duplicate their definitions.
- [`./timing-dependencies.md`](timing-dependencies.md) — for the ordering that constrains when each contract is exercised. For example, the AES-128 key MUST be installed before the first encrypted segment is written; the fMP4 init segment MUST be materialised before the playlist references it via `EXT-X-MAP`. These ordering rules are stated as timing contracts in that companion document.
- [`../technical/integration-interfaces.md`](../technical/integration-interfaces.md) — for the implementation-side narrative of each interface. Each contract here has a corresponding "Interface" section there with full source narrative (e.g., the HTTP `http_persistent` reuse logic is described once narratively there and is referenced from this document for its wire-format observable surface).
- [`../functionality/exception-handling.md`](../functionality/exception-handling.md) — for the `AVERROR(*)` codes that surface when contracts are violated at runtime. The "Failure mode if violated" entries in each contract above are wire-format-focused; the recovery-and-error-code mapping is fully tabulated in the exception-handling document.

---

## Validation Checklist

The nine items below match the nine contracts above one-to-one. A reviewer of a port can walk this list top to bottom, confirming each item against the new implementation by running an integration test or inspecting an emitted artefact.

- [ ] **AES-128 Key URI Fetch.** Key file referenced by the URI is exactly 16 raw bytes (no header, no text encoding); URI is exactly the value of `hls_enc_key_url` or the first line of `hls_key_info_file`; `KEYSIZE 16` constant matches the `av_aes_init` `key_bits=128` requirement. Empty URI or empty key path in the key info file is rejected with `AVERROR(EINVAL)`.
- [ ] **fMP4 Initialization Segment Delivery.** `hls_fmp4_init_filename` defaults to `"init.mp4"`; the init file is materialised before the first `#EXT-X-MAP:URI="..."` line is written; `hls_fmp4_init_resend=1` re-writes the init file on every playlist refresh.
- [ ] **HTTP Chunked Transfer (PUT default + http_persistent).** `method=PUT` is the default added to the option dictionary when `s->url` is HTTP; user-supplied `method` AVOption overrides PUT; `multiple_requests=1` is added when `http_persistent=1`; `crypto:`-wrapped writes are explicitly excluded from the persistent path.
- [ ] **Variant-Stream BANDWIDTH Annotation.** `#EXT-X-STREAM-INF` always opens with `BANDWIDTH=<bps>`; AVERAGE-BANDWIDTH, RESOLUTION, CODECS, AUDIO, CLOSED-CAPTIONS, SUBTITLES are conditional on the documented predicates; a `bandwidth=0` call emits no line at all (warning logged).
- [ ] **HLS Sample Encryption Transport.** The four `STREAM_TYPE_HLS_SE_*` values (`0xdb`, `0xcf`, `0xc1`, `0xc2`) are recognised by the demuxer's PMT parser; the muxer side produces no Sample-AES content (`hls_sample_encryption.o` linked into the demuxer object set only).
- [ ] **EXT-X-KEY METHOD Line Layout.** The literal METHOD is `AES-128` (not `SAMPLE-AES` or any other value) for the muxer's full-segment mode; the `,IV=0x<32-hex-chars>` portion is conditional on a non-empty `iv_string`; a new line is emitted only when URI or IV changes from the previous line.
- [ ] **HTTP DELETE for Old Segment Cleanup.** A dedicated `http_delete` AVIOContext is used for cleanup; `method=DELETE` is set explicitly in the option dictionary; DELETE requests are gated by the `HLS_DELETE_SEGMENTS` flag; failures are swallowed when `ignore_io_errors=1` and propagated otherwise.
- [ ] **EXT-X-MEDIA Audio/Subtitle Rendition Format.** Audio GROUP-ID is prefixed with the literal `group_`; subtitle GROUP-ID is not prefixed; attribute order is `TYPE`, `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE` (optional), `CHANNELS` (audio-only, optional), `URI`; the audio rendition's `group_` prefix matches the `EXT-X-STREAM-INF:AUDIO="group_..."` prefix.
- [ ] **EXT-X-PROGRAM-DATE-TIME ISO-8601 Format.** Date is `YYYY-MM-DDTHH:MM:SS`; milliseconds are exactly three digits zero-padded (`.000` through `.999`); timezone is `±HHMM` (no colon); a discontinuity-induced `discont_program_date_time` value overrides the rolling `prog_date_time` for the affected segment.
