# Consumer Dependencies — Downstream Map of HLS Pipeline Outputs

This document maps the downstream consumers of the FFmpeg HLS pipeline and explains what would break if those consumers' expectations were violated. All source references in this document are anchored to commit `566ad786`. See [`../README.md`](../README.md) for the canonical commit-anchor banner.

---

## Overview

### Plain-Language Summary

The HLS muxer produces two artifacts: an M3U8 playlist (a small text file in a format defined by RFC 8216) and a stream of media segment files (either MPEG-TS `.ts` or fragmented MP4 `.m4s`). The HLS demuxer is the reverse — it reads an M3U8 from somewhere (local disk, HTTP origin, CDN edge) and emits decoded packets. Anything that reads those artifacts, or anything whose internals depend on the HLS muxer's helper code, is a *downstream consumer* of this pipeline.

Six categories of downstream consumer exist for the HLS pipeline:

1. **HLS-compliant media players** — the largest population: every mobile player, web player, smart-TV app, and set-top box that purports to play HLS. These are external to FFmpeg and are validated against RFC 8216 plus a long tail of player-specific quirks.
2. **CDN ingest endpoints** — HTTP origin servers that accept the muxer's segment uploads when its output URL is `http://` or `https://`. The CDN must accept the muxer's choice of HTTP method, allow long-running connections, and tolerate HTTP DELETE for old-segment cleanup.
3. **FFmpeg's own FATE test suite** — under `tests/`. The tests compare muxer and demuxer output to recorded reference data byte for byte; any change to emission shape invalidates them. This section is a *survey only* because the `tests/` tree is explicitly out of scope for this documentation effort.
4. **The in-tree DASH muxer** (`libavformat/dashenc.c`) — which reuses the HLS pipeline's playlist tag-writer translation unit (`hlsplaylist.o`) to emit HLS-format manifests alongside its native DASH MPD manifest. This is a build-time coupling that is easy to miss.
5. **The generic `segment` muxer** (`libavformat/segment.c`) — an alternative segmenter that can also emit M3U8, but does *not* share code with the HLS muxer and produces a slightly different manifest shape. It is sometimes confused with the HLS muxer; the existing user-facing documentation at `[doc/muxers.texi:L1909-L1911]` cross-references it as a "more generic and flexible implementation of a segmenter".
6. **External automation** — orchestration scripts, monitoring tools, packaging pipelines, and any custom code that tails the published M3U8 or measures segment cadence. These consumers depend on a stable line-by-line format.

### A Note on Format-Registration Flags

The HLS muxer registers itself with the flag combination `AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS` at `[libavformat/hlsenc.c:L3198]`. Each flag is a contract with FFmpeg's generic format-handling layer and, transitively, with every downstream consumer:

- `AVFMT_NOFILE` tells the generic layer not to pre-open an output file for the muxer; the muxer opens its own files (one per segment, plus one per playlist). Downstream code that wraps `avformat_write_header` should not also try to open `s->pb` for `-f hls`.
- `AVFMT_GLOBALHEADER` forces callers to set `AV_CODEC_FLAG_GLOBAL_HEADER` on their video and audio encoders before opening them. If a caller forgets, the encoder will not emit extradata into `codecpar`, and the HLS sub-muxer (MPEG-TS or fMP4) will not be able to write the codec-private data that downstream players need to initialize their decoders. This is a real-world integrator pain point and the most common cause of "the player plays nothing" tickets.
- `AVFMT_NODIMENSIONS` waives the libavformat-side requirement that video streams declare width and height before `write_header`. The HLS muxer instead reads `width`/`height` from `codecpar` lazily during `write_codec_attr` at `[libavformat/hlsenc.c:L352+]` to populate the `RESOLUTION=` field of `#EXT-X-STREAM-INF`.

### Reading the Categories Below

Each category section opens with a plain-language summary aimed at integrators, then provides engineer-facing detail with source citations, and closes with a *worked breakage scenario* — a concrete example of what fails when the contract is violated. The scenarios are intentionally pessimistic: the purpose of this document is to surface non-obvious dependencies *before* an integrator deploys.

For component-level descriptions of segment generation and playlist construction, see [`./functional-inventory.md`](./functional-inventory.md). For the full option and tag inventory, see [`./inputs-outputs.md`](./inputs-outputs.md). For the I/O failure model, see [`./exception-handling.md`](./exception-handling.md). For the formal HTTP, AES-128, and BANDWIDTH integration contracts referenced here, see [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md). For the M3U8 shape contract, see [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).

---

### Consumer: HLS-Compliant Media Players

#### Plain-Language Summary

Any client that purports to play HLS — iOS and tvOS native players, AVPlayer on macOS, Android ExoPlayer, hls.js in the browser, Roku channels, smart-TV firmware, professional broadcast monitoring tools — must accept the M3U8 manifest shape this muxer emits. Three shape contracts dominate everything else: the version number on line 2 of every playlist (which controls which other tags the client must understand), the media-sequence integer that lets the client deduplicate already-fetched segments, and the target-duration integer that tells the client how often to refresh.

#### Technical Detail

**EXT-X-VERSION negotiation contract.** The version number on line 2 of every playlist is computed at `[libavformat/hlsenc.c:L1551-L1571]` as a five-outcome decision matrix:

| Condition | Version Emitted | Source |
|---|---|---|
| Default with `HLS_ROUND_DURATIONS` flag set | 2 | `[libavformat/hlsenc.c:L1551]` |
| Default without `HLS_ROUND_DURATIONS` | 3 | `[libavformat/hlsenc.c:L1552-L1554]` |
| Byterange mode (`HLS_SINGLE_FILE` or `max_seg_size > 0`) | 4 (and `sequence` reset to 0) | `[libavformat/hlsenc.c:L1556-L1559]` |
| `HLS_I_FRAMES_ONLY` flag | 4 | `[libavformat/hlsenc.c:L1561-L1563]` |
| `HLS_INDEPENDENT_SEGMENTS` flag | 6 | `[libavformat/hlsenc.c:L1565-L1567]` |
| `segment_type == SEGMENT_TYPE_FMP4` | 7 | `[libavformat/hlsenc.c:L1569-L1571]` |

The emission itself happens at `[libavformat/hlsplaylist.c:L37]` (`#EXT-X-VERSION:%d\n`) from inside `ff_hls_write_playlist_version`, which is called as the second line of every playlist immediately after `#EXTM3U` at `[libavformat/hlsplaylist.c:L36]`. Players that pin to a specific version — for example, players that reject anything ≥ 5 because they were written against an older spec — will fail when this number changes. The cascading sequence in the decision table means a single option change (e.g., enabling `HLS_INDEPENDENT_SEGMENTS`) can silently bump the playlist version from 3 to 6.

**EXT-X-MEDIA-SEQUENCE monotonicity contract.** Each playlist refresh declares a media-sequence integer at `[libavformat/hlsplaylist.c:L121]` (`#EXT-X-MEDIA-SEQUENCE:%"PRId64"\n`). It MUST be monotonically non-decreasing across refreshes; the muxer computes it as `FFMAX(hls->start_sequence, vs->sequence - vs->nb_entries)` at `[libavformat/hlsenc.c:L1539]`. Players use this integer to deduplicate already-fetched segments: when a refreshed playlist appears with media-sequence N, the client knows that segment indices `N..N+nb_entries-1` are listed in this refresh and that anything below N has been evicted from the live window. A non-monotonic decrement would cause the client to re-download segments it has already cached, or worse, re-render content it has already played.

**EXT-X-TARGETDURATION accuracy contract.** The target duration is emitted as an integer at `[libavformat/hlsplaylist.c:L120]` (`#EXT-X-TARGETDURATION:%d\n`), computed in `hls_window` at `[libavformat/hlsenc.c:L1584-L1587]` by walking every segment currently in the playlist and taking `target_duration = lrint(en->duration)` (the largest, rounded). RFC 8216 (URL referenced inline at `[libavformat/hls.c:L26]`) requires every segment duration to be ≤ TARGETDURATION; a compliant player aborts the stream with a "segment duration exceeds target" error otherwise. Several players also use target-duration to size their playlist refresh interval (typically half the target duration in live mode), so a wrong value causes either stuttering (target too high → not enough refresh) or unnecessary load (target too low → excessive refresh).

#### Worked Breakage Scenario

Suppose a refactor of `hls_window` "optimized" the target-duration computation by reading only the first segment's duration instead of taking the max across all segments. The change would be invisible for steady-state CBR streams where every segment is exactly `hls_time` seconds long. But the moment a single segment runs long — for example because a GOP ended slightly later than expected, or because a discontinuity forced an early or late cut — the published target would be too small. Every compliant player would either:

- reject the manifest with a "segment duration exceeds target" error (strict-mode players), or
- silently mis-schedule its refresh interval, causing live latency to grow without bound (lenient-mode players).

The bug would not be reproducible under the FATE test suite (whose reference streams are CBR), would not surface until production deployment, and would be diagnosed only via player-side telemetry. The existing logic that walks all segments via the for-loop at `[libavformat/hlsenc.c:L1584-L1587]` is therefore a load-bearing piece of correctness, not a redundant safety check.

---

### Consumer: CDN Ingest (HTTP Origin Servers)

#### Plain-Language Summary

When the muxer's output URL is an `http://` or `https://` URL, the M3U8 playlist and every segment file are uploaded to a remote origin server instead of written to local disk. The CDN ingest endpoint that receives those uploads is a downstream consumer with its own contract: it must accept the muxer's choice of HTTP method (PUT by default), allow long-running persistent connections (which the muxer reuses to amortize TLS handshake cost), and tolerate HTTP DELETE for old-segment cleanup in live sliding-window mode.

#### Technical Detail

**HTTP PUT default.** In `set_http_options` at `[libavformat/hlsenc.c:L337-L341]`, when the user does not explicitly set the `method` AVOption, the muxer forces `method=PUT` for any HTTP-protocol output URL:

| Condition | HTTP Method Used | Source |
|---|---|---|
| User sets `method` option explicitly | The user's value | `[libavformat/hlsenc.c:L337-L338]` |
| User does not set `method`, URL is HTTP/HTTPS | `PUT` | `[libavformat/hlsenc.c:L339-L341]` |
| User does not set `method`, URL is `file://` | (no method dictionary entry — file protocol does not use HTTP semantics) | `[libavformat/hlsenc.c:L335-L341]` |

This default has historical roots: PUT is the only RFC 7231 method with a clean "upload-this-resource-to-this-URI" semantics, and most HLS origin endpoints implement PUT specifically for ingest. Some origin servers, however, require POST or a vendor-specific method; integrators on those endpoints set `-method POST` (or similar) before opening the muxer.

**Persistent connections.** At `[libavformat/hlsenc.c:L298-L308]`, `hlsenc_io_open` reuses an existing `URLContext` for new uploads when three conditions are simultaneously true: the AVIOContext is already open (`*pb != NULL`), the target URL is HTTP (`ff_is_http_proto(filename)` returns true), and the user has enabled `hls->http_persistent`. The reuse mechanism calls `ff_http_do_new_request` at `[libavformat/hlsenc.c:L304]`, which issues a new HTTP request through the existing TCP and TLS context. This dramatically reduces handshake latency on long-running live streams where the muxer would otherwise establish a new TLS session every segment. The complementary `hlsenc_io_close` at `[libavformat/hlsenc.c:L313-L331]` mirrors the logic: when `http_persistent` is set and the protocol is HTTP, the function calls `avio_flush` and `ffurl_shutdown` rather than closing the underlying socket.

**HTTP DELETE for sliding-window cleanup.** Live HLS streams that use a fixed window of segments (`hls_list_size`) require old segments to be removed from the origin once they fall out of the window. When `HLS_DELETE_SEGMENTS` is set in `hls->flags` (the `delete_segments` flag value declared at `[libavformat/hlsenc.c:L99]` and registered as an AV_OPT_TYPE_CONST at `[libavformat/hlsenc.c:L3147]`), and the output URL is HTTP, the muxer issues an HTTP DELETE per evicted segment. The DELETE is routed through a dedicated AVIOContext field, `HLSContext::http_delete`, declared at `[libavformat/hlsenc.c:L261]` and used inside the deletion helper at `[libavformat/hlsenc.c:L515-L517]`, which calls `av_dict_set(&opt, "method", "DELETE", 0)` followed by `hlsenc_io_open(avf, &hls->http_delete, path, &opt)`. CDNs that disallow DELETE on the ingest endpoint will return HTTP 405 (Method Not Allowed); the muxer logs the error and either proceeds (if `ignore_io_errors=1` at `[libavformat/hlsenc.c:L3178]`) or terminates.

**`hls_base_url` semantics.** The option `hls_base_url` at `[libavformat/hlsenc.c:L3129]` is a plain string that is prepended to every segment file entry written into the playlist. The prepend itself happens inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L194-L195]` (`if (baseurl) avio_printf(out, "%s", baseurl);` immediately before the segment filename is written). The option's purpose is to bridge a split-origin deployment where the playlist is served from one host (typically a control-plane domain) and segments from another (a CDN edge). Crucially, `hls_base_url` does *not* affect the URL the muxer uploads the playlist to, nor the URL it uploads segments to — it affects only the URL strings written into the playlist text that players see. Integrators sometimes assume it controls upload routing; it does not.

#### Worked Breakage Scenario

A CDN that rejects PUT and only accepts POST (a small but real subset of object-storage gateways behave this way) will return HTTP 405 for every segment upload from the HLS muxer's default configuration. Each 405 propagates through `hlsenc_io_open` as a non-zero return value, becomes `AVERROR(EIO)` (or another I/O error from the HTTP protocol handler), and surfaces to the application as a write-packet failure. The application either terminates the muxer or, if it has set `ignore_io_errors=1` at `[libavformat/hlsenc.c:L3178]`, swallows the error and continues — but every subsequent segment is also rejected. The fix is for the integrator to set `-method POST` before invoking the muxer (or to set it via the AVOption API on the muxer's `AVFormatContext->priv_data`). The lesson for documentation readers: the default HTTP method is a load-bearing default that is invisible until an incompatible origin is encountered.

---

### Consumer: FFmpeg Test Suite (`tests/` Tree)

#### Plain-Language Summary

The FFmpeg `tests/` tree contains FATE (FFmpeg Automated Testing Environment) reference tests that exercise the HLS muxer and demuxer end-to-end. These tests compare the muxer's output and the demuxer's output to recorded reference files byte for byte. Any change to muxer or demuxer emission shape — even a change that is invisible to a player — will invalidate the references and break the test suite.

**This section is a survey only.** The `tests/` directory is explicitly out of scope for this documentation effort per AAP §0.8.2.2. No test files are read, modified, or extended by this documentation. The purpose of this section is solely to make integrators and refactor-authors aware that the HLS pipeline has a byte-exact reference contract enforced by the project's own continuous-integration system. Anyone planning a refactor must coordinate with the test maintainer to update references in lockstep with code changes.

#### Technical Detail

The HLS pipeline's test surface is, by convention, located in:

| Path Pattern | Purpose | Notes |
|---|---|---|
| `tests/fate/hls.mak` | FATE test recipes that invoke the HLS muxer and demuxer | Drives `ffmpeg`/`ffprobe` with the relevant `-f hls` flags and option combinations; not read by this documentation effort |
| `tests/ref/fate/hls-*` | Recorded reference output | Contains expected M3U8 text and expected segment binaries; not read by this documentation effort |

Because the references are byte-exact, every byte the muxer emits is part of an implicit contract with FATE. The following classes of refactor will break the reference even if no integrator or player notices:

- Reordering EXT-X-* tag emission inside `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]` — even an emission order change that the HLS specification permits.
- Changing the format specifier of an emission line, for example from `%d` to `%.0f` for `#EXT-X-TARGETDURATION` at `[libavformat/hlsplaylist.c:L120]` — the output is numerically identical but textually different.
- Changing whitespace or newline placement in any `avio_printf` call inside `libavformat/hlsplaylist.c` (lines 32–206) or `libavformat/hlsenc.c`.
- Changing the line at which `#EXT-X-DISCONTINUITY` is emitted (currently before the affected segment's `#EXTINF`, see `[libavformat/hlsplaylist.c:L156-L158]`).

The MPEG-TS sub-muxer's bytestream output also feeds FATE references; any change to packet packing, PAT/PMT scheduling, or PCR cadence in `libavformat/mpegtsenc.c` will affect the HLS reference set transitively.

#### Worked Breakage Scenario

Suppose a refactor of `ff_hls_write_playlist_header` switched the `#EXT-X-TARGETDURATION` format specifier from `%d` to `%.0f` at `[libavformat/hlsplaylist.c:L120]`. For an integer target duration the output is textually identical (`6` becomes `6` in both cases), but for any code path where `lrint` had a non-zero remainder before the conversion to `int`, the format would diverge. Even in the all-integer-output case, downstream readers using `printf("%d")` to re-emit the value would be unaffected, but any FATE reference that compares the playlist text byte-for-byte would fail because the format-string change is itself part of the source that the build is gated on. This is one of several reasons the muxer's exact emission format is a *freeze contract* — see [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md) for the formal version of the contract.

---

### Consumer: In-Tree DASH Muxer (`libavformat/dashenc.c`)

#### Plain-Language Summary

FFmpeg's DASH muxer reuses the HLS pipeline's playlist tag-writer translation unit (`hlsplaylist.o`) to emit HLS-format manifests *alongside* its native DASH MPD manifest. This is the most surprising hidden dependency in the HLS pipeline: a refactor that changes the ABI or signatures of the eight `ff_hls_write_*` functions does not just change HLS muxer behavior — it changes DASH muxer behavior simultaneously, because both muxers link against the same compiled object file.

#### Technical Detail

**Build-level coupling.** The coupling is declared in one line of the build system at `[libavformat/Makefile:L189]`:

```text
OBJS-$(CONFIG_DASH_MUXER)                += dash.o dashenc.o hlsplaylist.o
```

That is: when `CONFIG_DASH_MUXER` is enabled at configure time, the DASH muxer's object set includes `hlsplaylist.o` alongside its own `dash.o` and `dashenc.o`. The corresponding HLS muxer object set at `[libavformat/Makefile:L277]` (`OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`) also includes `hlsplaylist.o`. Both muxers thus call into the same compiled-symbols table at link time. There is no separate "DASH copy" or "HLS copy" of the tag-writer code.

**Shared function surface.** The DASH muxer calls into the eight exports declared at `[libavformat/hlsplaylist.h:L38-L63]`:

| Function | Purpose | Header Line |
|---|---|---|
| `ff_hls_write_playlist_version` | Emits `#EXTM3U` and `#EXT-X-VERSION` | `[libavformat/hlsplaylist.h:L38]` |
| `ff_hls_write_audio_rendition` | Emits `#EXT-X-MEDIA:TYPE=AUDIO,...` for one audio rendition | `[libavformat/hlsplaylist.h:L39-L41]` |
| `ff_hls_write_subtitle_rendition` | Emits `#EXT-X-MEDIA:TYPE=SUBTITLES,...` for one subtitle rendition | `[libavformat/hlsplaylist.h:L42-L44]` |
| `ff_hls_write_stream_info` | Emits `#EXT-X-STREAM-INF:BANDWIDTH=...` for one variant | `[libavformat/hlsplaylist.h:L45-L49]` |
| `ff_hls_write_playlist_header` | Emits the playlist header block (version, allow-cache, target-duration, media-sequence, playlist-type, i-frames-only) | `[libavformat/hlsplaylist.h:L50-L52]` |
| `ff_hls_write_init_file` | Emits `#EXT-X-MAP:URI=...` for fMP4 initialization-segment delivery | `[libavformat/hlsplaylist.h:L53-L54]` |
| `ff_hls_write_file_entry` | Emits one segment entry (`#EXTINF`, optional `#EXT-X-BYTERANGE`, `#EXT-X-PROGRAM-DATE-TIME`, baseurl prefix, filename) | `[libavformat/hlsplaylist.h:L55-L62]` |
| `ff_hls_write_end_list` | Emits `#EXT-X-ENDLIST` (VOD final tag) | `[libavformat/hlsplaylist.h:L63]` |

Each function emits one or more specific `#EXT-X-*` tags as fixed-format `avio_printf` lines (the implementation is in `libavformat/hlsplaylist.c` lines 32 through 206). A signature change to any of these eight functions — adding a parameter, changing a parameter type, changing the return type, renaming, removing — would require synchronous updates to both `hlsenc.c` and `dashenc.c`. Adding a new tag emission inside one of these functions changes the HLS *and* the DASH-emitted HLS manifest in lockstep.

#### Worked Breakage Scenario

Suppose a refactor concluded that the `iframe_mode` parameter of `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.h:L50-L52]` was "HLS-specific" (because `#EXT-X-I-FRAMES-ONLY` is an HLS concept) and removed it. The change would compile cleanly inside `hlsenc.c` because the I-frame emission path is updated locally. But `dashenc.c`'s call site — which today passes 0 for that parameter, because DASH manifests do not use the I-frames-only tag — would now have a parameter-mismatch error, breaking the entire DASH muxer build.

The reverse direction is just as fragile: adding a new feature to `ff_hls_write_file_entry` "for HLS" (for example, a new conditional `#EXT-X-` tag) would unintentionally also enable that emission in DASH-emitted HLS manifests, which might confuse a DASH client that received a side-channel HLS manifest. Cross-format consumers of "HLS-style helpers" must therefore always be considered when refactoring `libavformat/hlsplaylist.c` or `libavformat/hlsplaylist.h`. The surface area of "what can change without breaking DASH" is smaller than the surface area of "what can change without breaking HLS players".

---

### Consumer: Generic `segment` Muxer (`libavformat/segment.c`)

#### Plain-Language Summary

FFmpeg ships a generic alternative segmenter at `[libavformat/segment.c]`, registered as `ff_segment_muxer` and `ff_stream_segment_muxer`. It is a separate format from the HLS muxer, with separate code, separate option table, and separate output conventions. It can emit an M3U8 playlist as a side-channel artifact, but the M3U8 it emits is not byte-compatible with the HLS muxer's output, and the segment files it produces are configured through a different mechanism. Integrators sometimes mistake one muxer for the other because both produce `.ts` files with an accompanying `.m3u8`; the existing user-facing documentation at `[doc/muxers.texi:§hls]` explicitly cross-references the alternative.

#### Technical Detail

**Registration.** Two muxers are registered:

| Registration | Source Line | Notes |
|---|---|---|
| `ff_segment_muxer` | `[libavformat/segment.c:L1107]` | Format name `"segment"`, flags `AVFMT_NOFILE \| AVFMT_GLOBALHEADER` at `[libavformat/segment.c:L1110]` |
| `ff_stream_segment_muxer` | `[libavformat/segment.c:L1123]` | Format names `"stream_segment,ssegment"`, flags `AVFMT_NOFILE` at `[libavformat/segment.c:L1126]` |

Both share the same callback set (`seg_init`, `seg_write_header`, `seg_write_packet`, `seg_write_trailer`, `seg_free`, `seg_check_bitstream`) declared at `[libavformat/segment.c:L1113-L1118]` and `[libavformat/segment.c:L1129-L1134]`. They are gated by separate `CONFIG_SEGMENT_MUXER` and `CONFIG_STREAM_SEGMENT_MUXER` defines (visible in the surrounding `#if` blocks at `[libavformat/segment.c:L1106]` and `[libavformat/segment.c:L1122]`), so it is possible to build one without the other.

**Cross-reference in the user documentation.** The user-facing Texinfo documentation at `[doc/muxers.texi:L1909-L1911]` reads:

> See also the @ref{segment} muxer, which provides a more generic and flexible implementation of a segmenter, and can be used to perform HLS segmentation.

This is the canonical pointer from HLS-muxer users to the alternative. It is the source of much integrator confusion: the recommendation reads as if the two muxers are interchangeable, but they are not.

**Why the two are not interchangeable.** The `segment` muxer does not link against `hlsplaylist.o` — the Makefile at `[libavformat/Makefile]` declares `OBJS-$(CONFIG_SEGMENT_MUXER) += segment.o` without any HLS dependency. It therefore does *not* call into `ff_hls_write_playlist_version`, `ff_hls_write_playlist_header`, or any of the other shared writers. Instead, it constructs its M3U8 (when configured to do so via `-segment_list_type m3u8`) through a separate code path inside `segment.c` that has its own emission order, its own handling of `EXT-X-VERSION`, and its own handling of `EXT-X-TARGETDURATION`. The output is *valid* M3U8 (and is accepted by many players), but it is not byte-compatible with what `ff_hls_muxer` emits, and it does not implement the full HLS feature set (encryption modes, variant streams, fMP4 segments, sample encryption, program-date-time injection, periodic rekeying, et cetera).

#### Worked Breakage Scenario

Suppose an integrator notices a feature gap in `-f hls` — for example, they need a custom segment-naming pattern that `hls_segment_filename` does not support — and decides to switch to `-f segment -segment_format hls -segment_list_type m3u8` to gain access to the more flexible templating in the `segment` muxer. They re-run their pipeline, the new M3U8 looks superficially similar, and the test passes locally. In production, however, their player begins rejecting the manifest with a vendor-specific error code, because (for example) the `segment` muxer's M3U8 lacks `#EXT-X-PLAYLIST-TYPE:EVENT` or has a different ordering of leading metadata tags.

The integrator's diagnostic loop is slow because the new manifest *is* valid M3U8 — it just happens to be a slightly different valid M3U8 from the one the player was tested against. The lesson: `-f hls` and `-f segment -segment_format ... -segment_list_type m3u8` are *not* drop-in replacements for each other. Any switch between them is a deployment-grade change that requires player-side regression testing. The user-facing documentation at `[doc/muxers.texi:L1909-L1911]` does not warn about this incompatibility; this document does so explicitly.

---

### Breakage Analysis: If `hlsenc.c` Stopped Producing Valid M3U8

#### Plain-Language Summary

The HLS muxer is a *load-bearing* component for any FFmpeg consumer that publishes HLS. There is no in-tree fallback for `-f hls`: the `segment` muxer is the closest alternative but produces different M3U8 conventions and does not implement the HLS feature set; the DASH muxer publishes DASH MPD (with HLS-via-`hlsplaylist` only as a side channel), not the same shape as the HLS muxer's primary output. A hypothetical removal or breakage of `hlsenc.c` would silently degrade the HLS publishing capability of every downstream consumer that has not built in its own fallback logic.

#### Technical Detail

**No fallback in `ff_hls_muxer`.** The format-registration definition at `[libavformat/hlsenc.c:L3191-L3207]` declares the muxer's identity, codec defaults, format flags, private class, and lifecycle callback slots. The field assignments are summarized in the table below; the registration itself contains no alternative or deprecation pointer.

| Field | Value | Notes |
|---|---|---|
| `.p.name` | `"hls"` | The name resolved by `av_guess_format("hls", ...)` |
| `.p.long_name` | `"Apple HTTP Live Streaming"` (via `NULL_IF_CONFIG_SMALL`) | Used for diagnostic messages |
| `.p.extensions` | `"m3u8"` | The extension `-f` autodetect maps to this muxer |
| `.p.audio_codec` / `.p.video_codec` / `.p.subtitle_codec` | `AV_CODEC_ID_AAC` / `AV_CODEC_ID_H264` / `AV_CODEC_ID_WEBVTT` | Default codecs when the caller does not specify |
| `.p.flags` | `AVFMT_NOFILE \| AVFMT_GLOBALHEADER \| AVFMT_NODIMENSIONS` | See §"A Note on Format-Registration Flags" above |
| `.p.priv_class` | `&hls_class` | The `AVClass` exposing the AVOption table |
| `.priv_data_size` | `sizeof(HLSContext)` | Storage allocated by libavformat for the muxer's private state |
| `.init`/`.write_header`/`.write_packet`/`.write_trailer`/`.deinit` | `hls_init` / `hls_write_header` / `hls_write_packet` / `hls_write_trailer` / `hls_deinit` | The five lifecycle callbacks |

Callers that resolve the muxer by name `"hls"` via `av_guess_format` either receive a pointer to this `FFOutputFormat` or `NULL` — there is no graceful degradation path inside libavformat. Applications must implement their own fallback logic if they want one. `[inferred — no direct source]`

**`allformats.c` registration.** The two HLS format symbols are declared at `[libavformat/allformats.c:L216-L217]`:

```c
extern const FFInputFormat  ff_hls_demuxer;
extern const FFOutputFormat ff_hls_muxer;
```

These declarations are gathered by the `allformats.c` registration mechanism into the global format table at libavformat initialization time. Removing either declaration (or removing the corresponding object file from the link line) makes the format invisible to `av_guess_format("hls", ...)` and to the command-line `-f hls` flag dispatch.

**Build-system gating.** The HLS muxer's object set is conditional on `CONFIG_HLS_MUXER` at `[libavformat/Makefile:L277]`:

```text
OBJS-$(CONFIG_HLS_MUXER)                 += hlsenc.o hlsplaylist.o
```

Disabling `CONFIG_HLS_MUXER` at configure time would omit both `hlsenc.o` and `hlsplaylist.o` from the build. This has a second-order effect on the DASH muxer: `dashenc.c` *expects* `hlsplaylist.o` to be present in its object set (see `[libavformat/Makefile:L189]`), but only because it independently lists `hlsplaylist.o` in its own line. Disabling `CONFIG_HLS_MUXER` does *not* prevent the DASH muxer from also including `hlsplaylist.o` — the two lines accumulate into the same object set independently, so DASH continues to link `hlsplaylist.o` even when HLS is disabled. This is a build-system design choice that keeps the DASH muxer independent of whether HLS is configured.

#### Worked Breakage Scenario

Suppose `hlsenc.c` is removed from the build via `--disable-muxer=hls` at FFmpeg's configure stage. The effect cascades:

1. **`libavformat/Makefile`** at `[libavformat/Makefile:L277]` no longer adds `hlsenc.o` to the object set, because `CONFIG_HLS_MUXER` is unset.
2. **`libavformat/allformats.c`** at `[libavformat/allformats.c:L217]` still *declares* `extern const FFOutputFormat ff_hls_muxer`, but the symbol is no longer defined anywhere because `hlsenc.c` is not compiled. Depending on how the auto-generated `format_list.c` (built during `configure`) handles unconfigured formats, the declaration is either ifdef'd out at the registration call site or resolved as a linker error. Either way, `av_guess_format("hls", NULL, NULL)` at runtime returns `NULL`. `[inferred — no direct source]`
3. **Downstream applications** that did `av_guess_format("hls", NULL, NULL)` and relied on a non-`NULL` return now fail. The application either prints a "muxer not found" error and exits, or falls back to a different format (commonly `-f segment -segment_format mpegts -segment_list_type m3u8` from `libavformat/segment.c` at `[libavformat/segment.c:L1107]`), accepting that the output M3U8 will not be byte-identical to what users expect.
4. **The DASH muxer** continues to link `hlsplaylist.o` because its Makefile entry at `[libavformat/Makefile:L189]` lists it independently of the HLS-muxer configuration. The DASH muxer's HLS-side-channel output continues to function.
5. **The FATE test suite** under `tests/` flags every HLS-related test as skipped (because the muxer is unavailable). This is not a false failure, but it does mean the project's continuous integration loses coverage for the HLS publishing path.

The lesson: removing `hlsenc.c` is a load-bearing decision with no in-tree fallback. Integrators planning to disable the HLS muxer for footprint reasons (embedded builds, security-sensitive deployments) should plan for their consumers to fall back to a different format, *not* expect `-f hls` to silently produce equivalent output through another code path. There is no equivalent code path.

---

## Summary Table: Consumer-Side Contracts at a Glance

| Consumer | What They Read | Contract Source | Failure Mode If Violated |
|---|---|---|---|
| HLS-compliant media players | M3U8 playlist + segment files | RFC 8216; emission shape at `[libavformat/hlsplaylist.c:L32-L206]` | Manifest rejected, stream stalls, or wrong content played |
| CDN ingest endpoints (HTTP origins) | HTTP PUT/DELETE requests with playlist and segment bodies | Method default at `[libavformat/hlsenc.c:L337-L341]`; persistence at `[libavformat/hlsenc.c:L298-L308]`; DELETE at `[libavformat/hlsenc.c:L515-L517]` | Uploads rejected with HTTP 405 / 4xx; muxer fails unless `ignore_io_errors=1` |
| FFmpeg FATE test suite | Byte-exact M3U8 and segment binaries | Implicit — every `avio_printf` in `libavformat/hlsplaylist.c` and `libavformat/hlsenc.c` | CI failure on any emission-shape change |
| In-tree DASH muxer | The 8 `ff_hls_write_*` exports | `[libavformat/hlsplaylist.h:L38-L63]`; build linkage at `[libavformat/Makefile:L189]` | DASH muxer build break or runtime behavior change |
| Generic `segment` muxer (alternative) | Independent — does not consume HLS muxer output | Separate registrations at `[libavformat/segment.c:L1107]` and `[libavformat/segment.c:L1123]` | N/A (independent code path); user-doc cross-reference at `[doc/muxers.texi:L1909-L1911]` |
| External automation (orchestrators, monitors) | M3U8 line-by-line text | Implicit — every `avio_printf` in `libavformat/hlsplaylist.c` | Parser break; missing or duplicated segments in downstream telemetry |

For the formal pinned versions of each contract listed above, see [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md) (HTTP, AES-128, BANDWIDTH, EXT-X-KEY, EXT-X-MEDIA, EXT-X-PROGRAM-DATE-TIME) and [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md) (M3U8 header order, version negotiation rules, segment naming, discontinuity placement).
