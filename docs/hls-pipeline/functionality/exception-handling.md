# Exception Handling — HLS Pipeline Failure Catalog

This document maps every observed failure mode in the FFmpeg HLS muxer (`libavformat/hlsenc.c`) and demuxer (`libavformat/hls.c`) to its `AVERROR(*)` return code and recovery path. All source references in this document are anchored to commit `566ad786`. See [`../README.md`](../README.md) for the canonical commit-anchor banner and citation conventions.

---

## Overview

### Plain-Language Summary

HLS pipeline failure modes fall into roughly nine families: bad input packet, missing keyframe (silent), I/O failures opening or writing files, mid-stream discontinuities, clock discontinuities, segment-write failures, HTTP DELETE failures during sliding-window cleanup, demuxer broken-playlist retry exhaustion, and demuxer key-fetch failure. Each scenario maps to one or more `AVERROR(*)` codes returned from a public-API entry point — `avformat_write_header`, `av_write_frame`, `av_interleaved_write_frame`, `av_write_trailer` on the muxer side; `av_read_frame`, `avformat_open_input`, `av_seek_frame` on the demuxer side. A negative return from any of these functions is the integrator's signal that one of the scenarios below has occurred.

A single muxer option, `ignore_io_errors`, gates whether transient I/O failures propagate to the caller or are silently dropped. With the option off (its default of `0`), every I/O error short-circuits packet processing and propagates upward. With the option on, the muxer logs a warning and continues to the next packet. The option is declared as the `ignore_io_errors` AVOption at `[libavformat/hlsenc.c:L3178]` and is the single most important toggle for distinguishing "fail fast" muxing from "best effort" long-running network output.

### Cross-Document References

| For | See |
|-----|-----|
| Decision-table rendering of `ignore_io_errors` and `HLS_DELETE_SEGMENTS` branches | [`../technical/codec-logic.md`](../technical/codec-logic.md) |
| HTTP DELETE wire-format contract for sliding-window cleanup | [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md) |
| AVERROR convention itself (negative-errno return) | [`../README.md`](../README.md) glossary |
| Field semantics for `vs->discontinuity`, `vs->key_uri`, `vs->iv_string`, etc. | [`../technical/data-model.md`](../technical/data-model.md) |

---

### Scenario: Bad Input Packet

#### Plain-Language Summary

The muxer rejects the call when the inputs it receives are malformed or unrecoverable: an out-of-memory condition in a helper buffer, a zero-length `strftime` expansion of a filename template, an empty `%v` variant-name substitution, or a key info file with an empty URL or empty key path. The muxer returns `AVERROR(EINVAL)` for "malformed input value" and `AVERROR(ENOMEM)` for "could not allocate working memory"; the caller treats either as a fatal error for the current muxing session.

#### Technical Detail

- `[libavformat/hlsenc.c:L278]` — `strftime_expand` returns `AVERROR(ENOMEM)` when the working buffer allocation fails.
- `[libavformat/hlsenc.c:L285]` — `strftime_expand` returns `AVERROR(EINVAL)` when `strftime` produces a zero-length string from the user-supplied format.
- `[libavformat/hlsenc.c:L582]` and `[libavformat/hlsenc.c:L587]` — inside `hls_delete_old_segments`, `replace_int_data_in_filename` and `replace_str_data_in_filename` failures on `%v` substitution set `ret = AVERROR(EINVAL)` before jumping to the cleanup label.
- `[libavformat/hlsenc.c:L681]` and `[libavformat/hlsenc.c:L686]` — `do_encrypt` returns `AVERROR(EINVAL)` when `hls->key_uri` or `hls->key_file` is empty after parsing.
- `[libavformat/hlsenc.c:L744]` and `[libavformat/hlsenc.c:L749]` — `hls_encryption_start` returns `AVERROR(EINVAL)` when `vs->key_uri` or `vs->key_file` is empty after reading from the key info file.
- `[libavformat/hlsenc.c:L1180]` and `[libavformat/hlsenc.c:L1231]` — `parse_playlist` (muxer side, `HLS_APPEND_LIST` reload) sets `ret = AVERROR_INVALIDDATA` when a previously-written playlist line cannot be parsed (e.g., a malformed `#EXTINF` or a `#EXT-X-PROGRAM-DATE-TIME` whose `sscanf` field count is wrong).

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| Working buffer `av_mallocz` returns NULL | `AVERROR(ENOMEM)` | `[libavformat/hlsenc.c:L278]` |
| `strftime` produces zero-length string | `AVERROR(EINVAL)` | `[libavformat/hlsenc.c:L285]` |
| `%v` substitution fails (no `varname`, sub returns < 1) | `AVERROR(EINVAL)` | `[libavformat/hlsenc.c:L582]`, `[libavformat/hlsenc.c:L587]` |
| Key info file: empty URL or empty key path | `AVERROR(EINVAL)` | `[libavformat/hlsenc.c:L681]`, `[libavformat/hlsenc.c:L686]`, `[libavformat/hlsenc.c:L744]`, `[libavformat/hlsenc.c:L749]` |
| Reloaded playlist contains malformed line | `AVERROR_INVALIDDATA` | `[libavformat/hlsenc.c:L1180]`, `[libavformat/hlsenc.c:L1231]` |

#### Recovery Sketch

The caller checks the return code of `avformat_write_header`, `av_write_frame`, `av_interleaved_write_frame`, or `av_write_trailer`. On a negative return, the standard FFmpeg cleanup path applies: free the format context with `avformat_free_context` (or `avio_closep` first if the caller owned the I/O context), then either abort with a user-facing error or retry the session with corrected inputs. None of these errors is internally recoverable — the muxer does not attempt to repair its input.

---

### Scenario: Missing Keyframe (Silent)

#### Plain-Language Summary

When the muxer would normally cut a new segment at a video keyframe but the current packet is not a keyframe (and the `HLS_SPLIT_BY_TIME` flag is not set), the cut is silently skipped. No `AVERROR` is returned. The current segment grows past its target duration until the next keyframe arrives. This is intentional behaviour, not a defect — the HLS specification requires every segment to begin with a keyframe, so the muxer waits rather than emit a segment that no client could decode from its start. Integrators must be aware of it because the only visible effect is that the published `#EXTINF` durations exceed `hls_time`; there is no error code to catch.

#### Technical Detail

- `[libavformat/hlsenc.c:L2473-L2475]` — the `can_split` flag is computed inside the `vs->has_video` branch as `(codec_type == VIDEO) && ((pkt->flags & AV_PKT_FLAG_KEY) || (hls->flags & HLS_SPLIT_BY_TIME))`. For a non-keyframe video packet with `HLS_SPLIT_BY_TIME` unset, `can_split` is `0`.
- `[libavformat/hlsenc.c:L2478-L2479]` — when `pkt->pts == AV_NOPTS_VALUE`, both `is_ref_pkt` and `can_split` are forced to `0`. A stream that never carries a presentation timestamp is therefore never a candidate for cutting.
- `[libavformat/hlsenc.c:L2500]` — the line `can_split = can_split && (pkt->pts - vs->end_pts > 0);` further suppresses cutting when the next packet's timestamp has not advanced past the previous segment boundary.
- `[libavformat/hlsenc.c:L2501-L2502]` — the segment-cut check `if (vs->packets_written && can_split && av_compare_ts(...))` short-circuits when `can_split` is false; control falls through to the packet-append path without writing or rotating a segment.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| Video packet with no `AV_PKT_FLAG_KEY` and `HLS_SPLIT_BY_TIME` not set | none (silent) | `[libavformat/hlsenc.c:L2473-L2475]` |
| `pkt->pts == AV_NOPTS_VALUE` on any packet | none (silent) | `[libavformat/hlsenc.c:L2478-L2479]` |
| Duplicate-PTS keyframe (`pkt->pts == vs->end_pts`) | none (silent) | `[libavformat/hlsenc.c:L2500]` |

#### Recovery Sketch

The integrator ensures the upstream video encoder emits keyframes at intervals not exceeding `hls_time` (default 2 seconds, declared at `[libavformat/hlsenc.c:L3123]`). Common practice with `libx264` is to set `-g <fps * hls_time>` plus `-keyint_min <same>` so every GOP boundary is a candidate segment boundary. For codecs where every frame is independently decodable (most audio codecs, audio-only HLS), the scenario does not arise — `can_split` is gated on `vs->has_video` at `[libavformat/hlsenc.c:L2473]` and is always `1` on an audio-only variant. Setting `HLS_SPLIT_BY_TIME` (`-hls_flags +split_by_time`) opts into time-based cuts at non-keyframe boundaries, at the cost of producing segments that downstream clients cannot start playback from cleanly.

---

### Scenario: I/O Failures (File Open / Write / Close)

#### Plain-Language Summary

Opening a segment file, writing bytes into it, or closing it can fail for any of the reasons the underlying protocol can fail: disk full, network unavailable, permission denied, HTTP 5xx, socket reset, or a stale persistent connection that the server has just torn down. The muxer captures the underlying `AVERROR(*)` code from the protocol layer and, by default, returns it to the caller. With the `ignore_io_errors` option turned on, the muxer logs a warning and continues to the next packet so that long-running network output survives transient failures. The option is documented at `[libavformat/hlsenc.c:L3178]`.

#### Technical Detail

- `[libavformat/hlsenc.c:L292-L311]` — `hlsenc_io_open` wraps `s->io_open` (file, HTTP, or any registered protocol). The initial sentinel `err = AVERROR_MUXER_NOT_FOUND;` at `[libavformat/hlsenc.c:L297]` is only ever observed by the caller if no code path overwrites it — in normal use it is overwritten on the very next line by `s->io_open(...)` returning a real protocol error code (e.g., `AVERROR(ENOENT)`, `AVERROR(EACCES)`, an HTTP-status-derived `AVERROR(EIO)`).
- `[libavformat/hlsenc.c:L2571-L2578]` — the segment-file open path inside `hls_write_packet` includes the explicit short-circuit `return hls->ignore_io_errors ? 0 : ret;` after logging at either `AV_LOG_WARNING` or `AV_LOG_ERROR` level depending on the flag.
- `[libavformat/hlsenc.c:L2589-L2599]` — the segment-file close path retries once with a fresh `hlsenc_io_open` call on `hlsenc_io_close` failure (the log line is `"upload segment failed, will retry with a new http session.\n"`). The retry uses `reflush_dynbuf` to re-prime the buffer if the close fails. A second failure is not retried; control falls through to the next branch and the error propagates.
- `[libavformat/hlsenc.c:L2628-L2635]` — the playlist publish path retries `hls_window` once on failure with the same single-retry pattern. The log line is `"upload playlist failed, will retry with a new http session.\n"`.
- `[libavformat/hlsenc.c:L2686-L2687]` — after `ff_write_chained` writes the packet through the sub-muxer, the conditional `if (hls->ignore_io_errors) ret = 0;` masks any propagated I/O error to `0`. The packet is considered "successfully published" from `av_write_frame`'s perspective even if the underlying byte never reached the storage backend.

#### Failure Trigger Table

| Trigger | AVERROR | Site | Masked by `ignore_io_errors`? |
|---|---|---|---|
| Segment-file `s->io_open` fails | underlying protocol code | `[libavformat/hlsenc.c:L2571-L2578]` | Yes — returned as `0` |
| Segment-file `hlsenc_io_close` fails after one retry | underlying protocol code | `[libavformat/hlsenc.c:L2589-L2599]` | No — propagates |
| Playlist `hls_window` write fails after one retry | underlying protocol code | `[libavformat/hlsenc.c:L2628-L2635]` | No — propagates |
| Sub-muxer `ff_write_chained` fails | underlying protocol code | `[libavformat/hlsenc.c:L2686-L2687]` | Yes — masked to `0` |
| HTTP persistent reuse via `ff_http_do_new_request` fails | HTTP-mapped `AVERROR(EIO)` | `[libavformat/hlsenc.c:L304]` | inherits the open-path policy above |

#### Recovery Sketch

- **Strict mode** (`ignore_io_errors=0`, default): the caller receives the AVERROR from `av_write_frame` or `av_interleaved_write_frame`, calls `av_write_trailer` (which itself may fail again but releases internal state), then `avformat_free_context`, and either aborts or restarts the entire muxing session. The session cannot be partially recovered: the segment counter, the open sub-muxer file handles, and the per-variant in-memory segment list are all in an indeterminate state after a propagated I/O error.
- **Tolerant mode** (`ignore_io_errors=1`): the muxer drops the affected segment or playlist write silently and continues with the next packet. Suitable for live, long-running network output where transient network errors are expected and where dropping a segment is preferable to terminating the entire stream. Integrators choosing this mode must monitor the application log for `AV_LOG_WARNING` lines beginning with `"Failed to open file"` or `"upload segment failed"` so that prolonged outages are still detectable out of band.

---

### Scenario: Mid-Stream Discontinuities

#### Plain-Language Summary

When the input stream contains a content discontinuity — a deliberate timestamp gap, a source change between two concatenated programs, a codec-parameter change such as a resolution switch, or the muxer restarting against an existing playlist — the muxer sets a per-variant flag that causes the next segment's playlist entry to be preceded by an `#EXT-X-DISCONTINUITY` line. This is not an error condition; no `AVERROR(*)` is returned. The tag is part of the HLS protocol and tells downstream players to reset their decoder state at that segment boundary.

#### Technical Detail

- `[libavformat/hlsenc.c:L1090-L1093]` — inside `hls_append_segment`, when `vs->discontinuity` is set the new segment record is flagged `en->discont = 1` and the per-variant flag is cleared so the discontinuity is consumed once and never repeated on subsequent segments.
- `[libavformat/hlsenc.c:L3100-L3102]` — when the `HLS_APPEND_LIST` flag is set (muxer reloading and appending to a previously written playlist), `vs->discontinuity = 1` is set during initialization to mark the boundary between previously-published segments and the new ones the muxer is about to produce.
- `[libavformat/hlsplaylist.c:L156-L158]` — `ff_hls_write_file_entry` emits `#EXT-X-DISCONTINUITY\n` to the playlist immediately before the affected segment line when its `insert_discont` parameter is non-zero. The muxer passes `en->discont` for that parameter at `[libavformat/hlsenc.c:L1616]`.
- `[libavformat/hlsenc.c:L1593-L1596]` — a separate "discontinuity at the start of the playlist" emission is gated by the `HLS_DISCONT_START` flag and the `vs->discontinuity_set` guard, producing one `#EXT-X-DISCONTINUITY` line at the top of the segment list rather than between segments.
- `[libavformat/hls.c:L2904]` — the demuxer registers itself with `AVFMT_TS_DISCONT` set on `.p.flags`, which informs the libavformat layer that timestamps may discontinuously jump across segment boundaries. The demuxer does not parse `#EXT-X-DISCONTINUITY` lines explicitly; sub-demuxer timestamp resets and the flag together cover the playback path.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| `HLS_APPEND_LIST` startup sets `vs->discontinuity = 1` | none (informational) | `[libavformat/hlsenc.c:L3100-L3102]` |
| Application code explicitly sets `vs->discontinuity = 1` between packets [inferred — no direct source for the application-side hook] | none (informational) | `[libavformat/hlsenc.c:L1090-L1093]` (propagation site) |
| `HLS_DISCONT_START` flag enables top-of-playlist discontinuity | none (informational) | `[libavformat/hlsenc.c:L1593-L1596]` |

#### Recovery Sketch

No recovery is needed. Discontinuity is a published signal: downstream HLS-compliant players are required by RFC 8216 §4.4.4.3 to reset their decoder state at the discontinuity boundary. The muxer's only obligation is that the `#EXT-X-DISCONTINUITY` line precedes the affected segment entry in the playlist; that ordering is enforced structurally by `ff_hls_write_file_entry` writing the discontinuity tag first, then `#EXTINF`, then the segment URL.

---

### Scenario: Clock Discontinuities (PROGRAM-DATE-TIME Recompute)

#### Plain-Language Summary

When the muxer reloads an existing playlist on startup (the `HLS_APPEND_LIST` flag) and that playlist contains `#EXT-X-PROGRAM-DATE-TIME` lines, the wall-clock anchor is read from the most recent such line in the existing playlist and rolled forward by each parsed segment's duration so that the newly produced segments inherit the correct continued wall-clock timeline. The mechanism guarantees that a player observing the reloaded stream sees a single contiguous time axis even though the muxer restarted partway through. No error is raised on either the reload path or the emission path.

#### Technical Detail

- `[libavformat/hlsenc.c:L1226-L1244]` — inside `parse_playlist`, the muxer encounters `#EXT-X-PROGRAM-DATE-TIME:%d-%d-%dT%d:%d:%d.%lf` lines and parses them with `sscanf` into a `struct tm`, then computes the local-time anchor via `mktime`. A malformed line (wrong field count) sets `ret = AVERROR_INVALIDDATA` at `[libavformat/hlsenc.c:L1231]`.
- `[libavformat/hlsenc.c:L1273-L1275]` — for each subsequent segment line parsed from the existing playlist, the rolling anchor `discont_program_date_time` is written onto the segment's `discont_program_date_time` field, then advanced by that segment's duration so the next segment inherits the post-advance value.
- `[libavformat/hlsenc.c:L1620-L1623]` — on the emission side inside `hls_window`, `ff_hls_write_file_entry` is passed either `&en->discont_program_date_time` (when set on this specific segment) or the rolling `prog_date_time_p` pointer. After emission, segment-specific values are rewound by `en->duration` so the next read of the field represents the START time of the next emission.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| Reloaded playlist's `#EXT-X-PROGRAM-DATE-TIME` line has wrong field count | `AVERROR_INVALIDDATA` | `[libavformat/hlsenc.c:L1231]` |
| `mktime` reports system error (out-of-range date) [inferred — `mktime` may return `(time_t)-1` but the muxer does not check] | none (silent) | `[libavformat/hlsenc.c:L1243]` |

#### Recovery Sketch

The `AVERROR_INVALIDDATA` returned by a malformed reloaded playlist propagates from `hls_write_header` to the caller. The caller corrects the existing playlist or removes it so the muxer starts fresh on the next attempt. No mid-stream recovery is performed by the muxer.

---

### Scenario: Segment Write Failures

#### Plain-Language Summary

Writing a single segment to its final destination involves several stages: flushing the sub-muxer's dynamic buffer into the segment file's `AVIOContext`, optionally writing the fMP4 `styp` box and initialization data, completing the write through `hlsenc_io_close`, and (when `HLS_TEMP_FILE` is set) renaming `<segment>.tmp` to its final filename. A failure at any of these stages returns a negative `AVERROR(*)`. The HTTP-upload close path includes a single automatic retry with a fresh session; all other failures propagate unless `ignore_io_errors` masks them as described in the I/O Failures section.

#### Technical Detail

- `[libavformat/hlsenc.c:L2582-L2587]` — `flush_dynbuf` propagates the underlying `ret` when the dynamic-buffer-to-file copy fails; the call site frees the temporary filename and dictionary before returning. The flush itself is a wrapper around `avio_write` and `avio_close_dyn_buf` against the sub-muxer's in-memory accumulator.
- `[libavformat/hlsenc.c:L2589-L2599]` — `hlsenc_io_close` failure triggers a one-shot retry: `ff_format_io_close(s, &vs->out);` discards the failed I/O context, `hlsenc_io_open` is called again to establish a fresh session, `reflush_dynbuf` re-primes the buffer, and `hlsenc_io_close` is invoked once more. A second failure is not retried; control returns to the caller with the underlying error.
- `[libavformat/hlsenc.c:L1300-L1313]` — `hls_rename_temp_file` performs the temp-to-final rename. It allocates `final_filename` with `av_strdup` (returning `AVERROR(ENOMEM)` on allocation failure at `[libavformat/hlsenc.c:L1307]`), then calls `ff_rename`. A rename failure leaves a `<segment>.tmp` file on disk; the returned error code is the underlying filesystem error.
- `[libavformat/hlsenc.c:L2605-L2606]` — `hls_rename_temp_file` is invoked only when `use_temp_file` is set in the current call context; `use_temp_file` is in turn derived from the `HLS_TEMP_FILE` flag at `[libavformat/hlsenc.c:L1542]` for playlist writes and at the equivalent local guard for segment writes.
- `[libavformat/hlsenc.c:L2511-L2515]` — for `SEGMENT_TYPE_FMP4` segments, the first segment requires writing the initialization segment via `avio_close_dyn_buf`; the call returns `AVERROR(EINVAL)` when the dynamic buffer reports zero or negative length.
- `[libavformat/hlsenc.c:L2613-L2615]` — `av_strdup(oc->url)` failure returns `AVERROR(ENOMEM)` before the playlist publish step begins.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| `flush_dynbuf` copy fails (underlying I/O error) | propagated | `[libavformat/hlsenc.c:L2582-L2587]` |
| `hlsenc_io_close` fails on first attempt | retried, then propagated | `[libavformat/hlsenc.c:L2589-L2599]` |
| `av_strdup` allocates filename copy and returns NULL | `AVERROR(ENOMEM)` | `[libavformat/hlsenc.c:L1307]`, `[libavformat/hlsenc.c:L2613-L2615]` |
| `ff_rename` returns negative (filesystem error) | propagated | `[libavformat/hlsenc.c:L1309]` |
| fMP4 init buffer flush returns ≤ 0 length | `AVERROR(EINVAL)` | `[libavformat/hlsenc.c:L2514-L2515]` |

#### Recovery Sketch

On rename failure, the application may re-attempt the rename out-of-band; the muxer treats the segment as published and continues with the next one, so downstream players will not see the segment until the rename succeeds. On `flush_dynbuf` failure, the segment is partially written on disk and there is no automatic cleanup — the integrator may want to delete the stub file before restarting. On allocation failures (`ENOMEM`), recovery is process-level (the host is under memory pressure).

---

### Scenario: HTTP DELETE Failures (Sliding-Window Cleanup)

#### Plain-Language Summary

When the `HLS_DELETE_SEGMENTS` flag is set and the sliding window evicts a segment from the playlist, the muxer issues an HTTP `DELETE` request (for HTTP output) or a filesystem `unlink` (for file output) on the evicted segment URL. The request uses a dedicated `http_delete` `AVIOContext` field on the muxer's private data. A DELETE failure is logged at `AV_LOG_ERROR` level and the segment record is freed regardless — there is no retry, and the muxer does not return an `AVERROR(*)` to the caller. Orphaned segment files on the storage backend accumulate; periodic out-of-band cleanup is the integrator's responsibility.

#### Technical Detail

- `[libavformat/hlsenc.c:L261]` — `AVIOContext *http_delete;` field on `HLSContext`. It is a dedicated `AVIOContext` so that the DELETE method override does not interfere with the regular PUT path used for segment uploads.
- `[libavformat/hlsenc.c:L525-L527]` — the log path `"failed to delete old segment %s: %s\n"` inside `hls_delete_file` records the failed unlink/DELETE; the function then returns `0` so the caller's iteration continues unchanged.
- `[libavformat/hlsenc.c:L531-L639]` — `hls_delete_old_segments` walks the `vs->old_segments` linked list. The internal `goto fail;` paths inside the loop are taken only for `av_bprint` overflow (at `[libavformat/hlsenc.c:L603-L605]`) and for `hls_delete_file` returning a non-zero error code from `[libavformat/hlsenc.c:L608]` — but `hls_delete_file` is constructed to swallow unlink failures and return `0`, so in practice the file-delete error itself never propagates.
- `[libavformat/hlsenc.c:L628-L630]` — each old-segment record is freed unconditionally as the loop advances, irrespective of whether its underlying file delete succeeded.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| HTTP DELETE returns non-2xx, file `unlink` returns -1 | none (logged, swallowed) | `[libavformat/hlsenc.c:L525-L527]` |
| `av_bprint` overflow constructing the segment path | `AVERROR(ENOMEM)` | `[libavformat/hlsenc.c:L603-L605]` |

#### Recovery Sketch

Orphaned segment files accumulate on the CDN or origin server; periodic out-of-band cleanup (a cron job listing segment files older than the publish window) is required. The integrator may also override the `method` option to a different HTTP verb if the origin requires it. This is documented as a known limitation, not a defect — the design choice trades a strict cleanup guarantee for resilience against transient delete failures interrupting an otherwise healthy publish.

---

### Scenario: Demuxer Broken Playlist (Stale Manifest Retry)

#### Plain-Language Summary

When the HLS demuxer reloads a live playlist and finds the manifest's `#EXT-X-MEDIA-SEQUENCE` has not advanced (no new segments since the last fetch), the demuxer increments a per-playlist hold counter. After a configurable number of consecutive identical reloads — default 1000, declared at `[libavformat/hls.c:L2878-L2879]` — the demuxer returns `AVERROR_EOF` from `av_read_frame` to signal end-of-stream. The mechanism gives a publisher time to recover from a temporary stall before the player gives up.

#### Technical Detail

- `[libavformat/hls.c:L1624-L1631]` — the retry-counter loop inside the reload path: `v->m3u8_hold_counters++` when the latest reload's last sequence number matches the previous reload's, and `return AVERROR_EOF;` when the counter reaches `c->m3u8_hold_counters`.
- `[libavformat/hls.c:L2878-L2879]` — the `m3u8_hold_counters` AVOption declaration: `OFFSET(m3u8_hold_counters), AV_OPT_TYPE_INT, {.i64 = 1000}`. The default of 1000 multiplied by the half-target-duration polling cadence gives roughly 1000 × `target_duration/2` of patience before declaring EOF.
- `[libavformat/hls.c:L2193]` — the counter is reset to `0` on a successful new-segment discovery during initial parse, so a stream that stalls and then resumes does not carry stale counter state.
- `[libavformat/hls.c:L1635-L1640]` — the inner loop also handles the case where the playlist has run out of available segments: if the playlist is `finished` or is a subtitle stream, `AVERROR_EOF` is returned immediately; otherwise the loop waits for `reload_interval` and re-fetches.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| `m3u8_hold_counters` consecutive identical reloads | `AVERROR_EOF` | `[libavformat/hls.c:L1629-L1630]` |
| Playlist is marked `finished` and the demuxer has exhausted segments | `AVERROR_EOF` | `[libavformat/hls.c:L1635-L1637]` |
| Interrupt callback fires during reload wait | `AVERROR_EXIT` | `[libavformat/hls.c:L1640]` |

#### Recovery Sketch

The caller treats `AVERROR_EOF` as graceful end-of-stream and closes the demuxer with `avformat_close_input`. Live-streaming applications that need faster EOF detection on a stalled feed can lower `m3u8_hold_counters` (e.g., to `10`) via `av_dict_set` on the options dictionary passed to `avformat_open_input`. The trade-off is that the lower counter declares EOF on shorter publisher stalls; integrators should choose the value based on the expected worst-case publisher recovery time.

---

### Scenario: Demuxer Key Fetch Failure

#### Plain-Language Summary

When the HLS demuxer encounters an `#EXT-X-KEY:METHOD=AES-128,URI=...` line in a playlist, it materializes the absolute key URL onto each affected segment record so that the segment-fetch path can fetch the 16-byte key from the URI as part of opening the segment. A failed fetch (HTTP 404, HTTP 5xx, connection timeout, DNS failure, etc.) propagates as the underlying protocol's `AVERROR(*)` code — typically `AVERROR(EIO)` or `AVERROR_PROTOCOL_NOT_FOUND` if the required protocol is not built into the binary. The demuxer itself does not retry key fetches.

#### Technical Detail

- `[libavformat/hls.c:L867-L880]` — inside `parse_playlist`, the `#EXT-X-KEY:` branch captures the `METHOD`, `URI`, and optional `IV` via `handle_key_args`. Recognized methods are `AES-128` (setting `key_type = KEY_AES_128`) and `SAMPLE-AES` (`key_type = KEY_SAMPLE_AES`). The local `key[]` buffer holds the URI string for later attachment to each segment.
- `[libavformat/hls.c:L1017-L1029]` — when a segment line is parsed under an active `key_type != KEY_NONE`, the demuxer calls `ff_make_absolute_url` to resolve the URI against the playlist URL and `av_strdup` to attach the result to the segment record. `av_strdup` failure returns `AVERROR(ENOMEM)` at `[libavformat/hls.c:L1027]`; `ff_make_absolute_url` producing an empty string returns `AVERROR_INVALIDDATA` at `[libavformat/hls.c:L1020]`.
- `[libavformat/hls.c:L639]` — `open_url_keepalive` returns `AVERROR_PROTOCOL_NOT_FOUND` when the binary is built without HTTP protocol support, which is the failure shape an integrator sees when attempting to fetch an `https://` key URI from a binary missing TLS.
- `[libavformat/hls.c:L1800]` — `AVERROR(EPERM)` is returned from the security guard that refuses to open an external (non-`hls`) playlist or key path unless the application has explicitly allowed it.

#### Failure Trigger Table

| Trigger | AVERROR | Site |
|---|---|---|
| `ff_make_absolute_url` produces empty string | `AVERROR_INVALIDDATA` | `[libavformat/hls.c:L1020]` |
| `av_strdup` on key URL returns NULL | `AVERROR(ENOMEM)` | `[libavformat/hls.c:L1027]` |
| HTTP/TLS protocol not built in | `AVERROR_PROTOCOL_NOT_FOUND` | `[libavformat/hls.c:L639]` |
| External-file open denied for security | `AVERROR(EPERM)` | `[libavformat/hls.c:L1800]` |
| Underlying key fetch (HTTP 4xx/5xx, timeout) | `AVERROR(EIO)` or protocol-specific | propagated from the protocol layer |

#### Recovery Sketch

Integrator-side recovery typically involves retrying the fetch with backoff at the application level above libavformat. The HLS demuxer itself does not retry key fetches — the operation is one-shot per segment open. For production playback against an authenticated key endpoint, a wrapping HTTP client or short-lived caching proxy is the conventional pattern; libavformat does not provide a built-in key cache or retry policy.

---

## AVERROR Code Reference

### Plain-Language Summary

The table below has one row per distinct `AVERROR(*)` code returned by `libavformat/hlsenc.c` or `libavformat/hls.c` at commit `566ad786`. Use it as a quick lookup when an integrator sees a negative return code from an HLS-pipeline API call. The "Source" column identifies which source file emits the code; the "Scenario" column references the matching section above; the "Recovery" column states the caller's expected action. The set of codes is exhaustive — any negative return from a public-API call on the HLS muxer or demuxer at commit `566ad786` maps to exactly one of these twelve rows.

### Reference Table

| AVERROR Code | Source | Scenario | Recovery |
|---|---|---|---|
| `AVERROR(EINVAL)` | `hlsenc.c` | Bad input packet / key info file malformed / fMP4 init buffer empty | Caller validates inputs and aborts the session |
| `AVERROR(ENOMEM)` | `hlsenc.c`, `hls.c` | Memory allocation failure in any helper buffer | Caller aborts; long-running processes investigate memory pressure |
| `AVERROR_EOF` | `hlsenc.c`, `hls.c` | End of input stream (muxer) or stale-playlist retry exhaustion (demuxer) | Caller treats as graceful end and closes the format context |
| `AVERROR_INVALIDDATA` | `hlsenc.c`, `hls.c` | Malformed line in a reloaded or fetched M3U8 (`#EXTINF`, `#EXT-X-PROGRAM-DATE-TIME`, `#EXT-X-TARGETDURATION`, `#EXT-X-BYTERANGE`, key URL resolution) | Caller aborts; manifest fix is required |
| `AVERROR_MUXER_NOT_FOUND` | `hlsenc.c` | Initial sentinel returned by `hlsenc_io_open` when the function returns without overriding it (also returned by sub-muxer lookup at `[libavformat/hlsenc.c:L2991]`) | Internal sentinel — rarely user-visible; if seen, indicates the sub-muxer for `hls_segment_type` is not built into the binary |
| `AVERROR_PATCHWELCOME` | `hlsenc.c` | Unsupported configuration the maintainers have not implemented (`[libavformat/hlsenc.c:L853]`, `[libavformat/hlsenc.c:L1771]`) | Caller chooses a different configuration |
| `AVERROR(EIO)` | `hls.c` | Underlying I/O failure during seek (`[libavformat/hls.c:L2733]`, `[libavformat/hls.c:L2749]`) or HTTP fetch | Caller may retry at the application level |
| `AVERROR(ENOSYS)` | `hls.c` | Operation not supported — byte seek requested or input is unseekable (`[libavformat/hls.c:L2720]`) | Caller chooses a different operation |
| `AVERROR(EPERM)` | `hls.c` | External-file open denied for security (`[libavformat/hls.c:L1800]`) | Caller fixes filesystem ACLs, HTTP auth, or the `allowed_extensions` option |
| `AVERROR_BUG` | `hls.c` | Internal-consistency violation in the demuxer — stream index inconsistency between sub-demuxer and main-stream tables (`[libavformat/hls.c:L2681]`) | Caller files an upstream bug report; the condition is unrecoverable |
| `AVERROR_EXIT` | `hls.c` | Interrupt callback signaled application shutdown during a blocking wait (`[libavformat/hls.c:L1640]`, `[libavformat/hls.c:L1691]`) | Caller drains pending state and exits cleanly |
| `AVERROR_PROTOCOL_NOT_FOUND` | `hls.c` | A required protocol (`http`, `https`, `file`, `crypto`) is not built into the FFmpeg binary (`[libavformat/hls.c:L639]`) | Caller links a binary that includes the missing protocol |

### Notes on Use

- The muxer's tolerant mode (`ignore_io_errors=1` at `[libavformat/hlsenc.c:L3178]`) masks several of the I/O-derived rows above to `0`, as detailed in the I/O Failures section. The remaining rows propagate regardless of the option.
- `AVERROR_UNKNOWN` is emitted by `ff_hls_write_file_entry` in `libavformat/hlsplaylist.c` when `strftime` fails inside the `EXT-X-PROGRAM-DATE-TIME` emission path; the muxer's call sites log a warning and continue, so the code does not propagate to the caller through the HLS muxer's public surface and is therefore excluded from this catalog [inferred — no direct propagation observed].
- Codes returned from the sub-muxer (MPEG-TS via `libavformat/mpegtsenc.c` or fMP4 via the MOV muxer) propagate through `ff_write_chained` at `[libavformat/hlsenc.c:L2679]` and surface to the caller as whatever the sub-muxer returned. Such codes are the sub-muxer's responsibility and are outside the scope of this catalog.
