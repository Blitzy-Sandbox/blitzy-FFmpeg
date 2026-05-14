# Inputs and Outputs — Complete I/O Reference for the HLS Pipeline

This document is the exhaustive I/O catalog for the FFmpeg HLS muxer and demuxer: every `AVPacket` field consumed, every `AVOption` recognized, every `#EXT-X-*` line emitted, every M3U8 line type parsed. All source references in this document are anchored to commit `566ad786`. See [`../README.md`](../README.md) for the canonical commit-anchor banner, citation format, and reading order.

---

## Overview

### What the HLS Muxer Consumes and Produces

The HLS muxer is a *meta-muxer*: it consumes `AVPacket` objects through `av_write_frame` / `av_interleaved_write_frame`, reads dozens of configuration values via the `AVOption` framework, and produces two kinds of output. The first kind is **media segments** — either MPEG-TS files or fragmented MP4 files, written by an internal sub-muxer for each variant stream. The second kind is **M3U8 playlist files** — a parent master playlist (when configured), one media playlist per variant, optional subtitle playlists, and (in fMP4 mode) an initialization segment referenced by `#EXT-X-MAP`. The demuxer is the reverse: it consumes one M3U8 file (plus the segment URLs it cites) and produces a stream of `AVPacket` objects.

### What This Document Catalogs

This document enumerates every distinct input and output, individually, without grouping or summarizing. Every `AVOption` has its own table row. Every `AV_OPT_TYPE_CONST` enum-value alias has its own row. Every `#EXT-X-*` tag emission has its own row identifying the writer function, the format string, the conditional emission rule, and the exact source line. Every M3U8 line type the demuxer parser recognizes has its own row. The result is a flat, table-driven reference suitable for porting checklists, conformance test plans, and integrator option discovery.

### Reading the Tables

- **"Default"** columns show the literal value from the source declaration; an empty entry means the option has no compile-time default (the field is zero-initialized through `priv_data_size` allocation).
- **"Bounds"** columns show the `min..max` range declared in the `AVOption` record. `INT_MIN..INT_MAX`, `INT64_MIN..INT64_MAX`, and `0..UINT_MAX` are written in those exact forms; sentinel-coded ranges (e.g., `-1..1` for "auto/no/yes") are preserved verbatim.
- **"Type"** columns show the `AV_OPT_TYPE_*` token verbatim. `AV_OPT_TYPE_DURATION` values are in microseconds; `AV_OPT_TYPE_FLAGS` is a bit-mask whose enumerated bits are listed in §5.2.
- **"Effect"** columns provide a one-sentence plain-language explanation. For exhaustive semantics, see [`../technical/codec-logic.md`](../technical/codec-logic.md) (muxer decision logic) and [`../technical/data-model.md`](../technical/data-model.md) (field-level dictionary).
- **"Source"** columns cite the exact line range at commit `566ad786` using the form `[<path>:L<start>-L<end>]`.

### Cross-Document References

| For | See |
|-----|-----|
| Component-level descriptions of segment generation, playlist construction, encryption | [`./functional-inventory.md`](./functional-inventory.md) |
| Failure scenarios triggered by malformed inputs or I/O errors | [`./exception-handling.md`](./exception-handling.md) |
| Decision tables for every branch that consumes these inputs | [`../technical/codec-logic.md`](../technical/codec-logic.md) |
| Field-level struct dictionary for every `OFFSET(...)` referenced in §5.1 | [`../technical/data-model.md`](../technical/data-model.md) |
| Formal type/bounds/contract pinning | [`../api-contracts/data-contracts.md`](../api-contracts/data-contracts.md) |
| Invariants that must hold for every emission listed in §7 | [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md) |

---

## Muxer Inputs: AVPacket Fields Consumed by `hls_write_packet`

### Plain-Language Summary

When the application calls `av_write_frame` or `av_interleaved_write_frame` to write a media packet, FFmpeg dispatches the packet to `hls_write_packet` at `[libavformat/hlsenc.c:L2410]`. This function reads specific `AVPacket` fields to (a) route the packet to the correct *variant stream*, (b) decide whether the packet is eligible to start a new segment, (c) accumulate segment-duration statistics, (d) track byte positions for byterange mode, and finally (e) forward the packet to the underlying MPEG-TS or fMP4 sub-muxer. The packet payload itself is never inspected by the HLS layer — only the metadata fields below are read.

### Technical Detail

The function signature is `static int hls_write_packet(AVFormatContext *s, AVPacket *pkt)` at `[libavformat/hlsenc.c:L2410]`. Every field read from `pkt` in the body of this function is enumerated below.

| AVPacket Field | Used For | Source |
|---|---|---|
| `pkt->stream_index` | Index into `s->streams[]` to obtain the matching `AVStream` (`AVStream *st = s->streams[pkt->stream_index]` at L2414); also used to walk `vs->streams[]` to identify which variant stream owns the packet, and to check `pkt->stream_index == vs->reference_stream_index` to decide whether this packet drives the segment-cut clock (`is_ref_pkt` at L2476). | `[libavformat/hlsenc.c:L2414, L2425-L2446, L2476]` |
| `pkt->pts` | Initializes `vs->start_pts` on the first packet (L2463-L2464); on audio-first start with a later video packet, lowers `vs->start_pts` to the video pts (L2468-L2470); seeds `vs->end_pts` on the first reference packet (L2482-L2483); accumulates segment duration via `(pkt->pts - vs->end_pts) * st->time_base.num / st->time_base.den` (L2486-L2487, L2495); drives the segment-cut comparison `av_compare_ts(pkt->pts - vs->start_pts, ..., end_pts, AV_TIME_BASE_Q) >= 0` (L2501-L2502); recomputes `cur_duration` at segment finalization (L2617). | `[libavformat/hlsenc.c:L2463-L2502, L2617, L2619]` |
| `pkt->pts == AV_NOPTS_VALUE` | Sentinel check that disables both `can_split` and `is_ref_pkt` so a packet with no timestamp can never trigger a segment cut (L2478-L2479). | `[libavformat/hlsenc.c:L2478-L2479]` |
| `pkt->flags & AV_PKT_FLAG_KEY` | Combined with `(hls->flags & HLS_SPLIT_BY_TIME)` to set `can_split` (L2473-L2475): only key-frame video packets are eligible to cut a segment unless the user has explicitly requested time-based splitting. Also used at packet forwarding to detect the keyframe boundary for byterange tracking (L2681-L2682). | `[libavformat/hlsenc.c:L2473-L2475, L2681-L2682]` |
| `pkt->duration` | Drives the per-packet duration accumulator: on the first reference packet `vs->dpp = (double)(pkt->duration) * st->time_base.num / st->time_base.den` (L2488); subsequent packets add to `vs->duration` (L2491). When `pkt->duration` is zero, a warning is logged and the duration is approximated from the pts delta (L2492-L2496). | `[libavformat/hlsenc.c:L2488, L2490-L2496]` |
| `pkt->size` | Added to `vs->video_keyframe_size` on every successful packet forward (L2680); contributes (via `avio_tell` deltas) to the segment-size byterange tracking. | `[libavformat/hlsenc.c:L2680]` |
| `pkt->data` | Forwarded to the sub-muxer via `ff_write_chained(oc, stream_index, pkt, s, 0)` at L2679. The HLS layer never reads or modifies `pkt->data`. | `[libavformat/hlsenc.c:L2679]` |
| `pkt->dts` | Forwarded to the sub-muxer through `ff_write_chained`; not directly inspected by `hls_write_packet`, but the `AV_NOPTS_VALUE` check on `pkt->pts` at L2478 protects against decoder-side timing anomalies. | `[libavformat/hlsenc.c:L2679]` |

Additional contextual reads that are NOT direct `pkt->*` field accesses but are inferred from packet routing: the function also reads `st->codecpar->codec_type` (to branch on `AVMEDIA_TYPE_VIDEO` / `AVMEDIA_TYPE_AUDIO` / `AVMEDIA_TYPE_SUBTITLE` at L2429-L2436, L2465, L2474, L2476, L2681) and `st->time_base` (for the duration scaling at L2487, L2488, L2491, L2495, L2617). These belong to the `AVStream` rather than the packet but are enumerated here because the routing decision uses them in conjunction with `pkt->stream_index`.


## Muxer Inputs: AVFormatContext Fields Consumed

### Plain-Language Summary

Beyond per-packet metadata, the HLS muxer reads context-level information from the `AVFormatContext` (the `s` parameter passed to every callback): the output URL, the array of input streams and their codec parameters, the user-provided AVDictionary of options, and a handful of behavior-control flags. These reads happen primarily during `hls_init` (for stream classification and variant allocation), during `hls_write_header` (for codec parameter copy-out to the sub-muxer), inside `hls_mux_init` (when creating each variant's child `AVFormatContext`), and inside the per-packet path through `hls_write_packet`.

### Technical Detail

| AVFormatContext Field | Used For | Source |
|---|---|---|
| `s->priv_data` | Cast to `HLSContext *hls` at the top of every callback (e.g., `HLSContext *hls = s->priv_data;` at L2412). The `priv_data` allocation is sized by `FFOutputFormat.priv_data_size = sizeof(HLSContext)` at L3196. | `[libavformat/hlsenc.c:L2412, L3196]` |
| `s->streams[]` | Iterated in `hls_init` for stream classification (video/audio/subtitle) and codec-attribute extraction. In `hls_write_packet`, `s->streams[pkt->stream_index]` resolves the `AVStream` for the incoming packet (L2414). | `[libavformat/hlsenc.c:L2414]` |
| `s->nb_streams` | Loop bound when scanning all streams during initialization and during master-playlist generation. | `[libavformat/hlsenc.c]` (referenced throughout `hls_init` and `hls_write_header`) |
| `s->streams[i]->codecpar->codec_type` | Branched on `AVMEDIA_TYPE_VIDEO`, `AVMEDIA_TYPE_AUDIO`, `AVMEDIA_TYPE_SUBTITLE` at L2429, L2433, L2465, L2474, L2476, L2681 to control routing (subtitle packets go to `vs->vtt_avf`, others to `vs->avf`) and reference-packet classification (`is_ref_pkt = ... codec_type == AVMEDIA_TYPE_VIDEO ...`). | `[libavformat/hlsenc.c:L2429, L2433, L2465, L2474, L2476, L2681]` |
| `s->streams[i]->codecpar->codec_id` | Read by `write_codec_attr` at `[libavformat/hlsenc.c:L352+]` to compute the RFC 6381 `CODECS="..."` string emitted in `#EXT-X-STREAM-INF` and the per-rendition `codec_attr` recorded on `VariantStream`. | `[libavformat/hlsenc.c:L352+]` |
| `s->streams[i]->codecpar->extradata` / `extradata_size` | Copied to the sub-muxer's `codecpar` when each variant's child `AVFormatContext` is created in `hls_mux_init`. The HLS muxer registers with `AVFMT_GLOBALHEADER` (`[libavformat/hlsenc.c:L3199]`), forcing upstream encoders to deliver complete extradata. | `[libavformat/hlsenc.c:L3199]` |
| `s->streams[i]->codecpar->width` / `height` | Read by `write_codec_attr` to populate `RESOLUTION=%dx%d` in `#EXT-X-STREAM-INF`. | `[libavformat/hlsenc.c:L352+, L3199]` |
| `s->streams[i]->codecpar->ch_layout.nb_channels` | Read for `#EXT-X-MEDIA:TYPE=AUDIO,CHANNELS="N"` emission via `ff_hls_write_audio_rendition`. | `[libavformat/hlsplaylist.c:L40-L56]` |
| `s->streams[i]->time_base` | Used for timestamp scaling in segment-cut duration computation at L2487, L2488, L2491, L2495, L2617. | `[libavformat/hlsenc.c:L2487-L2617]` |
| `s->streams[i]->avg_frame_rate` | Read by `write_codec_attr` to emit `FRAME-RATE=...` in `#EXT-X-STREAM-INF`. | `[libavformat/hlsplaylist.c:L78-L107]` |
| `s->url` | The application-supplied output URL. Parsed by `av_dirname`/`av_basename` to derive the per-variant playlist filenames; consumed by `avio_find_protocol_name` at L2545, L607 to detect `file` vs `http(s)`; checked by `ff_is_http_proto` at L296, L316, L335 to drive HTTP-persistent-connection behavior. | `[libavformat/hlsenc.c:L296, L316, L335, L607, L2545]` |
| `s->oformat` | The `FFOutputFormat *` registration record (`ff_hls_muxer` at L3191-L3207). Cited indirectly: callbacks dispatch through `s->oformat->...` slots set at L3201-L3206. | `[libavformat/hlsenc.c:L3191-L3207]` |
| `s->io_open` | Function pointer wrapped by `hlsenc_io_open` at L292-L311 — when no persistent HTTP connection exists, `hlsenc_io_open` falls back to `s->io_open(s, pb, filename, AVIO_FLAG_WRITE, options)` at L299. Also called directly by `hls_encryption_start` at L723 (to read the key-info file) and L753 (to read the raw key file). | `[libavformat/hlsenc.c:L299, L723, L753]` |
| `s->io_close2` (implicit via `ff_format_io_close`) | Used inside `hlsenc_io_close` at L321 when persistent HTTP is disabled. | `[libavformat/hlsenc.c:L321]` |
| `s->flags` | Read at init; `AVFMT_FLAG_*` flags propagate to the child sub-muxer (`oc->flags = s->flags` is set in `hls_mux_init`). | `[libavformat/hlsenc.c:L773+]` |
| `s->avoid_negative_ts` | Propagated to the child sub-muxer in `hls_mux_init`. | `[libavformat/hlsenc.c:L773+]` |
| `s->interrupt_callback` | Copied to `oc->interrupt_callback` in `hls_mux_init` (the child mux inherits interrupt handling). | `[libavformat/hlsenc.c:L773+]` |
| `s->max_delay` | Copied to the child sub-muxer's `max_delay` in `hls_mux_init` so child muxers buffer for the same delay budget. | `[libavformat/hlsenc.c:L773+]` |
| `s->opaque` | Copied to the child sub-muxer's `opaque` in `hls_mux_init`. | `[libavformat/hlsenc.c:L773+]` |
| `s->strict_std_compliance` | Copied to the child sub-muxer to keep strict-mode behavior consistent end-to-end. | `[libavformat/hlsenc.c:L773+]` |
| `s->metadata` | Forwarded to the child sub-muxer; permits the application's container-level metadata to flow into the segment files. | `[libavformat/hlsenc.c:L773+]` |
| `s->pb` (initialization side-effects only) | The HLS muxer registers with `AVFMT_NOFILE` at L3199, so the framework does NOT open `s->pb` for the application. Output I/O is opened lazily per variant via `hlsenc_io_open`. | `[libavformat/hlsenc.c:L3199]` |



## Muxer Inputs: AVOptions (Complete Enumeration)

### Plain-Language Summary

The HLS muxer exposes **35 distinct user-tunable AVOptions** plus **23 `AV_OPT_TYPE_CONST` enum-value aliases** through the standard FFmpeg `AVOption` framework. Users set these on the command line as `-option_name value`, in the `AVFormatContext` private-data structure, or via `av_opt_set` API calls. The const aliases let users pass symbolic names like `mpegts` / `fmp4` or `vod` / `event` instead of opaque integer codes. The full table at `[libavformat/hlsenc.c:L3121-L3181]` contains 58 entries total (35 + 23) plus a NULL terminator. Each entry is enumerated individually below — no grouping.

### Source Anchor

The complete option array is declared as `static const AVOption options[] = { /* 58 entries */, {NULL} };` at `[libavformat/hlsenc.c:L3121-L3181]`. The `OFFSET(x)` macro at L3119 expands to `offsetof(HLSContext, x)` and locates each option's storage inside `HLSContext`. The `E` macro at L3120 expands to `AV_OPT_FLAG_ENCODING_PARAM` — every option carries this flag, marking them all as muxer-side (write-direction) settings.

### §5.1 — User-Tunable AVOptions (35 Entries)

Each row corresponds to one distinct option in source order. The "OFFSET" column shows the `HLSContext` field name to which the option binds — see [`../technical/data-model.md`](../technical/data-model.md) for the field-level dictionary of `HLSContext`.

| # | Option Name | Type | OFFSET | Default | Bounds | Effect | Source |
|---|---|---|---|---|---|---|---|
| 1 | `start_number` | `AV_OPT_TYPE_INT64` | `start_sequence` | `0` | `0..INT64_MAX` | Sets the first number in the `#EXT-X-MEDIA-SEQUENCE` series. Overridden by `hls_start_number_source` when that option is not `generic`. | `[libavformat/hlsenc.c:L3122]` |
| 2 | `hls_time` | `AV_OPT_TYPE_DURATION` | `time` | `2000000` µs (2 sec) | `0..INT64_MAX` | Target segment length in microseconds. Drives the segment-cut comparison in `hls_write_packet`; segments may be longer than this because the cut only happens at a keyframe boundary. | `[libavformat/hlsenc.c:L3123]` |
| 3 | `hls_init_time` | `AV_OPT_TYPE_DURATION` | `init_time` | `0` | `0..INT64_MAX` | Initial segment length in microseconds used for the first `hls_list_size` segments; falls back to `hls_time` thereafter. `0` means "use `hls_time` from the start." | `[libavformat/hlsenc.c:L3124]` |
| 4 | `hls_list_size` | `AV_OPT_TYPE_INT` | `max_nb_segments` | `5` | `0..INT_MAX` | Maximum number of segment entries kept in the media playlist. `0` means unlimited (typical for VOD). When the live sliding-window is active, older segments are evicted from the playlist when this count is exceeded. | `[libavformat/hlsenc.c:L3125]` |
| 5 | `hls_delete_threshold` | `AV_OPT_TYPE_INT` | `hls_delete_threshold` | `1` | `1..INT_MAX` | Minimum number of *unreferenced* segments to retain on disk before `HLS_DELETE_SEGMENTS` actually deletes them. Acts as a grace buffer so clients in-flight on a recent playlist can still fetch the just-evicted segment. Minimum value is 1 (NOT 0). | `[libavformat/hlsenc.c:L3126]` |
| 6 | `hls_vtt_options` | `AV_OPT_TYPE_STRING` | `vtt_format_options_str` | `NULL` | n/a | Options string forwarded to the WebVTT subtitle sub-muxer when subtitle streams are present. | `[libavformat/hlsenc.c:L3127]` |
| 7 | `hls_allow_cache` | `AV_OPT_TYPE_INT` | `allowcache` | `-1` (unset) | `INT_MIN..INT_MAX` | Sets the `#EXT-X-ALLOW-CACHE:YES|NO` tag for cache-control hints to HLS clients. `-1` means do NOT emit the tag at all. `0` emits `NO`; `1` emits `YES`. | `[libavformat/hlsenc.c:L3128]` |
| 8 | `hls_base_url` | `AV_OPT_TYPE_STRING` | `baseurl` | `NULL` | n/a | URL prefix prepended to every segment URI emitted in the playlist (e.g., `https://cdn.example.com/`). Useful when playlist and segments are served from different origins. | `[libavformat/hlsenc.c:L3129]` |
| 9 | `hls_segment_filename` | `AV_OPT_TYPE_STRING` | `segment_filename` | `NULL` | n/a | Filename template for segment files. Supports `%d` (sequence), `%v` (variant index or name), `%t` (timestamp), and `strftime` substitution when `strftime` is enabled. When NULL, segment filenames are derived from the playlist filename plus `POSTFIX_PATTERN "_%d"`. | `[libavformat/hlsenc.c:L3130]` |
| 10 | `hls_segment_options` | `AV_OPT_TYPE_DICT` | `format_options` | `NULL` | n/a | Dictionary of format options forwarded to each segment's sub-muxer (MPEG-TS or fMP4). Useful for tuning sub-muxer behavior without affecting HLS-level logic. | `[libavformat/hlsenc.c:L3131]` |
| 11 | `hls_segment_size` | `AV_OPT_TYPE_INT` | `max_seg_size` | `0` | `0..INT_MAX` | Maximum size of each segment file in bytes. When non-zero, activates byterange mode (which forces `#EXT-X-VERSION:4` and emits `#EXT-X-BYTERANGE:size@offset` per segment). `0` disables this constraint. | `[libavformat/hlsenc.c:L3132]` |
| 12 | `hls_key_info_file` | `AV_OPT_TYPE_STRING` | `key_info_file` | `NULL` | n/a | Path to a 2-or-3-line text file specifying the AES-128 encryption key URI, the local key file path, and an optional 32-hex-character IV. When set, every segment is AES-128 encrypted and `#EXT-X-KEY:METHOD=AES-128,URI=...` is emitted. See §13 for the file format. | `[libavformat/hlsenc.c:L3133]` |
| 13 | `hls_enc` | `AV_OPT_TYPE_BOOL` | `encrypt` | `0` | `0..1` | Boolean shortcut to enable AES-128 encryption without supplying a key info file: the muxer generates a random key. Combine with `hls_enc_key_url` to publish the key URI in the playlist. | `[libavformat/hlsenc.c:L3134]` |
| 14 | `hls_enc_key` | `AV_OPT_TYPE_STRING` | `key` | (unset) | n/a | Hex-coded 16-byte key (32 hex characters) used by `hls_enc=1`. Overrides the auto-generated random key. The declaration omits an explicit default, so the field is zero-initialized. | `[libavformat/hlsenc.c:L3135]` |
| 15 | `hls_enc_key_url` | `AV_OPT_TYPE_STRING` | `key_url` | `NULL` | n/a | URL emitted in `#EXT-X-KEY:URI=...` so clients know where to fetch the key. Required when using `hls_enc=1` (otherwise the URI defaults to a constructed value). | `[libavformat/hlsenc.c:L3136]` |
| 16 | `hls_enc_iv` | `AV_OPT_TYPE_STRING` | `iv` | (unset) | n/a | Hex-coded 16-byte initialization vector (32 hex characters). When omitted, the muxer derives the IV from the current segment sequence number. The declaration omits an explicit default. | `[libavformat/hlsenc.c:L3137]` |
| 17 | `hls_subtitle_path` | `AV_OPT_TYPE_STRING` | `subtitle_filename` | `NULL` | n/a | Filesystem path prefix for HLS subtitle playlists (WebVTT). When set, subtitle segment files and playlists are written under this directory. | `[libavformat/hlsenc.c:L3138]` |
| 18 | `hls_segment_type` | `AV_OPT_TYPE_INT` | `segment_type` | `SEGMENT_TYPE_MPEGTS` (0) | `0..SEGMENT_TYPE_FMP4` (1) | Selects segment container: `mpegts` (default) or `fmp4`. fMP4 mode forces `#EXT-X-VERSION:7`, emits `#EXT-X-MAP` referencing the init file, and uses `.m4s` segment extensions. Bound to `.unit = "segment_type"` for the const aliases listed in §5.2. | `[libavformat/hlsenc.c:L3139]` |
| 19 | `hls_fmp4_init_filename` | `AV_OPT_TYPE_STRING` | `fmp4_init_filename` | `"init.mp4"` | n/a | Filename for the fMP4 initialization segment when `hls_segment_type=fmp4`. Referenced from the playlist via `#EXT-X-MAP:URI="..."`. | `[libavformat/hlsenc.c:L3142]` |
| 20 | `hls_fmp4_init_resend` | `AV_OPT_TYPE_BOOL` | `resend_init_file` | `0` | `0..1` | When set, the init.mp4 is re-uploaded on every playlist refresh (useful for upload-style live distribution where intermediaries may purge stale init files). | `[libavformat/hlsenc.c:L3143]` |
| 21 | `hls_flags` | `AV_OPT_TYPE_FLAGS` | `flags` | `0` | `0..UINT_MAX` | Bit-mask of feature flags. Bound to `.unit = "flags"` for the 15 const aliases listed in §5.2 (single_file through iframes_only). Multiple flags are OR'd together. | `[libavformat/hlsenc.c:L3144]` |
| 22 | `strftime` | `AV_OPT_TYPE_BOOL` | `use_localtime` | `0` | `0..1` | When set, `hls_segment_filename` is expanded via `strftime` against the local time at each segment creation. Enables time-coded filenames like `seg-%Y%m%d-%H%M%S.ts`. | `[libavformat/hlsenc.c:L3160]` |
| 23 | `strftime_mkdir` | `AV_OPT_TYPE_BOOL` | `use_localtime_mkdir` | `0` | `0..1` | When set, creates the last directory component of the strftime-expanded path if it does not exist. Requires `strftime=1`. | `[libavformat/hlsenc.c:L3161]` |
| 24 | `hls_playlist_type` | `AV_OPT_TYPE_INT` | `pl_type` | `PLAYLIST_TYPE_NONE` (0) | `0..PLAYLIST_TYPE_NB-1` | Sets the `#EXT-X-PLAYLIST-TYPE` tag: `NONE` (omit the tag), `EVENT` (append-only live), or `VOD` (complete, finite). Bound to `.unit = "pl_type"` for the const aliases listed in §5.2. | `[libavformat/hlsenc.c:L3162]` |
| 25 | `method` | `AV_OPT_TYPE_STRING` | `method` | `NULL` | n/a | HTTP method override for playlist and segment uploads. When NULL and the protocol is http(s), defaults to `PUT` (see `set_http_options` at L337-L341). | `[libavformat/hlsenc.c:L3165]` |
| 26 | `hls_start_number_source` | `AV_OPT_TYPE_INT` | `start_sequence_source_type` | `HLS_START_SEQUENCE_AS_START_NUMBER` (0) | `0..HLS_START_SEQUENCE_LAST-1` | Selects the source for the initial `#EXT-X-MEDIA-SEQUENCE` value: `generic` (the `start_number` option), `epoch` (seconds since Unix epoch), `epoch_us` (microseconds since epoch), or `datetime` (YYYYMMDDhhmmss). Bound to `.unit = "start_sequence_source_type"`. | `[libavformat/hlsenc.c:L3166]` |
| 27 | `http_user_agent` | `AV_OPT_TYPE_STRING` | `user_agent` | `NULL` | n/a | Override the HTTP `User-Agent` header on all outbound HTTP requests (segment PUT, playlist PUT, DELETE). | `[libavformat/hlsenc.c:L3171]` |
| 28 | `var_stream_map` | `AV_OPT_TYPE_STRING` | `var_stream_map` | `NULL` | n/a | String mapping input streams to output variants and renditions. Syntax: `"v:0,a:0 v:1,a:1"` declares two variants each with one video and one audio stream. Enables multi-bitrate HLS. | `[libavformat/hlsenc.c:L3172]` |
| 29 | `cc_stream_map` | `AV_OPT_TYPE_STRING` | `cc_stream_map` | `NULL` | n/a | String mapping closed-captions tracks (CEA-608/708) to renditions for the master playlist's `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` entries. | `[libavformat/hlsenc.c:L3173]` |
| 30 | `master_pl_name` | `AV_OPT_TYPE_STRING` | `master_pl_name` | `NULL` | n/a | Filename for the master playlist (e.g., `master.m3u8`). Required when `var_stream_map` declares multiple variants. When NULL, no master playlist is written. | `[libavformat/hlsenc.c:L3174]` |
| 31 | `master_pl_publish_rate` | `AV_OPT_TYPE_INT` | `master_publish_rate` | `0` | `0..UINT_MAX` | Re-publish the master playlist every N segment intervals. `0` means publish only once. | `[libavformat/hlsenc.c:L3175]` |
| 32 | `http_persistent` | `AV_OPT_TYPE_BOOL` | `http_persistent` | `0` | `0..1` | When set and the destination is HTTP(S), reuses the underlying TCP connection across requests via `ff_http_do_new_request`. NOTE: the demuxer-side option of the same name defaults to `1`. | `[libavformat/hlsenc.c:L3176]` |
| 33 | `timeout` | `AV_OPT_TYPE_DURATION` | `timeout` | `-1` (no override) | `-1..INT_MAX` | Socket I/O timeout in microseconds, propagated to the HTTP protocol layer via `set_http_options`. `-1` means use the protocol layer's own default. | `[libavformat/hlsenc.c:L3177]` |
| 34 | `ignore_io_errors` | `AV_OPT_TYPE_BOOL` | `ignore_io_errors` | `0` | `0..1` | When set, swallows I/O errors during segment and playlist writes (log warning instead of returning `AVERROR(EIO)`). Useful for long-running live uploads where transient failures should not abort the stream. See [`./exception-handling.md`](./exception-handling.md). | `[libavformat/hlsenc.c:L3178]` |
| 35 | `headers` | `AV_OPT_TYPE_STRING` | `headers` | `NULL` | n/a | Custom HTTP headers, overriding any defaults set by `set_http_options`. Format follows the standard FFmpeg HTTP protocol header syntax (CRLF-delimited). | `[libavformat/hlsenc.c:L3179]` |



### §5.2 — `AV_OPT_TYPE_CONST` Enum-Value Aliases (23 Entries)

These entries are not standalone options — they bind to a parent option's `.unit` and let users pass symbolic names instead of integer codes. For example, `-hls_segment_type fmp4` and `-hls_segment_type 1` are equivalent. Every alias is enumerated individually below.

#### Aliases for `hls_segment_type` (`.unit = "segment_type"`) — 2 entries

| Const Name | Numeric Value | Effect | Source |
|---|---|---|---|
| `mpegts` | `SEGMENT_TYPE_MPEGTS` (0) | Segments are MPEG-TS files (`.ts`). Default. | `[libavformat/hlsenc.c:L3140]` |
| `fmp4` | `SEGMENT_TYPE_FMP4` (1) | Segments are fragmented MP4 files (`.m4s`); an initialization segment (`init.mp4` by default) is referenced via `#EXT-X-MAP`. Forces version 7. | `[libavformat/hlsenc.c:L3141]` |

#### Aliases for `hls_flags` (`.unit = "flags"`) — 15 entries

The 15 bit-mask flags are ORed together when combining behaviors. The integer value column shows the symbolic constant the alias resolves to; the actual numeric bit value is defined by the `HLSFlags` enum in `[libavformat/hlsenc.c:L99-L114]` — see [`../technical/data-model.md`](../technical/data-model.md).

| Const Name | Constant | Effect | Source |
|---|---|---|---|
| `single_file` | `HLS_SINGLE_FILE` | All segments are concatenated into one file; the playlist uses `#EXT-X-BYTERANGE` to address them. Forces version 4. | `[libavformat/hlsenc.c:L3145]` |
| `temp_file` | `HLS_TEMP_FILE` | Segment files are written as `<name>.tmp` first and atomically renamed once closed; prevents partial-file reads by clients. Only effective when the protocol is `file`. | `[libavformat/hlsenc.c:L3146]` |
| `delete_segments` | `HLS_DELETE_SEGMENTS` | Old segment files (and subtitle segments) are deleted from disk (or via HTTP DELETE) when evicted from the playlist by the sliding window. | `[libavformat/hlsenc.c:L3147]` |
| `round_durations` | `HLS_ROUND_DURATIONS` | Emit `#EXTINF` as an integer (`lrint(duration)`) rather than a float; prevents bumping `#EXT-X-VERSION` to 3. | `[libavformat/hlsenc.c:L3148]` |
| `discont_start` | `HLS_DISCONT_START` | Emit an `#EXT-X-DISCONTINUITY` line at the top of the playlist for the very first segment (signals to clients that this stream is a continuation of an unrelated source). | `[libavformat/hlsenc.c:L3149]` |
| `omit_endlist` | `HLS_OMIT_ENDLIST` | Suppress the trailing `#EXT-X-ENDLIST` tag even at end-of-stream; keeps the playlist appearing "live" forever. | `[libavformat/hlsenc.c:L3150]` |
| `split_by_time` | `HLS_SPLIT_BY_TIME` | Allow segment cuts at non-keyframe boundaries (time-only criterion). Risks decoder-side artifacts; only useful when input is all-keyframe or audio-only. | `[libavformat/hlsenc.c:L3151]` |
| `append_list` | `HLS_APPEND_LIST` | Append new segments to an existing playlist file rather than overwriting it. Disables `hls_init_time` behavior. | `[libavformat/hlsenc.c:L3152]` |
| `program_date_time` | `HLS_PROGRAM_DATE_TIME` | Emit `#EXT-X-PROGRAM-DATE-TIME` per segment using ISO-8601 format with millisecond precision and timezone offset. | `[libavformat/hlsenc.c:L3153]` |
| `second_level_segment_index` | `HLS_SECOND_LEVEL_SEGMENT_INDEX` | When `strftime=1`, include the segment index in the expanded filename (substitutes `%d` after strftime processing). | `[libavformat/hlsenc.c:L3154]` |
| `second_level_segment_duration` | `HLS_SECOND_LEVEL_SEGMENT_DURATION` | When `strftime=1`, include the segment duration in the expanded filename (substitutes `%t` after strftime processing). | `[libavformat/hlsenc.c:L3155]` |
| `second_level_segment_size` | `HLS_SECOND_LEVEL_SEGMENT_SIZE` | When `strftime=1`, include the segment byte size in the expanded filename (substitutes `%s` after strftime processing). | `[libavformat/hlsenc.c:L3156]` |
| `periodic_rekey` | `HLS_PERIODIC_REKEY` | Re-read `hls_key_info_file` on every segment boundary, allowing the key URI / IV to rotate at any time. Without this flag the key info is read once at startup. | `[libavformat/hlsenc.c:L3157]` |
| `independent_segments` | `HLS_INDEPENDENT_SEGMENTS` | Emit `#EXT-X-INDEPENDENT-SEGMENTS` in the playlist (signals clients that every segment begins with an independent decoder state). Only emitted when `vs->has_video` is true. Forces version 6. | `[libavformat/hlsenc.c:L3158]` |
| `iframes_only` | `HLS_I_FRAMES_ONLY` | Emit `#EXT-X-I-FRAMES-ONLY` for I-frame-only playlists (used for trick-play streams). Forces version 4. | `[libavformat/hlsenc.c:L3159]` |

#### Aliases for `hls_playlist_type` (`.unit = "pl_type"`) — 2 entries

| Const Name | Constant | Effect | Source |
|---|---|---|---|
| `event` | `PLAYLIST_TYPE_EVENT` | Emit `#EXT-X-PLAYLIST-TYPE:EVENT` — append-only live stream; segments are appended over time and never removed. | `[libavformat/hlsenc.c:L3163]` |
| `vod` | `PLAYLIST_TYPE_VOD` | Emit `#EXT-X-PLAYLIST-TYPE:VOD` — complete, finite stream. With VOD, the playlist is only published once at end-of-stream (see `[libavformat/hlsenc.c:L2627]`). | `[libavformat/hlsenc.c:L3164]` |

#### Aliases for `hls_start_number_source` (`.unit = "start_sequence_source_type"`) — 4 entries

| Const Name | Constant | Effect | Source |
|---|---|---|---|
| `generic` | `HLS_START_SEQUENCE_AS_START_NUMBER` | Use the `start_number` option value verbatim. Default. | `[libavformat/hlsenc.c:L3167]` |
| `epoch` | `HLS_START_SEQUENCE_AS_SECONDS_SINCE_EPOCH` | Set initial sequence to seconds since the Unix epoch at start time. | `[libavformat/hlsenc.c:L3168]` |
| `epoch_us` | `HLS_START_SEQUENCE_AS_MICROSECONDS_SINCE_EPOCH` | Set initial sequence to microseconds since the Unix epoch at start time. | `[libavformat/hlsenc.c:L3169]` |
| `datetime` | `HLS_START_SEQUENCE_AS_FORMATTED_DATETIME` | Set initial sequence to `YYYYMMDDhhmmss` formatted as an integer. | `[libavformat/hlsenc.c:L3170]` |



## Muxer Outputs: Segment Files (TS and fMP4)

### Plain-Language Summary

For each variant stream, the HLS muxer emits a sequence of media segment files on disk (or via HTTP PUT). The format depends on `hls_segment_type`: MPEG-TS (the default) produces 188-byte-aligned `.ts` files written by an internal `ff_mpegts_muxer` child context, while fMP4 produces fragmented MP4 `.m4s` files plus a single initialization segment (`init.mp4` by default) written by the MP4 muxer in fragment mode. The naming scheme follows the user's `hls_segment_filename` template, or — when that option is unset — the playlist filename with the suffix `_%d` (from `POSTFIX_PATTERN` at `[libavformat/hlsenc.c:L74]`). Two special modes alter the file layout: `single_file` concatenates all segments into one byteranged file, and `temp_file` writes through a `.tmp` filename and atomically renames the result.

### Technical Detail

| Output Artifact | Format | Default Filename Pattern | Trigger / Rule | Source |
|---|---|---|---|---|
| Media segment (MPEG-TS, default) | MPEG-TS, 188-byte packets, no `.styp` header | `<basename>_<N>.ts` where `<N>` is the sequence number derived from `POSTFIX_PATTERN "_%d"` | Emitted on every segment cut when `hls_segment_type=mpegts` (default). | `[libavformat/hlsenc.c:L74]` (POSTFIX_PATTERN), `[libavformat/hlsenc.c:L3139-L3140]` |
| Media segment (fMP4) | Fragmented MP4 with `styp` box per segment | `<basename>_<N>.m4s` (extension provided by sub-muxer; the exact suffix is taken from the user's `hls_segment_filename` template) | Emitted on every segment cut when `hls_segment_type=fmp4`. The `styp` box is written via `write_styp(vs->out)` at L2580. | `[libavformat/hlsenc.c:L2580, L3141]` |
| Initialization segment (fMP4 only) | MP4 init box (`ftyp` + `moov`) — contains decoder configuration but no media samples | `init.mp4` (configurable via `hls_fmp4_init_filename`) | Emitted once at startup when `hls_segment_type=fmp4`. Resent on every playlist refresh when `hls_fmp4_init_resend=1` (see `hls_init_file_resend` at L2362-L2377). | `[libavformat/hlsenc.c:L2362-L2377, L3142, L3143]` |
| Single-file output (single_file mode) | One file containing all segments concatenated; segments addressed by `#EXT-X-BYTERANGE:size@offset` | Determined entirely by `hls_segment_filename`; the `_%d` suffix is NOT applied | Activated by `hls_flags single_file` (`HLS_SINGLE_FILE`). The byterange tracking is driven by `byterange_mode = (hls->flags & HLS_SINGLE_FILE) \|\| (hls->max_seg_size > 0)` at L2504. Forces `#EXT-X-VERSION:4`. | `[libavformat/hlsenc.c:L2504, L3145]` |
| Temp-file (atomic rename) mode | Same content as default segments; only the file path differs while open | `<filename>.tmp` until close, then renamed via `hls_rename_temp_file` | Activated by `hls_flags temp_file` (`HLS_TEMP_FILE`). Only effective when the destination protocol is `file` (HTTP destinations skip temp-renaming). The detection is `use_temp_file = proto && !strcmp(proto, "file") && (hls->flags & HLS_TEMP_FILE)` at L2546-L2547. | `[libavformat/hlsenc.c:L2546-L2547, L2605-L2606, L3146]` |
| Encrypted segment wrapper | Same on-the-wire format as MPEG-TS or fMP4; the bytes are passed through the AES-128 encryptor before write | n/a (filename is the same; payload is encrypted) | Activated when `hls->key_info_file` or `hls->encrypt` is set. The filename is wrapped as `crypto:<original_url>` and `encryption_key` / `encryption_iv` dict entries are set on `hlsenc_io_open` (L2553-L2557). | `[libavformat/hlsenc.c:L2553-L2557]` |
| Subtitle segments (WebVTT) | WebVTT files | Derived from the per-variant subtitle URL `vs->vtt_avf->url` | Emitted for each subtitle stream when subtitle inputs are present. Closed alongside the media segment at L2529-L2531. | `[libavformat/hlsenc.c:L2529-L2531]` |

The actual filename construction in default (multi-file) mode applies `replace_int_data_in_filename` / `replace_str_data_in_filename` to substitute `%d` (sequence number), `%v` (variant index) and similar tokens. When `strftime=1`, `strftime_expand` is applied to substitute `%Y`, `%m`, `%H`, etc. into the filename before write.

---

## Muxer Outputs: M3U8 Playlist Tag Emissions

### Plain-Language Summary

Every M3U8 playlist published by the HLS muxer is composed of `#EXT-X-*` directives followed by per-segment entries. The structure is: header tags (version, target duration, sequence, playlist type, allow-cache), optional rendition lines (master playlist only), per-segment lines (key, init-file, discontinuity, EXTINF, byterange, program-date-time, URI), and optional trailer (`#EXT-X-ENDLIST`). Each emission is produced by exactly one `avio_printf` call inside one of seven writer functions in `[libavformat/hlsplaylist.c]` or directly inside `hls_window` in `[libavformat/hlsenc.c]`. The table below enumerates every distinct emission individually.

### Technical Detail

The writer functions and their format strings are all visible in `[libavformat/hlsplaylist.c]` (206 lines total). The conditional emissions inside `hls_window` itself span `[libavformat/hlsenc.c:L1531-L1660]`.

| Tag / Line | Conditional Emission Rule | Writer | Format String / Notes | Source |
|---|---|---|---|---|
| `#EXTM3U` | Always (line 1 of every playlist) | `ff_hls_write_playlist_version` | Literal `"#EXTM3U\n"` | `[libavformat/hlsplaylist.c:L36]` |
| `#EXT-X-VERSION:%d` | Always (line 2 of every playlist) | `ff_hls_write_playlist_version` | `"#EXT-X-VERSION:%d\n"` where the version is negotiated by `hls_window` at `[libavformat/hlsenc.c:L1551-L1571]` (values 2, 3, 4, 6, or 7) | `[libavformat/hlsplaylist.c:L37]` |
| `#EXT-X-MEDIA:TYPE=AUDIO,...` | Per audio rendition declared via `var_stream_map` | `ff_hls_write_audio_rendition` | Composes `TYPE=AUDIO,GROUP-ID="group_<agroup>",NAME="audio_<N>",DEFAULT=<YES\|NO>,LANGUAGE=<lang>,CHANNELS="<N>",URI="<rel-uri>"` | `[libavformat/hlsplaylist.c:L40-L56]` |
| `#EXT-X-MEDIA:TYPE=SUBTITLES,...` | Per subtitle rendition declared via `var_stream_map` | `ff_hls_write_subtitle_rendition` | Composes `TYPE=SUBTITLES,GROUP-ID="<sgroup>",NAME="<sub_N>",DEFAULT=<YES\|NO>,LANGUAGE=<lang>,URI="<rel-uri>"` | `[libavformat/hlsplaylist.c:L58-L75]` |
| `#EXT-X-STREAM-INF:BANDWIDTH=%d,...` | Per variant in the master playlist (when `master_pl_name` is set) | `ff_hls_write_stream_info` | Composes `BANDWIDTH=%d`, optionally `,AVERAGE-BANDWIDTH=%d`, `,RESOLUTION=%dx%d`, `,FRAME-RATE=%.3f`, `,CODECS="%s"`, `,AUDIO="group_<agroup>"`, `,CLOSED-CAPTIONS="%s"`, `,SUBTITLES="%s"`, followed by optional `NAME=` and the per-variant playlist URI on the next line | `[libavformat/hlsplaylist.c:L78-L108]` |
| `#EXT-X-ALLOW-CACHE:YES\|NO` | Emitted only when `hls_allow_cache == 0` or `1` (i.e., NOT when it is `-1`). Note: this tag is conditional on `allowcache == 0 \|\| allowcache == 1` per L115-L116. | `ff_hls_write_playlist_header` | `"#EXT-X-ALLOW-CACHE:%s\n"` with `"YES"` or `"NO"` | `[libavformat/hlsplaylist.c:L115-L119]` |
| `#EXT-X-TARGETDURATION:%d` | Always in a media playlist | `ff_hls_write_playlist_header` | `"#EXT-X-TARGETDURATION:%d\n"` — the target is the integer maximum of all current segment durations (computed by `hls_window` at L1583-L1586 via `lrint`) | `[libavformat/hlsplaylist.c:L120]` |
| `#EXT-X-MEDIA-SEQUENCE:%"PRId64"` | Always in a media playlist | `ff_hls_write_playlist_header` | `"#EXT-X-MEDIA-SEQUENCE:%"PRId64"\n"` — the sequence number is the first-segment number computed by `hls_window` at L1539 | `[libavformat/hlsplaylist.c:L121]` |
| `#EXT-X-PLAYLIST-TYPE:EVENT` | When `pl_type == PLAYLIST_TYPE_EVENT` | `ff_hls_write_playlist_header` | Literal `"#EXT-X-PLAYLIST-TYPE:EVENT\n"` | `[libavformat/hlsplaylist.c:L122-L125]` |
| `#EXT-X-PLAYLIST-TYPE:VOD` | When `pl_type == PLAYLIST_TYPE_VOD` | `ff_hls_write_playlist_header` | Literal `"#EXT-X-PLAYLIST-TYPE:VOD\n"` | `[libavformat/hlsplaylist.c:L126-L128]` |
| `#EXT-X-I-FRAMES-ONLY` | When `iframe_mode != 0` (set by caller when `HLS_I_FRAMES_ONLY` flag is active) | `ff_hls_write_playlist_header` | Literal `"#EXT-X-I-FRAMES-ONLY\n"` | `[libavformat/hlsplaylist.c:L129-L131]` |
| `#EXT-X-DISCONTINUITY` (playlist-top variant) | When `(hls->flags & HLS_DISCONT_START) && sequence == hls->start_sequence && vs->discontinuity_set == 0` | Direct `avio_printf` inside `hls_window` | Literal `"#EXT-X-DISCONTINUITY\n"`. Sets `vs->discontinuity_set = 1` to prevent duplicate emission. | `[libavformat/hlsenc.c:L1593-L1596]` |
| `#EXT-X-INDEPENDENT-SEGMENTS` | When `vs->has_video && (hls->flags & HLS_INDEPENDENT_SEGMENTS)` | Direct `avio_printf` inside `hls_window` | Literal `"#EXT-X-INDEPENDENT-SEGMENTS\n"`. NOT emitted for audio-only variants. | `[libavformat/hlsenc.c:L1597-L1599]` |
| `#EXT-X-KEY:METHOD=AES-128,URI="..."` | When `(hls->encrypt \|\| hls->key_info_file)` AND the key URI or IV differs from the previously emitted key — emitted before the segment(s) it applies to | Direct `avio_printf` inside the segment loop of `hls_window` | `"#EXT-X-KEY:METHOD=AES-128,URI=\"%s\""` then optionally `",IV=0x%s"` then `"\n"` | `[libavformat/hlsenc.c:L1601-L1609]` |
| `#EXT-X-MAP:URI="..."` | When `hls->segment_type == SEGMENT_TYPE_FMP4` AND this is the first segment being written | `ff_hls_write_init_file` (called from `hls_window` at L1612) | `"#EXT-X-MAP:URI=\"%s\""`, optionally `",BYTERANGE=\"%"PRId64"@%"PRId64"\""`, then `"\n"` | `[libavformat/hlsplaylist.c:L134-L142]`, called from `[libavformat/hlsenc.c:L1611-L1614]` |
| `#EXT-X-DISCONTINUITY` (per-segment variant) | When the segment's `en->discont` flag is set (passed as `insert_discont` parameter) | `ff_hls_write_file_entry` | Literal `"#EXT-X-DISCONTINUITY\n"` — emitted immediately before that segment's `#EXTINF` line | `[libavformat/hlsplaylist.c:L156-L158]` |
| `#EXTINF:<duration>,` (integer form) | When the caller passes `round_duration != 0` (set by `HLS_ROUND_DURATIONS` flag) | `ff_hls_write_file_entry` | `"#EXTINF:%ld,\n"` with `lrint(duration)`. Keeps EXT-X-VERSION at 2. | `[libavformat/hlsplaylist.c:L159-L160]` |
| `#EXTINF:<duration>,` (float form) | When `round_duration == 0` (default — float durations) | `ff_hls_write_file_entry` | `"#EXTINF:%f,\n"`. Float durations force `#EXT-X-VERSION:3` minimum. | `[libavformat/hlsplaylist.c:L161-L162]` |
| `#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"` | When `byterange_mode != 0` (set by `HLS_SINGLE_FILE` or `max_seg_size > 0`) | `ff_hls_write_file_entry` | `"#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n"` with `size@offset`; when `iframe_mode` is set, uses `video_keyframe_size` and `video_keyframe_pos` instead | `[libavformat/hlsplaylist.c:L163-L165]` |
| `#EXT-X-PROGRAM-DATE-TIME:...` | When the caller passes a non-NULL `prog_date_time` pointer (set when `HLS_PROGRAM_DATE_TIME` flag is on or when a per-segment `discont_program_date_time` override exists) | `ff_hls_write_file_entry` | `"#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s\n"` — ISO 8601 form `YYYY-MM-DDTHH:MM:SS.millis±zone` (e.g., `2024-06-15T14:30:45.123+0100`) | `[libavformat/hlsplaylist.c:L167-L192]` |
| `<baseurl>` prefix | When `hls_base_url` is set | `ff_hls_write_file_entry` | Prepended as a raw string (no trailing newline) before the segment URI | `[libavformat/hlsplaylist.c:L194-L195]` |
| `<segment-filename>\n` | Always, per segment | `ff_hls_write_file_entry` | `"%s\n"` — the segment's relative URI (or full URI with baseurl prepended) | `[libavformat/hlsplaylist.c:L196]` |
| `#EXT-X-ENDLIST` | When `last && (hls->flags & HLS_OMIT_ENDLIST) == 0` — i.e., at end-of-stream AND `omit_endlist` flag NOT set | `ff_hls_write_end_list` (called from `hls_window` at L1630) | Literal `"#EXT-X-ENDLIST\n"` | `[libavformat/hlsplaylist.c:L201-L206]`, called from `[libavformat/hlsenc.c:L1629-L1630]` |

#### Notes on EXT-X-* Emission Sites

`#EXT-X-DISCONTINUITY` is emitted in **two distinct places** with different conditional rules:

1. **At the top of the playlist**, once, when `HLS_DISCONT_START` flag is set and this is the first publish (`vs->discontinuity_set == 0`). The emission site is in `hls_window` at `[libavformat/hlsenc.c:L1594]`.
2. **Before each segment's `#EXTINF`**, when that segment was marked discontinuous (typically because of a mid-stream input gap or `append_list` rejoin). The emission site is in `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L157]`.

`#EXT-X-KEY` is emitted **only when the key URI or IV changes** from the previously emitted value (or for the very first encrypted segment). This is the per-segment loop inside `hls_window` at `[libavformat/hlsenc.c:L1601-L1609]`. When `HLS_PERIODIC_REKEY` is active, the key info file is re-read at each segment boundary, so the URI or IV can change on any segment.

`#EXT-X-INDEPENDENT-SEGMENTS` is gated on `vs->has_video` — audio-only variants do NOT receive this tag even when the user sets the flag.

The HLS version negotiation at `[libavformat/hlsenc.c:L1551-L1571]` cascades as follows: start at version 2; bump to 3 unless `HLS_ROUND_DURATIONS` is set; bump to 4 if byterange mode is active or `HLS_I_FRAMES_ONLY` is set; bump to 6 if `HLS_INDEPENDENT_SEGMENTS` is set; bump to 7 if `hls_segment_type == SEGMENT_TYPE_FMP4`. The negotiated version is emitted by `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L37]`.



## Muxer Outputs: Sub-Muxer Callbacks

### Plain-Language Summary

The HLS muxer never writes packet payloads to disk directly. Instead, it is a *meta-muxer*: each `VariantStream` owns a child `AVFormatContext` (the `vs->avf` field) bound either to the MPEG-TS muxer (`ff_mpegts_muxer`) or to the MP4 muxer in fragment mode, plus an optional `vs->vtt_avf` for WebVTT subtitle output. The HLS muxer dispatches the standard avformat write API (`avformat_write_header`, `av_write_frame`, `av_write_trailer`) on these child contexts to materialize segments. Codec parameters, `extradata`, time-bases, and AVFormatContext fields flow from the parent context down into each child during `hls_mux_init`. The child sub-muxer is responsible for the actual byte format of the segment file.

### Technical Detail

| Forwarded Call | Target Sub-Muxer | When | Source |
|---|---|---|---|
| `avformat_alloc_output_context2(&vs->avf, oformat, ...)` | Allocates the child context bound to either `ff_mpegts_muxer` or the MP4 muxer in fragment mode | Called from `hls_mux_init` for each variant during `hls_init` / `hls_write_header` | `[libavformat/hlsenc.c:L773+]` |
| `avformat_write_header(vs->avf, NULL)` | Child MPEG-TS or fMP4 muxer | Called once per variant from `hls_write_header` at L2311 | `[libavformat/hlsenc.c:L2311]` |
| `avpriv_set_pts_info(...)` | Propagates timestamp info into the child sub-muxer's stream | Called from `hls_write_header` at L2338 for each variant stream — ensures sub-muxer timestamps use a consistent time base | `[libavformat/hlsenc.c:L2338]` |
| `ff_write_chained(oc, stream_index, pkt, s, 0)` | Child MPEG-TS or fMP4 muxer (`oc` points to `vs->avf` for media or `vs->vtt_avf` for subtitles) | Called from `hls_write_packet` at L2679 for every accepted packet | `[libavformat/hlsenc.c:L2679]` |
| `av_write_frame(oc, NULL)` (flush) | Child muxer (segment boundary) | Called from `hls_write_packet` at L2507 to flush buffered data right before closing the current segment file | `[libavformat/hlsenc.c:L2507]` |
| `av_write_frame(ctx, NULL)` (flush during fMP4 init capture) | Child fMP4 muxer | Called from `flush_dynbuf` at L489 immediately before `avio_close_dyn_buf` extracts the init.mp4 bytes | `[libavformat/hlsenc.c:L489]` |
| `av_write_trailer(vs->avf)` | Child muxer | Called for each variant from `hls_write_trailer` at `[libavformat/hlsenc.c:L2727+]` to finalize the last segment | `[libavformat/hlsenc.c:L2727+]` |
| Codec parameter copy (`avcodec_parameters_copy`) | Each `AVStream` in the child sub-muxer | Performed inside `hls_mux_init` so the sub-muxer sees the same codec parameters as the parent | `[libavformat/hlsenc.c:L773+]` |
| Field propagation (`max_delay`, `opaque`, `io_open`, `io_close2`, `strict_std_compliance`, `interrupt_callback`, `metadata`) | Each child `AVFormatContext` | Performed inside `hls_mux_init` to ensure parent-context configuration flows into the child | `[libavformat/hlsenc.c:L773+]` |
| `av_opt_set(oc->priv_data, "mpegts_flags", "resend_headers", 0)` | Child MPEG-TS muxer (in `single_file` mode after a cut) | Called from `hls_write_packet` at L2651 to force PAT/PMT re-emission at every segment boundary | `[libavformat/hlsenc.c:L2650-L2652]` |
| `av_opt_set(&options, "mpegts_flags", "resend_headers", 0)` | Child MPEG-TS muxer (in temp-file mode) | Called from `hls_write_packet` at L2567 — every temp-file segment starts with full PAT/PMT/headers | `[libavformat/hlsenc.c:L2566-L2567]` |
| WebVTT sub-muxer dispatch (`vs->vtt_avf`) | Child WebVTT muxer | Subtitle packets are routed to `vs->vtt_avf` instead of `vs->avf` (see L2434 in `hls_write_packet`) | `[libavformat/hlsenc.c:L2434]` |

---

## Muxer Outputs: HTTP Requests

### Plain-Language Summary

When the destination URL uses an HTTP(S) protocol (detected by `ff_is_http_proto` at `[libavformat/hlsenc.c:L296, L316, L335]`), every segment write and every playlist publish results in an HTTP request to the destination. The default method is **PUT**; users can override via the `method` option. Sliding-window eviction (when `HLS_DELETE_SEGMENTS` is active) issues HTTP **DELETE** requests through a separate persistent `http_delete` `AVIOContext` (the `hls->http_delete` field at L261). The `http_persistent` option lets the muxer reuse a single underlying TCP connection across requests by calling `ff_http_do_new_request` instead of opening a fresh AVIOContext.

### Technical Detail

| HTTP Method | Trigger | Body | Target URL | Source |
|---|---|---|---|---|
| `PUT` (default for segment write) | Every segment finalization when destination is http(s) and `method` option is unset | Segment bytes (MPEG-TS or fMP4) | Per-segment URL (segment_filename expanded) | `[libavformat/hlsenc.c:L339-L341]` (`set_http_options` sets `"method" = "PUT"` when http and no override), `[libavformat/hlsenc.c:L2571]` (the `hlsenc_io_open` call) |
| `PUT` (default for playlist write) | Every playlist publish in `hls_window` when destination is http(s) | M3U8 playlist body | Per-variant playlist URL (or master playlist URL) | `[libavformat/hlsenc.c:L339-L341]`, `[libavformat/hlsenc.c:L1531-L1660]` |
| `DELETE` | Sliding-window eviction when `HLS_DELETE_SEGMENTS` flag is set and the destination is http (or when `method` option is set so HTTP delete is forced) | Empty body | URL of the segment being evicted | `[libavformat/hlsenc.c:L510-L527]` (`hls_delete_file`) — sets `"method" = "DELETE"` at L515 and opens via `hls->http_delete` persistent AVIOContext at L517 |
| Custom method (user override) | When `-method <verb>` is set on the muxer | Same body as PUT (segment or playlist) | Per-segment / per-playlist URL | `[libavformat/hlsenc.c:L337-L338, L3165]` |
| Header propagation | All HTTP requests | n/a | n/a | `set_http_options` at `[libavformat/hlsenc.c:L333-L350]` sets `"user_agent"` from `http_user_agent`, `"multiple_requests" = 1` when `http_persistent`, `"timeout"` when `timeout >= 0`, and a raw `"headers"` blob when `headers` option is set |
| TCP-connection reuse | All HTTP PUT and DELETE when `http_persistent=1` | Same as base requests; the change is at the transport layer | n/a | `[libavformat/hlsenc.c:L298-L308]` (`hlsenc_io_open` calls `ff_http_do_new_request` instead of `s->io_open` when a persistent context exists) |

#### HTTP DELETE Behavior

The HTTP DELETE path is gated by `hls->method` OR by the protocol being plain `"http"` (at L510). When neither condition holds (i.e., file destination), the segment is removed via the local `unlink(path)` syscall at L524. DELETE failures are degraded according to the `ignore_io_errors` flag: when set, the function returns `1` (continue) instead of propagating the error (L519-L520).

#### When HTTP-Persistent Connections Are Skipped

`hlsenc_io_close` at L313-L331 deliberately *avoids* the persistent path when encryption is active (`hls->key_info_file || hls->encrypt`) — see L320. This is because the AES-128 crypto wrapper opens via a `crypto:<url>` URL that does not share state with the underlying HTTP context. In this case `ff_format_io_close` is called normally and a fresh AVIOContext is opened for the next request.



## Demuxer Inputs: M3U8 Line Types Accepted by `hls.c` Parser

### Plain-Language Summary

The HLS demuxer parses the M3U8 manifest line-by-line. Every line that begins with `#EXT-X-` is checked against an explicit if/else chain in `parse_playlist` at `[libavformat/hls.c:L862-L1080]`. Lines that match a known tag drive parser state (variant declaration, key info, target duration, sequence number, playlist type, init section, start offset, end-of-list, segment duration, byterange); lines that don't match any explicit tag but begin with `#` are silently skipped via the catch-all at L988-L990; non-comment lines are interpreted as URLs that complete either a variant declaration (after `#EXT-X-STREAM-INF`) or a segment entry (after `#EXTINF`). The parser is comparatively narrow — several common HLS tags are NOT explicitly handled and are silently ignored.

### Technical Detail

The parser's `if-else` chain is rooted at `[libavformat/hls.c:L862]` and uses `av_strstart(line, "<tag>", &ptr)` to detect each tag. Each row below corresponds to one branch in that chain.

| M3U8 Line | Effect | Source |
|---|---|---|
| `#EXT-X-STREAM-INF:<attrs>` | Sets `is_variant = 1`, zero-initializes a `variant_info` struct, parses key-value attributes via `handle_variant_args` (BANDWIDTH, RESOLUTION, CODECS, AUDIO, SUBTITLES, etc.). The *next* non-comment line becomes the variant's playlist URL via `new_variant`. | `[libavformat/hls.c:L863-L866, L992-L998]` |
| `#EXT-X-KEY:METHOD=<m>[,URI=<u>][,IV=<iv>]` | Parses METHOD (`AES-128` → `KEY_AES_128`, `SAMPLE-AES` → `KEY_SAMPLE_AES`, otherwise `KEY_NONE`), URI (copied to `key`), and IV (when starts with `0x`, parsed via `ff_hex_to_data` into a 16-byte buffer). The key state applies to all subsequent segments until the next `#EXT-X-KEY`. | `[libavformat/hls.c:L867-L880]` |
| `#EXT-X-MEDIA:<attrs>` | Zero-initializes `rendition_info`, parses TYPE/GROUP-ID/NAME/DEFAULT/LANGUAGE/URI/CHANNELS, calls `new_rendition` to add the rendition to `c->renditions`. | `[libavformat/hls.c:L881-L884]` |
| `#EXT-X-TARGETDURATION:<int>` | Calls `ensure_playlist`, parses the integer via `strtoll`. Range check: rejects negative values and values that would overflow when multiplied by `AV_TIME_BASE`. On success, sets `pls->target_duration = t * AV_TIME_BASE`. | `[libavformat/hls.c:L885-L895]` |
| `#EXT-X-MEDIA-SEQUENCE:<int>` | Calls `ensure_playlist`, parses the value as unsigned 64-bit via `strtoull`. If the value exceeds `INT64_MAX/2`, masks out the high bit to coerce to a signed range, then assigns to `pls->start_seq_no`. | `[libavformat/hls.c:L896-L907]` |
| `#EXT-X-PLAYLIST-TYPE:EVENT\|VOD` | Calls `ensure_playlist`, then maps the tail of the line to `PLS_TYPE_EVENT` or `PLS_TYPE_VOD` (the demuxer's `enum PlaylistType` — see `[libavformat/hls.c:L91-L96]`). | `[libavformat/hls.c:L908-L915]` |
| `#EXT-X-MAP:<attrs>` | Calls `ensure_playlist`, parses URI and BYTERANGE via `handle_init_section_args`, allocates a new init section via `new_init_section`. Copies the current key state (key_type, IV) onto the init section. When key_type is not `KEY_NONE`, resolves the URI to absolute form via `ff_make_absolute_url` and duplicates it into `cur_init_section->key`. | `[libavformat/hls.c:L916-L952]` |
| `#EXT-X-START:TIME-OFFSET=<float>` | Calls `ensure_playlist`. When the attribute is `TIME-OFFSET=...`, parses the float and stores it as `pls->start_time_offset = offset * AV_TIME_BASE`. Sets `pls->time_offset_flag = 1`. Any other attribute logs a warning and is skipped. | `[libavformat/hls.c:L953-L967]` |
| `#EXT-X-ENDLIST` | Sets `pls->finished = 1`, marking the playlist as complete (VOD or live-finished). | `[libavformat/hls.c:L968-L970]` |
| `#EXTINF:<float>,[<title>]` | Parses the duration as a `double`, multiplies by `AV_TIME_BASE`. Negative, infinite, or NaN values are coerced to 0 with a warning. Sets `is_segment = 1` and stashes `duration` so the next non-comment URL line becomes a segment. | `[libavformat/hls.c:L971-L978]` |
| `#EXT-X-BYTERANGE:<size>[@<offset>]` | Parses `seg_size` via `strtoll`. When `@` is present, parses `seg_offset`. Range check: rejects negative size and rejects offset+size overflow. The values are applied to the next segment URL. | `[libavformat/hls.c:L979-L987]` |
| `#<anything-else>` | Catch-all: logs `"Skip ('%s')"` at `AV_LOG_VERBOSE` level and continues. **Silently ignored tags include `#EXT-X-VERSION`, `#EXT-X-ALLOW-CACHE`, `#EXT-X-INDEPENDENT-SEGMENTS`, `#EXT-X-I-FRAMES-ONLY`, `#EXT-X-PROGRAM-DATE-TIME`, `#EXT-X-DISCONTINUITY`, `#EXT-X-DISCONTINUITY-SEQUENCE`, `#EXT-X-GAP`, `#EXT-X-DEFINE`, `#EXT-X-PART-INF`, `#EXT-X-SERVER-CONTROL`, `#EXT-X-SKIP`, `#EXT-X-PART`, `#EXT-X-PRELOAD-HINT`, `#EXT-X-RENDITION-REPORT`, `#EXT-X-DATERANGE`, `#EXT-X-SESSION-DATA`, `#EXT-X-SESSION-KEY`, `#EXT-X-BITRATE`** — and any future tag the parser has not been updated to handle. | `[libavformat/hls.c:L988-L990]` |
| Plain URL line | When `is_variant`, creates a variant via `new_variant`. When `is_segment`, creates a `struct segment`, copies the current key state and IV onto it, resolves the URL to absolute via `ff_make_absolute_url`, sets duration / size / offset / `init_section` pointer, and appends to `pls->segments`. | `[libavformat/hls.c:L991-L1080]` |

#### Implications of Silent-Skip Behavior

Because `#EXT-X-DISCONTINUITY` is silently skipped, the demuxer detects discontinuities **indirectly** — by observing PTS gaps between consecutive segments rather than by parsing the explicit tag. Similarly, `#EXT-X-PROGRAM-DATE-TIME` is parsed by the muxer's emission path but the demuxer derives ID3-based timing instead (see `fill_timing_for_id3_timestamped_stream` at L2573-L2576 in `hls_read_packet`). `#EXT-X-INDEPENDENT-SEGMENTS` and `#EXT-X-I-FRAMES-ONLY` are likewise informational only — the demuxer performs no special handling.

The `#EXT-X-VERSION` line is silently skipped — the demuxer is version-agnostic and tries to parse any playlist it receives.



## Demuxer Inputs: AVOptions (Complete Enumeration — 12 Entries)

### Plain-Language Summary

The HLS demuxer exposes 12 user-tunable AVOptions controlling live-stream entry point, allowed file extensions, retry behavior, and HTTP transport tuning. Every entry is enumerated below. Note that the demuxer-side `http_persistent` defaults to `1` (enabled) — opposite of the muxer-side default of `0`.

### Source Anchor

The complete option array is at `[libavformat/hls.c:L2850-L2891]`. The `OFFSET(x)` macro at L2850 expands to `offsetof(HLSContext, x)` (note: this is the *demuxer's* `HLSContext` struct, distinct from the muxer's — see `[libavformat/hls.c:L204+]` and [`../technical/data-model.md`](../technical/data-model.md)). The `FLAGS` macro at L2851 expands to `AV_OPT_FLAG_DECODING_PARAM` — every demuxer option is marked decode-direction.

### §11.1 — Demuxer AVOptions

| # | Option Name | Type | OFFSET | Default | Bounds | Effect | Source |
|---|---|---|---|---|---|---|---|
| 1 | `live_start_index` | `AV_OPT_TYPE_INT` | `live_start_index` | `-3` | `INT_MIN..INT_MAX` | Segment index from which a live stream begins playback. Negative values count from the end of the current playlist (e.g., `-3` starts three segments before the live edge — the HLS-recommended live latency). | `[libavformat/hls.c:L2853-L2854]` |
| 2 | `prefer_x_start` | `AV_OPT_TYPE_BOOL` | `prefer_x_start` | `0` | `0..1` | When `1` and the playlist contains `#EXT-X-START:TIME-OFFSET=...`, use that offset instead of `live_start_index`. | `[libavformat/hls.c:L2855-L2856]` |
| 3 | `allowed_extensions` | `AV_OPT_TYPE_STRING` | `allowed_extensions` | `"3gp,aac,avi,ac3,eac3,flac,mkv,m3u8,m4a,m4s,m4v,mpg,mov,mp2,mp3,mp4,mpeg,mpegts,ogg,ogv,oga,ts,vob,vtt,wav,webvtt,cmfv,cmfa,ec3,fmp4"` | `INT_MIN..INT_MAX` | Comma-separated whitelist of file extensions the demuxer is allowed to open. Restricts SSRF risk by refusing playlist URIs with unknown extensions. The default list combines standard HLS extensions with workarounds for specific services (cmfv/cmfa per Ticket 11526, ec3 per Ticket 11435, fmp4 per yt-dlp issue 12700). | `[libavformat/hls.c:L2857-L2864]` |
| 4 | `allowed_segment_extensions` | `AV_OPT_TYPE_STRING` | `allowed_segment_extensions` | Same as `allowed_extensions` PLUS `,html` (for `https://flash1.bogulus.cfd/` workaround) | `INT_MIN..INT_MAX` | Comma-separated whitelist of file extensions allowed for individual segment URIs (separate from the playlist allowlist). The `html` extension is included as a workaround for one specific service. | `[libavformat/hls.c:L2865-L2873]` |
| 5 | `extension_picky` | `AV_OPT_TYPE_BOOL` | `extension_picky` | `1` (enabled) | `0..1` | Strict extension validation: when set, reject any URI whose extension is not in the allowlist. When unset, validation is relaxed (a security regression but useful for legacy streams). | `[libavformat/hls.c:L2874-L2875]` |
| 6 | `max_reload` | `AV_OPT_TYPE_INT` | `max_reload` | `100` | `0..INT_MAX` | Maximum number of times to reload an "insufficient" playlist (playlist that does not yet contain the next expected segment) before giving up. | `[libavformat/hls.c:L2876-L2877]` |
| 7 | `m3u8_hold_counters` | `AV_OPT_TYPE_INT` | `m3u8_hold_counters` | `1000` | `0..INT_MAX` | Maximum number of consecutive identical-content playlist reloads before the demuxer concludes the stream is stalled and returns `AVERROR_EOF`. Protects against server-side stuck playlists. | `[libavformat/hls.c:L2878-L2879]` |
| 8 | `http_persistent` | `AV_OPT_TYPE_BOOL` | `http_persistent` | `1` (enabled) | `0..1` | Reuse the underlying TCP connection across HTTP requests via `ff_http_do_new_request`. NOTE: demuxer default (1) is opposite to the muxer default (0). | `[libavformat/hls.c:L2880-L2881]` |
| 9 | `http_multiple` | `AV_OPT_TYPE_BOOL` | `http_multiple` | `-1` (auto) | `-1..1` | Issue parallel HTTP connections for segment prefetch. `-1` = auto (decide per protocol); `0` = disable; `1` = force. | `[libavformat/hls.c:L2882-L2883]` |
| 10 | `http_seekable` | `AV_OPT_TYPE_BOOL` | `http_seekable` | `-1` (auto) | `-1..1` | Use HTTP byte-range (partial) requests for segments. `-1` = auto, `0` = disable, `1` = enable. Required for byterange-mode HLS playback. | `[libavformat/hls.c:L2884-L2885]` |
| 11 | `seg_format_options` | `AV_OPT_TYPE_DICT` | `seg_format_opts` | `NULL` | n/a | Dictionary of format options forwarded to each segment's underlying demuxer (typically MPEG-TS or MP4). Mirrors the muxer's `hls_segment_options`. | `[libavformat/hls.c:L2886-L2887]` |
| 12 | `seg_max_retry` | `AV_OPT_TYPE_INT` | `seg_max_retry` | `0` (no retry) | `0..INT_MAX` | Maximum number of times to re-attempt a single segment fetch when it fails. | `[libavformat/hls.c:L2888-L2889]` |

---

## Demuxer Outputs: AVPacket Fields Populated by `hls_read_packet`

### Plain-Language Summary

Each call to `hls_read_packet` at `[libavformat/hls.c:L2546-L2707]` returns one `AVPacket` whose fields are derived from the underlying segment demuxer (typically MPEG-TS, but also MP4, AAC, raw H.264, etc.). The HLS layer is mostly a passthrough: it reads packets via `av_read_frame(pls->ctx, pls->pkt)`, optionally decrypts them when Sample-AES is active, optionally fills timestamps for ID3-timestamped streams, then `av_packet_move_ref`s the packet to the caller after remapping `pkt->stream_index` from the per-segment-demuxer stream index to the unified parent stream index.

### Technical Detail

| AVPacket Field | Source / Population Mechanism | Source |
|---|---|---|
| `pkt->stream_index` | Initially set by the segment sub-demuxer; remapped at L2688 via `pkt->stream_index = st->index` so that the parent context's caller sees a unified stream index across segment boundaries (the same logical stream survives even when the sub-demuxer is re-created per segment). | `[libavformat/hls.c:L2688]` |
| `pkt->pts` | Inherited from the segment sub-demuxer. Adjusted for ID3-timestamped streams when `fill_timing_for_id3_timestamped_stream` runs at L2573-L2576 (this is how raw-AAC and similar streams get usable timestamps in HLS). | `[libavformat/hls.c:L2573-L2576, L2687]` |
| `pkt->dts` | Inherited from the segment sub-demuxer (no HLS-side modification). | `[libavformat/hls.c:L2687]` |
| `pkt->duration` | Inherited from the segment sub-demuxer. | `[libavformat/hls.c:L2687]` |
| `pkt->size` | Inherited from the segment sub-demuxer payload size. | `[libavformat/hls.c:L2687]` |
| `pkt->data` | Inherited from the segment sub-demuxer. For Sample-AES segments where the segment demuxer is NOT `mov` (which has its own native sample-encryption support), the payload is decrypted in-place via `ff_hls_senc_decrypt_frame` before being returned. | `[libavformat/hls.c:L2600-L2605, L2687]` |
| `pkt->flags` | Inherited from the segment sub-demuxer (includes `AV_PKT_FLAG_KEY` when the sub-demuxer marks a keyframe). | `[libavformat/hls.c:L2687]` |
| `pkt->buf` (via `av_packet_move_ref`) | Reference-counted ownership transferred via `av_packet_move_ref(pkt, pls->pkt)` at L2687 — the caller assumes ownership of the underlying buffer and the demuxer's `pls->pkt` becomes empty. | `[libavformat/hls.c:L2687]` |
| Codec info propagation (`AVStream->codecpar`) | When the sub-demuxer probes a different codec than the one originally declared in the master playlist's `CODECS=` attribute, `set_stream_info_from_input_stream` is called at L2697-L2702 to update the parent `AVStream->codecpar`, ensuring downstream decoders see the correct codec parameters. | `[libavformat/hls.c:L2697-L2702]` |

The demuxer's main loop is `av_read_frame(pls->ctx, pls->pkt)` calling into the per-segment child demuxer (`pls->ctx`). For subtitle streams a parallel `read_subtitle_packet` path handles WebVTT segment reads. When a segment is exhausted, the demuxer opens the next segment in the playlist and continues; this transparent boundary handling is invisible to callers.



## Encryption Key Input File Format: `hls_key_info_file`

### Plain-Language Summary

When the application wants the muxer to produce AES-128–encrypted HLS segments, the simplest mechanism is to set `-hls_key_info_file <path>` to point at a small plain-text file. That file tells the muxer three things:

1. **What URI to publish in the playlist** so clients know where to fetch the key.
2. **Where on the local filesystem to read the raw 16-byte key bytes** that the muxer will use to encrypt each segment.
3. **(Optional) A 32-hex-character initialization vector (IV)** to use. If omitted, the muxer derives the IV from each segment's sequence number.

The file is opened via `s->io_open` so the URI itself can be a remote location (e.g., a key-management service), although filesystem paths are the most common case. The file is read once at the start of each variant's encryption setup, and re-read on every segment boundary when the `periodic_rekey` flag is enabled, so the key can rotate during streaming.

### Technical Detail

The parser is `hls_encryption_start` at `[libavformat/hlsenc.c:L714-L771]`. The function is called from the segment-start path (`hls_start` at `[libavformat/hlsenc.c:L1675+]`) once per variant per key period.

#### §13.1 — File Format (Line-by-Line)

| Line # | Required? | Field | Maximum Length | Effect | Source |
|---|---|---|---|---|---|
| 1 | Required | Key URI (string, no surrounding quotes) | `LINE_BUFFER_SIZE` (= `MAX_URL_SIZE`) | Stored into `vs->key_uri`. Emitted verbatim in every applicable `#EXT-X-KEY:METHOD=AES-128,URI="<...>"` line. Clients use this URI to GET the raw key. Trailing `\r\n` is stripped via `strcspn`. | `[libavformat/hlsenc.c:L731-L732]` |
| 2 | Required | Filesystem path (or URI) to the raw 16-byte key file | `LINE_BUFFER_SIZE` | Stored into `vs->key_file`. The muxer opens this path via `s->io_open` with `AVIO_FLAG_READ`, reads exactly `KEYSIZE` (= 16) bytes via `avio_read`, then hex-encodes them into `vs->key_string` for forwarding to the underlying TS/fMP4 encryption pipeline. | `[libavformat/hlsenc.c:L734-L735, L753, L760-L768]` |
| 3 | Optional | 32 hex characters (16 binary bytes) of IV | `LINE_BUFFER_SIZE` (only first 32 hex chars used) | Stored into `vs->iv_string`. Used verbatim in the optional `,IV=0x<hex>` suffix of the `#EXT-X-KEY` line and forwarded as the `encryption_iv` dictionary entry to the sub-muxer. If this line is missing or empty, the muxer auto-derives the IV per segment (see §13.3 below). | `[libavformat/hlsenc.c:L737-L738, L1791]` |

#### §13.2 — Validation Rules

| Rule | Behavior on Violation | Source |
|---|---|---|
| Line 1 must be non-empty | Returns `AVERROR(EINVAL)` with log message `"no key URI specified in key info file"` | `[libavformat/hlsenc.c:L742-L745]` |
| Line 2 must be non-empty | Returns `AVERROR(EINVAL)` with log message `"no key file specified in key info file"` | `[libavformat/hlsenc.c:L747-L750]` |
| Line 2's referenced file must read exactly 16 bytes | If `avio_read` returns `!= KEYSIZE` (16), returns `AVERROR(EINVAL)` (or `AVERROR_EOF` if appropriate); short read is fatal | `[libavformat/hlsenc.c:L760-L767]` |
| All three lines have their `\r\n` stripped before use | Via `vs->key_uri[strcspn(vs->key_uri, "\r\n")] = '\0'` and equivalents | `[libavformat/hlsenc.c:L732, L735, L738]` |

#### §13.3 — IV Auto-Derivation (When Line 3 Is Absent)

When `vs->iv_string` is empty after parsing the key-info file, the muxer derives an IV per segment in `hls_start`:

| Condition | IV Derivation | Source |
|---|---|---|
| `vs->iv_string` empty after `hls_encryption_start` | `snprintf(iv_string, sizeof(iv_string), "%032" PRIx64, vs->sequence)` — 32-hex representation of the current segment sequence number, zero-padded; written to `vs->iv_string` | `[libavformat/hlsenc.c:L1795-L1800]` |

Each segment thus gets a distinct IV equal to its zero-padded sequence number. This is the standard HLS interoperability convention (see RFC 8216 §5.2: "If the IV attribute is not present, the IV value is the Media Sequence Number of the Media Segment").

#### §13.4 — Alternative: Inline Encryption via `hls_enc` (No File)

When `-hls_enc 1` is set without `-hls_key_info_file`, the muxer takes a parallel "inline" path in `do_encrypt` (a function near `[libavformat/hlsenc.c:L640-L710]`):

| Behavior | Source |
|---|---|
| Key: read from `-hls_enc_key` if set; otherwise generated via `av_random_bytes(key, KEYSIZE)` | `[libavformat/hlsenc.c:L691-L700]` |
| Key URI in playlist: from `-hls_enc_key_url` option (`hls->key_url`) at L660 | `[libavformat/hlsenc.c:L658-L660]` |
| Key file location: derived from `hls->key_basename` at L662 | `[libavformat/hlsenc.c:L661-L664]` |
| IV: from `-hls_enc_iv` if set (`hls->iv` raw bytes via `memcpy(iv, hls->iv, sizeof(iv))`); otherwise zero-filled then `AV_WB64(iv + 8, vs->sequence)` — first 8 bytes zero, last 8 bytes big-endian sequence | `[libavformat/hlsenc.c:L666-L676]` |
| Generated key bytes are written to disk via `avio_write(pb, key, KEYSIZE)` at the path under `hls->key_file` | `[libavformat/hlsenc.c:L702-L708]` |

#### §13.5 — Periodic Re-Key Reload

When the `periodic_rekey` flag (`HLS_PERIODIC_REKEY`) is set in `-hls_flags`, the muxer re-reads `hls_key_info_file` at every segment boundary, allowing the deployer to swap the file's contents on disk between segments to rotate keys mid-stream without restarting the encode. The reload is bandwidth-bounded by `LINE_BUFFER_SIZE` per line (the maximum URL size).

#### §13.6 — Example File Contents

A typical 3-line key-info file (the third line is optional):

```text
https://keys.example.com/keys/stream.key
/var/secrets/stream-aes128.key
0123456789abcdef0123456789abcdef
```

A 2-line key-info file (IV auto-derived from sequence number) — equivalent except clients compute IV from the playlist's `#EXT-X-MEDIA-SEQUENCE` arithmetic per RFC 8216 §5.2:

```text
https://keys.example.com/keys/stream.key
/var/secrets/stream-aes128.key
```

In both cases, the muxer reads `/var/secrets/stream-aes128.key` (16 raw bytes), encrypts each segment with AES-128-CBC, and emits a `#EXT-X-KEY:METHOD=AES-128,URI="https://keys.example.com/keys/stream.key"` line at the top of the playlist (plus `,IV=0x...` if line 3 is present and the playlist version is ≥ 2).

---

## Document End

This file completes the I/O inventory for the FFmpeg HLS muxer and demuxer at commit `566ad786`. Every option, every const alias, every tag emission, every AVPacket field consumed and produced, and every M3U8 line type parsed has been enumerated individually per the No-Summarizing Rule (AAP §0.10.9).

Cross-references:

- [`./functional-inventory.md`](./functional-inventory.md) — describes the components that consume these inputs and produce these outputs.
- [`./exception-handling.md`](./exception-handling.md) — describes the failure modes triggered by malformed inputs.
- [`./consumer-dependencies.md`](./consumer-dependencies.md) — describes what downstream systems depend on these outputs.
- [`../technical/codec-logic.md`](../technical/codec-logic.md) — decision tables governing how the inputs are interpreted.
- [`../technical/data-model.md`](../technical/data-model.md) — struct fields that back each option.
- [`../api-contracts/data-contracts.md`](../api-contracts/data-contracts.md) — formal type/bounds contract for each input.
- [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md) — wire-format contracts for outputs (M3U8 lines, segment file layout, HTTP requests).

