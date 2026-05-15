# Codec and Format Decision Logic — Exhaustive Branch Reference

> **Commit Anchor:** All source references in this document are anchored to commit `566ad786` (full hash `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). Line numbers cited as `[<path>:L<start>-L<end>]` are valid at this commit. See [`../README.md`](../README.md) for the documentation-set-wide commit-anchor convention and citation format.

> **No pseudocode.** Per the no-pseudocode rule in [`../README.md`](../README.md) (AAP §0.10.3), every codec, format, and behavior decision in the FFmpeg HLS muxer and demuxer is expressed in this document as a **decision table** with the columns **Condition** | **Branch** | **Resulting Behavior** | **Source Citation**. Some tables carry an additional **Risk** column where the decision is high-risk for a port or refactor (segment cut on non-keyframe, segment-type selection, periodic rekey cadence). Control-flow keywords (`if`, `else`, `for`, `while`) do not appear in narrative text — they only appear inside table cells where they describe the condition under which a branch fires.

---

## Overview

This document is the **canonical decision-logic catalog** for the FFmpeg HLS muxer (`libavformat/hlsenc.c`, `libavformat/hlsplaylist.c`, `libavformat/hlsplaylist.h`) and demuxer (`libavformat/hls.c`). Its purpose is to enumerate **every** conditional behavior — every `if`/`else if`/`switch`/ternary branch that controls a user-visible outcome — so that a porting engineer can reproduce the same output, byte-for-byte and behavior-for-behavior, without re-reading the C source.

In plain language: the HLS muxer has a small number of "big" decisions (which container to use, which playlist version to declare, when to cut a segment, when to rotate the encryption key) and a large number of "small" decisions (which tag to emit, which line to write next, which default to apply). All decisions live in this document. Each section opens with a plain-language summary describing what the decision controls and why it matters; the decision table immediately below names every input condition, every branch, the resulting behavior, and the source-code citation that supports the row.

A decision table uses a fixed schema:

- **Condition** — the predicate that selects this row (e.g., `pkt->flags & AV_PKT_FLAG_KEY` is set; `segment_type == SEGMENT_TYPE_FMP4`).
- **Branch** — the variable assignment or call that fires when the condition holds (e.g., `version = 7`; `vs->discontinuity_set = 1`).
- **Resulting Behavior** — the observable outcome in the playlist, segment file, or downstream HTTP exchange (e.g., "`EXT-X-VERSION:7` emitted"; "`EXT-X-DISCONTINUITY` line precedes next segment entry").
- **Source Citation** — `[<repo-relative-path>:L<start>-L<end>]` of the source range that supports the row.

The high-risk decisions are marked **Risk: HIGH** (`HLS_SPLIT_BY_TIME` allowing non-keyframe cuts; `SEGMENT_TYPE_FMP4` selection because it cascades through version, encryption support, and EXT-X-MAP emission; `HLS_PERIODIC_REKEY` because failed re-reads leave indeterminate key state). All other decisions are implicitly Risk: LOW and do not carry an explicit marker.

### Decision Sections in This Document

The decisions are presented in the sequence specified by AAP §0.5.2.7. Each is a top-level H2 section.

1. HLS_VERSION Negotiation — which `EXT-X-VERSION:N` to emit
2. Segment Cut (Keyframe + Time) — when to finalize the current segment and open the next
3. SEGMENT_TYPE Selection — MPEG-TS versus fragmented MP4
4. EXT-X-TARGETDURATION Computation — the integer-second ceiling for player reload cadence
5. EXT-X-DISCONTINUITY Placement — when and where the discontinuity tag appears
6. EXT-X-PROGRAM-DATE-TIME Injection — when the wall-clock tag is emitted
7. Byterange Mode Activation — single-file output with byterange entries
8. Start Sequence Source — how the initial `EXT-X-MEDIA-SEQUENCE` is computed
9. Second-Level Filename Templating — `%d`/`%t`/`%s` expansion in segment filenames
10. HTTP Method Selection — default `PUT` and the `method` AVOption override
11. Periodic Rekey Cadence — re-reading `hls_key_info_file` per segment
12. EXT-X-INDEPENDENT-SEGMENTS Emission — gated by `vs->has_video`
13. EXT-X-I-FRAMES-ONLY Emission — trick-play playlists
14. EXT-X-MAP Emission — fMP4 init-segment reference
15. EXT-X-BYTERANGE Emission — per-segment `<size>@<offset>` line
16. EXT-X-ENDLIST Emission — VOD/end-of-stream signal
17. `hls_init_time` Activation — shorter initial segments for fast start
18. `append_list` Mode — resume from existing M3U8
19. Demuxer Key Type Resolution — `KEY_NONE`/`KEY_AES_128`/`KEY_SAMPLE_AES`

A "Cross-References" section at the bottom links every decision to the related document elsewhere in the set (`data-model.md` for enum and struct definitions, `process-flows.md` for the diagrams that visualize these branches, `../api-contracts/functional-invariants.md` for the "MUST"/"MUST NOT" companion specification).

---

## Decision — HLS_VERSION Negotiation

**Plain-language summary.** The `#EXT-X-VERSION:N` line emitted on the second line of every M3U8 playlist declares the minimum HLS protocol version required for clients to render the playlist correctly. The muxer computes this by starting at version 2 and cascading upward through five feature gates: float-second durations require version 3; byteranges require version 4; I-frames-only mode requires version 4; independent segments require version 6; fragmented MP4 segments require version 7. Each enabled feature pushes the minimum higher; the cascading assignments at `[libavformat/hlsenc.c:L1551-L1571]` are linear and order-independent because each branch only raises the version, never lowers it.

The decision is materially **a MAX, not a switch**: a configuration that enables both byteranges and fragmented MP4 emits version 7 (the higher of 4 and 7). All citations below point at the exact assignment line within the cascade at `[libavformat/hlsenc.c:L1551-L1571]`.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| Initial assignment (always executes first) | `hls->version = 2` | Baseline `EXT-X-VERSION:2` candidate before further gating | `[libavformat/hlsenc.c:L1551]` |
| `HLS_ROUND_DURATIONS` flag is NOT set (i.e., float durations allowed) | `hls->version = 3` | `EXT-X-VERSION:3` raised — `#EXTINF:<float>` permitted | `[libavformat/hlsenc.c:L1552-L1554]` |
| `byterange_mode` is true (`HLS_SINGLE_FILE` set OR `max_seg_size > 0`) | `hls->version = 4` and `sequence = 0` | `EXT-X-VERSION:4` raised — `EXT-X-BYTERANGE` supported; media-sequence reset to 0 | `[libavformat/hlsenc.c:L1556-L1559]` |
| `HLS_I_FRAMES_ONLY` flag is set | `hls->version = 4` | `EXT-X-VERSION:4` raised (or held) — `EXT-X-I-FRAMES-ONLY` supported | `[libavformat/hlsenc.c:L1561-L1563]` |
| `HLS_INDEPENDENT_SEGMENTS` flag is set | `hls->version = 6` | `EXT-X-VERSION:6` raised — `EXT-X-INDEPENDENT-SEGMENTS` permitted | `[libavformat/hlsenc.c:L1565-L1567]` |
| `hls->segment_type == SEGMENT_TYPE_FMP4` | `hls->version = 7` | `EXT-X-VERSION:7` raised — fragmented MP4 segments + `EXT-X-MAP` | `[libavformat/hlsenc.c:L1569-L1571]` |
| Two or more conditions above active together | Successive assignments; final value is the highest | The cascade is monotonic: only upward writes occur | `[libavformat/hlsenc.c:L1551-L1571]` |

The final value is written to the playlist by `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]`, which emits the literal text `#EXT-X-VERSION:%d\n` on the second line (after the mandatory `#EXTM3U` first line).

---

## Decision — Segment Cut (Keyframe + Time)

**Plain-language summary.** The muxer decides whether the current packet should trigger a new segment by combining a **structural** condition (is this packet a video keyframe, or does the user permit non-keyframe cuts via `HLS_SPLIT_BY_TIME`?) with a **temporal** condition (has enough time elapsed since the segment started?). Both must hold for a cut to fire. Cutting a segment at a non-keyframe boundary produces a segment whose first frames cannot be decoded standalone, breaking the standard HLS spec assumption that every segment is self-decodable — this is the documented danger of `HLS_SPLIT_BY_TIME`. The default behavior is **keyframe-only cuts**, which defer segment cuts to the next keyframe when the time budget has expired but the current packet is a P/B frame.

The `can_split` decision is computed in two stages at `[libavformat/hlsenc.c:L2473-L2479]` (structural gate) and refined at `[libavformat/hlsenc.c:L2500]` (`pts - end_pts > 0` confirmation, preventing zero-duration segments). The temporal predicate (`av_compare_ts(pkt->pts - start_pts, ..., end_pts, AV_TIME_BASE_Q) >= 0`) is evaluated at `[libavformat/hlsenc.c:L2501-L2502]`. When both hold and the current segment has at least one packet (`vs->packets_written`), the cut path at `[libavformat/hlsenc.c:L2503-L2675]` flushes the underlying mux, finalizes the segment file, appends a new `HLSSegment` node, publishes a refreshed playlist (mid-stream only — VOD waits), and opens the next segment via `hls_start`.

| Condition | Branch | Resulting Behavior | Source Citation | Risk |
|---|---|---|---|---|
| `vs->has_video` and `pkt->codec_type == VIDEO` and `pkt->flags & AV_PKT_FLAG_KEY` | `can_split = 1` | Cut allowed when the time budget is met — segment starts on a keyframe | `[libavformat/hlsenc.c:L2473-L2475]` | LOW |
| `vs->has_video` and `pkt->codec_type == VIDEO` and NOT `pkt->flags & AV_PKT_FLAG_KEY` and `HLS_SPLIT_BY_TIME` is set | `can_split = 1` | Cut allowed at any video-packet boundary — non-self-decodable segment possible | `[libavformat/hlsenc.c:L2473-L2475]` | HIGH |
| `vs->has_video` and `pkt->codec_type == VIDEO` and NOT `pkt->flags & AV_PKT_FLAG_KEY` and NOT `HLS_SPLIT_BY_TIME` | `can_split = 0` | No cut at this packet — defer to next keyframe | `[libavformat/hlsenc.c:L2473-L2475]` | LOW |
| `vs->has_video == 0` (audio-only variant) | `can_split` remains `1` (default initializer) | Cut allowed at any audio-packet boundary (audio has no keyframe concept in HLS) | `[libavformat/hlsenc.c:L2417,L2473]` | LOW |
| `pkt->pts == AV_NOPTS_VALUE` | `is_ref_pkt = can_split = 0` | Cut disallowed — invalid PTS cannot drive a time-comparison | `[libavformat/hlsenc.c:L2478-L2479]` | LOW |
| `can_split == 1` and `pkt->pts - vs->end_pts > 0` is false | `can_split = 0` (refined) | Cut suppressed — would produce a zero or negative duration segment | `[libavformat/hlsenc.c:L2500]` | LOW |
| `vs->packets_written && can_split && av_compare_ts(pkt->pts - vs->start_pts, st->time_base, end_pts, AV_TIME_BASE_Q) >= 0` | Enter cut path | Flush underlying mux (`av_write_frame(oc, NULL)`), record `vs->size`, finalize segment file, append segment node, publish playlist, open next | `[libavformat/hlsenc.c:L2501-L2675]` | LOW |
| Time threshold not yet met (`av_compare_ts(...) < 0`) | Skip cut path | Packet forwarded to underlying mux via `ff_write_chained` (no segment finalization) | `[libavformat/hlsenc.c:L2677-L2688]` | LOW |
| `vs->packets_written == 0` (segment is empty, no first packet seen yet) | Skip cut path | First packet of segment cannot trigger a cut — would close an empty segment | `[libavformat/hlsenc.c:L2501]` | LOW |

The cut path's first action is `av_write_frame(oc, NULL)` at `[libavformat/hlsenc.c:L2507]`, which flushes the sub-muxer's pending buffers. The segment is then finalized, appended via `hls_append_segment` at `[libavformat/hlsenc.c:L2618]`, and the playlist is republished via `hls_window` at `[libavformat/hlsenc.c:L2628]` only when `pl_type != PLAYLIST_TYPE_VOD` (VOD playlists are published once at trailer time to avoid intermediate-state observation). The next segment is opened via `hls_start` at `[libavformat/hlsenc.c:L2649,L2657,L2667]` depending on byterange/single-file mode.

---

## Decision — SEGMENT_TYPE Selection (MPEG-TS vs fMP4)

**Plain-language summary.** The muxer can produce segments in two container formats: **MPEG-TS** (`.ts`, the historical default) or **fragmented MP4 (fMP4)** (`.m4s`, plus a one-time `init.mp4`). The choice is controlled by the `hls_segment_type` AVOption (`SEGMENT_TYPE_MPEGTS` = 0, default; `SEGMENT_TYPE_FMP4` = 1). The selection cascades through five other decisions: it forces `EXT-X-VERSION` to 7 (because fMP4 is a version 7 feature); it enables `EXT-X-MAP` emission on every playlist publish (because fMP4 segments require a separately-fetched initialization segment containing the moov box); it changes the default filename pattern from `%d.ts` to `%d.m4s` at `[libavformat/hlsenc.c:L2882]`; it forbids whole-segment AES-128 encryption (`AVERROR_PATCHWELCOME` at `[libavformat/hlsenc.c:L1768-L1771]`); and it rejects multi-file byterange mode with `max_seg_size > 0` at `[libavformat/hlsenc.c:L846-L854]`.

The enum is declared at `[libavformat/hlsenc.c:L115-L118]`:

| Condition | Branch | Resulting Behavior | Source Citation | Risk |
|---|---|---|---|---|
| `hls_segment_type=mpegts` (default; AVOption `.i64 = SEGMENT_TYPE_MPEGTS`) | `hls->segment_type = SEGMENT_TYPE_MPEGTS` (value 0) | MPEG-TS PAT/PMT per segment; `.ts` extension; no `EXT-X-MAP`; AES-128 whole-segment encryption permitted | `[libavformat/hlsenc.c:L116,L3139-L3140]` | LOW |
| `hls_segment_type=fmp4` (user-set; AVOption `.i64 = SEGMENT_TYPE_FMP4`) | `hls->segment_type = SEGMENT_TYPE_FMP4` (value 1) | fragmented MP4 segments; `.m4s` extension; one-time `init.mp4` published via `EXT-X-MAP`; `EXT-X-VERSION` raised to 7 | `[libavformat/hlsenc.c:L117,L3141]` | HIGH |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and (`c->key_info_file` or `c->encrypt` is non-zero) | Return `AVERROR_PATCHWELCOME` from `hls_start` | Whole-segment AES-128 with fMP4 not supported — configuration rejected | `[libavformat/hlsenc.c:L1768-L1772]` | HIGH |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and `hls->max_seg_size > 0` | Return `AVERROR_PATCHWELCOME` from `hls_mux_init` | Multi-file byterange (split by size) with fMP4 not supported | `[libavformat/hlsenc.c:L846-L854]` | HIGH |
| `hls->segment_type == SEGMENT_TYPE_FMP4` (any) | Default extension changes from `%d.ts` to `%d.m4s` | Segment filename pattern at variant init differs from MPEG-TS default | `[libavformat/hlsenc.c:L2882]` | LOW |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and writing first playlist entry | `ff_hls_write_init_file` writes `#EXT-X-MAP:URI="<fmp4_init_filename>"` before file entries | Init segment URI declared at the top of the segment list | `[libavformat/hlsenc.c:L1611-L1614]`, `[libavformat/hlsplaylist.c:L134-L142]` | LOW |
| `hls->segment_type == SEGMENT_TYPE_FMP4` (any) | `hls->version = 7` (in the version cascade) | `EXT-X-VERSION:7` mandatory for fMP4 segments | `[libavformat/hlsenc.c:L1569-L1571]` | LOW |

The `AVERROR_PATCHWELCOME` returns at `[libavformat/hlsenc.c:L1771]` and `[libavformat/hlsenc.c:L853]` are the two **hard incompatibilities** with fMP4 selection. Both are silent-fail-free — the muxer logs `AV_LOG_ERROR` ("Encrypted fmp4 not yet supported"; "Multi-file byterange mode is currently unsupported in the HLS muxer") and returns the error code rather than producing a malformed playlist. This is the only hard option incompatibility in the muxer.

---

## Decision — EXT-X-TARGETDURATION Computation

**Plain-language summary.** Every M3U8 playlist begins with `#EXT-X-TARGETDURATION:N`, an integer-second value that clients use as a hint for how often to reload the playlist and as an upper bound on segment duration. The muxer computes `N` as the **maximum integer-rounded duration** across all segments currently in the window. The rounding is done via `lrint(double)`, which is C99 round-to-nearest-even for floating-point seconds.

The computation runs inside `hls_window` at `[libavformat/hlsenc.c:L1584-L1587]` immediately before the playlist header is written, so the value reflects the segments in `vs->segments` at the moment of publication. There is **no caching across publications**: every call to `hls_window` recomputes the maximum.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| Loop iteration over `vs->segments` linked list, with current node `en` | `target_duration <= en->duration` predicate | Per-segment integer ceiling computed via `lrint(en->duration)` | `[libavformat/hlsenc.c:L1584-L1587]` |
| `target_duration <= en->duration` is true | `target_duration = lrint(en->duration)` | Running maximum updated | `[libavformat/hlsenc.c:L1585-L1586]` |
| `target_duration <= en->duration` is false (current row's segment is shorter than running max) | No update | Running maximum retained | `[libavformat/hlsenc.c:L1585-L1586]` |
| Loop ends (last segment processed) | Final value frozen | `target_duration` reflects max integer-second duration across the window | `[libavformat/hlsenc.c:L1587]` |
| `target_duration` passed to `ff_hls_write_playlist_header` | Emitted as `#EXT-X-TARGETDURATION:%d\n` | Playlist header line written by `avio_printf` | `[libavformat/hlsenc.c:L1590-L1591]`, `[libavformat/hlsplaylist.c:L120]` |

Because the value is integer seconds and segments are tracked as `double` seconds via `vs->dpp` accumulation, a segment whose duration is 2.7 s emits `target_duration = 3` (`lrint(2.7) = 3`). Per RFC 8216 §4.4.2.1, the target duration **MUST** be greater than or equal to the maximum actual segment duration — `lrint`'s round-to-nearest semantics satisfy this only when the longest segment rounds up rather than down; the implementation accepts this minor risk because over many segments the value converges and clients tolerate one-second slack. The corresponding invariant is restated in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).

---

## Decision — EXT-X-DISCONTINUITY Placement

**Plain-language summary.** The `#EXT-X-DISCONTINUITY` tag tells the player to reset its internal PTS/PCR tracking before the next segment. Two situations trigger emission: (a) the very first segment of a sequence when the user sets `hls_flags=discont_start` (used when resuming after a known external interruption); (b) a mid-stream segment whose `discont` field was set on append because the muxer detected a PTS jump or because `parse_playlist` flagged it during an `append_list` resume.

The first-segment emission is written directly into the playlist header at `[libavformat/hlsenc.c:L1593-L1596]`. The mid-stream emission is carried on each `HLSSegment` node's `discont` field at `[libavformat/hlsenc.c:L80,L1091]` and rendered as the `insert_discont` argument to `ff_hls_write_file_entry` at `[libavformat/hlsenc.c:L1616]`, which emits `#EXT-X-DISCONTINUITY\n` at `[libavformat/hlsplaylist.c:L156-L158]` before that segment's `#EXTINF` line.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_DISCONT_START` flag set AND `sequence == hls->start_sequence` AND `vs->discontinuity_set == 0` | `avio_printf("#EXT-X-DISCONTINUITY\n")` and `vs->discontinuity_set = 1` | Discontinuity declared at sequence start (one-shot — guarded by `discontinuity_set` so it fires only once) | `[libavformat/hlsenc.c:L1593-L1596]` |
| Mid-stream segment with `vs->discontinuity` set | `en->discont = 1` and `vs->discontinuity = 0` (on append) | Discontinuity carried on segment node | `[libavformat/hlsenc.c:L1090-L1093]` |
| Segment iterated in `hls_window` with `en->discont == 1` | `ff_hls_write_file_entry` called with `insert_discont = 1` | `#EXT-X-DISCONTINUITY\n` written before the segment's `#EXTINF` line | `[libavformat/hlsenc.c:L1616]`, `[libavformat/hlsplaylist.c:L156-L158]` |
| Append-list resume (`HLS_APPEND_LIST` set) | `vs->discontinuity = 1` set in `hls_init` after `parse_playlist` | Next-appended segment carries discontinuity flag | `[libavformat/hlsenc.c:L3100-L3102]` |
| Time-gap detected by `parse_playlist` while reading existing M3U8 | `discont_program_date_time` accumulator updated; per-segment carriage of recompute | Mid-stream PTS reset can be recomputed against wall clock | `[libavformat/hlsenc.c:L1273-L1276]` |
| No conditions above are active | `en->discont = 0` and `vs->discontinuity_set` unchanged | No discontinuity line emitted; player retains continuous PTS tracking | `[inferred — no direct source]` |

The discontinuity-line emission is order-locked: it always precedes the `#EXTINF` for the affected segment — see [`../api-contracts/timing-dependencies.md`](../api-contracts/timing-dependencies.md) for the line-ordering contract.

---

## Decision — EXT-X-PROGRAM-DATE-TIME Injection

**Plain-language summary.** When `hls_flags=program_date_time` is set, every segment in the playlist is preceded by a `#EXT-X-PROGRAM-DATE-TIME:<ISO-8601-timestamp>` line that gives the wall-clock anchor for that segment's first sample. The anchor is seeded once at `hls_init` time via `av_gettime() / 1000000.0` (Unix epoch seconds as a `double`) and advanced segment-by-segment by the segment's duration. When a discontinuity is detected mid-stream, the anchor is **recomputed from the new system time** rather than advanced, because the previous accumulated total is no longer aligned with reality.

The flag is `HLS_PROGRAM_DATE_TIME = (1 << 7)` at `[libavformat/hlsenc.c:L105]`. The seed is `initial_program_date_time = av_gettime() / 1000000.0` at `[libavformat/hlsenc.c:L2877]`, propagated to every variant stream at `[libavformat/hlsenc.c:L2972]`. The pointer-or-null gate is at `[libavformat/hlsenc.c:L1547-L1548]`: `prog_date_time_p` is non-NULL only when the flag is set, and it is passed to `ff_hls_write_file_entry` at `[libavformat/hlsenc.c:L1620]`, which prints the timestamp with millisecond precision at `[libavformat/hlsplaylist.c:L167-L192]` and advances the accumulator at `[libavformat/hlsplaylist.c:L192]`. The discontinuity recompute is at `[libavformat/hlsenc.c:L1622-L1623]`.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_PROGRAM_DATE_TIME` flag NOT set | `prog_date_time_p = NULL` | No `EXT-X-PROGRAM-DATE-TIME` line emitted on any segment | `[libavformat/hlsenc.c:L105,L1547-L1548]` |
| `HLS_PROGRAM_DATE_TIME` set, first segment of variant | Anchor `prog_date_time = vs->initial_prog_date_time` seeded from `av_gettime()` at `hls_init` | First PDT line uses the muxer's `hls_init` system time | `[libavformat/hlsenc.c:L1547,L2877,L2972]` |
| `HLS_PROGRAM_DATE_TIME` set, mid-stream continuous segment | `ff_hls_write_file_entry` emits `#EXT-X-PROGRAM-DATE-TIME:<ISO>` then advances `*prog_date_time += duration` | Each subsequent segment's PDT = prior PDT + prior duration (rolling sum) | `[libavformat/hlsenc.c:L1620]`, `[libavformat/hlsplaylist.c:L191-L192]` |
| `HLS_PROGRAM_DATE_TIME` set, segment with `en->discont_program_date_time != 0` | `ff_hls_write_file_entry` uses `&en->discont_program_date_time` instead of the rolling pointer | Discontinuity-segment PDT is its own pinned wall-clock value, not the rolling sum | `[libavformat/hlsenc.c:L1620]` |
| Discontinuity-segment PDT consumed | `en->discont_program_date_time -= en->duration` (post-emission) | Next playlist publish does not double-count the discontinuity offset | `[libavformat/hlsenc.c:L1622-L1623]` |
| `parse_playlist` reads an existing `#EXT-X-PROGRAM-DATE-TIME` line during append_list | `discont_program_date_time` local accumulator updated; carried into newly appended segment via `vs->last_segment->discont_program_date_time` | Resumed stream re-anchors PDT from the existing playlist's last value | `[libavformat/hlsenc.c:L1273-L1276]` |
| `HLS_PROGRAM_DATE_TIME` set; `hls_list_size` aging (controls `max_nb_segments`) removes oldest segment | `vs->initial_prog_date_time += en->duration` when oldest segment's PDT is not its own discontinuity anchor | Sliding window advances PDT seed forward | `[libavformat/hlsenc.c:L1124-L1125]` |

The ISO-8601 format emitted by `ff_hls_write_file_entry` is `YYYY-MM-DDTHH:MM:SS.<msec><tz>` at `[libavformat/hlsplaylist.c:L175-L191]`, with three-digit millisecond precision (`av_clip(lrint(1000*(*prog_date_time - tt)), 0, 999)` at `[libavformat/hlsplaylist.c:L173]`) and a fall-back UTC-offset computation when `strftime("%z", ...)` does not yield a usable two-digit hour-offset prefix at `[libavformat/hlsplaylist.c:L179-L190]`.

---

## Decision — Byterange Mode Activation

**Plain-language summary.** Byterange mode is an alternative segment layout in which segments are byte-ranges inside one or more larger files instead of separate files-per-segment. Each `#EXTINF` line is followed by a `#EXT-X-BYTERANGE:<size>@<offset>` line. The mode is activated when either (a) `HLS_SINGLE_FILE` is set — all segments live in a single output file — or (b) `max_seg_size > 0` — segments are bundled in files of up to that many bytes, with new files opened as needed. The activation predicate is the C expression `(hls->flags & HLS_SINGLE_FILE) || (hls->max_seg_size > 0)` at three sites: `[libavformat/hlsenc.c:L779]` (in `hls_mux_init`), `[libavformat/hlsenc.c:L1049]` (in `hls_append_segment`), and `[libavformat/hlsenc.c:L1549]` (in `hls_window`), and `[libavformat/hlsenc.c:L2504]` (in `hls_write_packet`).

When byterange mode is active, the `EXT-X-VERSION` cascade forces a minimum of 4 (because `EXT-X-BYTERANGE` is a version 4 feature) and **resets `sequence` to 0** at `[libavformat/hlsenc.c:L1558]` so byterange offsets remain stable relative to file start.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_SINGLE_FILE` flag set (1<<0) | `byterange_mode = 1` | All segments share one output file; each segment is a byte-range | `[libavformat/hlsenc.c:L98,L1549]` |
| `hls->max_seg_size > 0` (and `SEGMENT_TYPE_MPEGTS`) | `byterange_mode = 1` | Multi-file byterange — each file capped at `max_seg_size` bytes | `[libavformat/hlsenc.c:L1549]` |
| Both `HLS_SINGLE_FILE` and `max_seg_size > 0` | `byterange_mode = 1` | `HLS_SINGLE_FILE` dominates (output remains one file regardless of size cap) | `[libavformat/hlsenc.c:L1549]` |
| Neither condition | `byterange_mode = 0` | Standard one-segment-per-file mode | `[libavformat/hlsenc.c:L1549]` |
| `byterange_mode = 1` (any path) | `hls->version = 4` and `sequence = 0` | Version pinned to 4 minimum; media-sequence reset to 0 for offset stability | `[libavformat/hlsenc.c:L1557-L1559]` |
| `byterange_mode = 1` and writing playlist | `m3u8_out` shared dynamic buffer used instead of per-variant `vs->out` | Single-output-file mode reuses one writer | `[libavformat/hlsenc.c:L1578,L1590]` |
| `byterange_mode = 1` and writing file entry | `ff_hls_write_file_entry` called with `byterange_mode = 1` | `#EXT-X-BYTERANGE:<size>@<offset>` emitted after `#EXTINF` | `[libavformat/hlsenc.c:L1616]`, `[libavformat/hlsplaylist.c:L163-L165]` |
| `byterange_mode = 1` and `SEGMENT_TYPE_FMP4` and `max_seg_size > 0` | Return `AVERROR_PATCHWELCOME` from `hls_mux_init` | Combination explicitly rejected | `[libavformat/hlsenc.c:L846-L854]` |

The reset to `sequence = 0` on byterange activation at `[libavformat/hlsenc.c:L1558]` is **non-negotiable**: clients use the byte offsets reported in `#EXT-X-BYTERANGE` to seek inside the single output file, and a non-zero start-sequence would shift the offset frame of reference. The corresponding invariant is restated in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).

---

## Decision — Start Sequence Source (4 Modes)

**Plain-language summary.** The first segment's `#EXT-X-MEDIA-SEQUENCE` value can come from one of four sources, selected by the `hls_start_number_source` AVOption. The default ("generic") uses the literal `start_number` integer the user provided; the other three derive it from the current wall clock (seconds since Unix epoch, microseconds since Unix epoch, or a 14-digit decimal `YYYYMMDDhhmmss` representation of the local time). The wall-clock modes are useful when restarting an HLS stream and clients need a fresh, monotonic sequence number — for example, broadcasters who re-encode the same channel at a regular cadence.

The four enum values are declared at `[libavformat/hlsenc.c:L57-L63]`:

- `HLS_START_SEQUENCE_AS_START_NUMBER = 0` at L58
- `HLS_START_SEQUENCE_AS_SECONDS_SINCE_EPOCH = 1` at L59
- `HLS_START_SEQUENCE_AS_FORMATTED_DATETIME = 2` at L60 (with the YYYYMMDDhhmmss formatting comment at L60)
- `HLS_START_SEQUENCE_AS_MICROSECONDS_SINCE_EPOCH = 3` at L61

The resolution happens at `[libavformat/hlsenc.c:L2931-L2948]` inside `hls_init`. The default branch (mode 0) is implicit: no assignment runs, and `hls->start_sequence` retains the user-supplied `start_number` AVOption value (whose default is 0 per `[libavformat/hlsenc.c:L3122]`).

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `start_sequence_source_type == HLS_START_SEQUENCE_AS_START_NUMBER` (default, value 0) | No reassignment in `hls_init` | `hls->start_sequence` retains the `start_number` AVOption value (default 0) | `[libavformat/hlsenc.c:L58,L3122]` |
| `start_sequence_source_type == HLS_START_SEQUENCE_AS_SECONDS_SINCE_EPOCH` (value 1) | `hls->start_sequence = (int64_t)t` where `t = time(NULL)` | First `#EXT-X-MEDIA-SEQUENCE` is Unix epoch seconds at `hls_init` time | `[libavformat/hlsenc.c:L59,L2934,L2937-L2938]` |
| `start_sequence_source_type == HLS_START_SEQUENCE_AS_FORMATTED_DATETIME` (value 2) | `strftime("%Y%m%d%H%M%S", localtime_r(&t, ...))` → `strtoll(..., NULL, 10)` | First `#EXT-X-MEDIA-SEQUENCE` is 14-digit decimal local time (e.g., `20240115143022`) | `[libavformat/hlsenc.c:L60,L2939-L2946]` |
| `start_sequence_source_type == HLS_START_SEQUENCE_AS_MICROSECONDS_SINCE_EPOCH` (value 3) | `hls->start_sequence = av_gettime()` | First `#EXT-X-MEDIA-SEQUENCE` is microseconds since epoch | `[libavformat/hlsenc.c:L61,L2935-L2936]` |
| Mode 2 (`FORMATTED_DATETIME`) and `localtime_r` returns NULL | Return `AVERROR(errno)` | `hls_init` fails — wall-clock unavailable | `[libavformat/hlsenc.c:L2942-L2943]` |
| Mode 2 (`FORMATTED_DATETIME`) and `strftime` returns 0 | Return `AVERROR(ENOMEM)` | `hls_init` fails — formatted string did not fit in buffer | `[libavformat/hlsenc.c:L2944-L2945]` |
| Any mode 1/2/3 used | `AV_LOG_DEBUG` "start_number evaluated to %d" logged | Resolved value visible at log level DEBUG | `[libavformat/hlsenc.c:L2948]` |

The wall-clock modes only fire once at `hls_init` — there is no per-segment re-evaluation. Subsequent `#EXT-X-MEDIA-SEQUENCE` values increment from this starting point as segments are appended.

---

## Decision — Second-Level Filename Templating

**Plain-language summary.** Second-level filename templating allows the user to embed the sequence number (`%d`), segment duration in microseconds (`%t`), or segment cumulative byte size (`%s`) into the segment filename pattern. This is opt-in via the `HLS_SECOND_LEVEL_SEGMENT_INDEX`, `HLS_SECOND_LEVEL_SEGMENT_DURATION`, and `HLS_SECOND_LEVEL_SEGMENT_SIZE` flags. The substitutions happen in two passes: the **at-open pass** in `sls_flag_use_localtime_filename` at `[libavformat/hlsenc.c:L998-L1037]` (when the segment file is opened) and the **at-close pass** in `sls_flags_filename_process` at `[libavformat/hlsenc.c:L908-L946]` (when the segment is finalized and the actual size/duration is known).

The flags are at `[libavformat/hlsenc.c:L106-L108]`. The `%d` placeholder is replaced at open time with `vs->sequence`. The `%s` and `%t` placeholders cannot be filled at open time (size and duration are unknown until the segment closes), so they are placeholder-substituted with `0` at open time and replaced with the real values at close. The intermediate filename pattern is preserved in `vs->current_segment_final_filename_fmt`.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| None of `HLS_SECOND_LEVEL_SEGMENT_*` flags set | No template substitution by `sls_flag_use_localtime_filename` or `sls_flags_filename_process` | Filename used as-is (or via `strftime` if `use_localtime` set) | `[libavformat/hlsenc.c:L998-L1004,L912]` |
| `HLS_SECOND_LEVEL_SEGMENT_INDEX` set (1<<8) | `replace_int_data_in_filename(&filename, oc->url, 'd', vs->sequence)` at file-open time | `%d` in filename → current sequence number | `[libavformat/hlsenc.c:L106,L1000-L1010]` |
| `HLS_SECOND_LEVEL_SEGMENT_INDEX` substitution returns `< 1` (no placeholder found) | `AV_LOG_ERROR` and return `AVERROR(EINVAL)` | Open path fails — template lacks `%d` despite flag being set | `[libavformat/hlsenc.c:L1004-L1008]` |
| `HLS_SECOND_LEVEL_SEGMENT_SIZE` set (1<<10) | `av_strlcpy` saves URL into `vs->current_segment_final_filename_fmt`; `replace_int_data_in_filename('s', 0)` substitutes placeholder | `%s` reserved with `0`; real value substituted at close | `[libavformat/hlsenc.c:L108,L1013-L1024]` |
| `HLS_SECOND_LEVEL_SEGMENT_DURATION` set (1<<9) | Same `current_segment_final_filename_fmt` save; `replace_int_data_in_filename('t', 0)` substitutes placeholder | `%t` reserved with `0`; real value substituted at close | `[libavformat/hlsenc.c:L107,L1026-L1036]` |
| At-close pass (`sls_flags_filename_process`) and `HLS_SECOND_LEVEL_SEGMENT_SIZE` or `HLS_SECOND_LEVEL_SEGMENT_DURATION` is set and `vs->current_segment_final_filename_fmt` non-empty | `ff_format_set_url(vs->avf, av_strdup(vs->current_segment_final_filename_fmt))` restores template | At-close substitution begins from preserved template | `[libavformat/hlsenc.c:L912-L918]` |
| At-close, `HLS_SECOND_LEVEL_SEGMENT_SIZE` set | `replace_int_data_in_filename(&filename, vs->avf->url, 's', pos + size)` | `%s` → final segment size in bytes (`pos + size`) | `[libavformat/hlsenc.c:L919-L930]` |
| At-close, `HLS_SECOND_LEVEL_SEGMENT_DURATION` set | `replace_int_data_in_filename(&filename, vs->avf->url, 't', (int64_t)round(duration * HLS_MICROSECOND_UNIT))` | `%t` → duration in microseconds (rounded) | `[libavformat/hlsenc.c:L931-L943]` |
| At-close substitution returns `< 1` (no placeholder found) | `AV_LOG_ERROR` and return `AVERROR(EINVAL)` | Close path fails — template lacks `%s` or `%t` despite flag being set | `[libavformat/hlsenc.c:L921-L926,L933-L938]` |
| Any second-level flag set but `use_localtime` NOT set | Configuration rejected by `sls_flag_check_duration_size_index` (returns `AVERROR(EINVAL)`) | Second-level templating requires `use_localtime` | `[libavformat/hlsenc.c:L948-L969]` |
| Any second-level flag set and `file:` protocol NOT used | Configuration rejected by `sls_flag_check_duration_size` | Second-level templating requires file output (cannot be HTTP) | `[libavformat/hlsenc.c:L971-L989]` |

The `HLS_MICROSECOND_UNIT` constant is defined at `[libavformat/hlsenc.c:L72]` as `1000000`, so a 2.7 s segment with `HLS_SECOND_LEVEL_SEGMENT_DURATION` yields a `%t` substitution of `2700000`. Composite templates (e.g., `seg_%d_%t_%s.ts`) are supported by chaining substitutions: index resolves at open, then duration and size resolve at close.

---

## Decision — HTTP Method Selection

**Plain-language summary.** When the output URL uses the `http://` or `https://` scheme, the HLS muxer needs an HTTP verb for each segment and playlist upload. The default verb is `PUT`; the user can override via the `method` AVOption (typically to use `POST`). For non-HTTP outputs (file, pipe, custom protocols), no `method` dictionary entry is set, and the underlying I/O backend ignores the absence.

The `set_http_options` function at `[libavformat/hlsenc.c:L333-L350]` populates an `AVDictionary *options` that is passed to every `avio_open2`/`hlsenc_io_open` call. The HTTP-method assignment is the first thing it does, at `[libavformat/hlsenc.c:L337-L341]`.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `c->method` is non-NULL (user-set `method` AVOption) | `av_dict_set(options, "method", c->method, 0)` | User-provided verb (e.g., `POST`, `PUT`, custom) propagated to HTTP backend | `[libavformat/hlsenc.c:L337-L338]` |
| `c->method` is NULL and `ff_is_http_proto(s->url) != 0` (output URL is HTTP/HTTPS) | `av_dict_set(options, "method", "PUT", 0)` | Default `PUT` applied | `[libavformat/hlsenc.c:L339-L340]` |
| `c->method` is NULL and `ff_is_http_proto(s->url) == 0` (non-HTTP output) | No `method` entry inserted | I/O backend uses its native verb (no HTTP method needed) | `[libavformat/hlsenc.c:L339-L340]` (negative path) |
| `c->user_agent` non-NULL | `av_dict_set(options, "user_agent", c->user_agent, 0)` | `User-Agent` header propagated | `[libavformat/hlsenc.c:L342-L343]` |
| `c->http_persistent != 0` | `av_dict_set_int(options, "multiple_requests", 1, 0)` | TCP connection reused across segment uploads via `multiple_requests` | `[libavformat/hlsenc.c:L344-L345]` |
| `c->timeout >= 0` | `av_dict_set_int(options, "timeout", c->timeout, 0)` | Per-request timeout (microseconds) propagated | `[libavformat/hlsenc.c:L346-L347]` |
| `c->headers` non-NULL | `av_dict_set(options, "headers", c->headers, 0)` | Extra HTTP headers propagated verbatim | `[libavformat/hlsenc.c:L348-L349]` |
| `hls_init` detects HTTP URL and `c->method` is NULL | `AV_LOG_WARNING` "No HTTP method set, hls muxer defaulting to method PUT" logged | User informed of implicit default | `[libavformat/hlsenc.c:L2896-L2898]` |

The HTTP-method dictionary is reset by every call (the same `AVDictionary **` is repeatedly populated and consumed). The persistent-connection effect is a downstream contract — see [`integration-contracts.md`](../api-contracts/integration-contracts.md) for the HTTP persistence specification.

---

## Decision — Periodic Rekey Cadence

**Risk: HIGH.** A failed key-file re-read leaves the variant stream's `vs->key_string` and `vs->iv_string` either stale (and the next segment encrypts to the old key while the playlist declares a new URI), inconsistent (truncated read), or inaccessible (file unlinked). Downstream players cannot decrypt without successfully fetching the new key URI. Operators who deploy `HLS_PERIODIC_REKEY` must ensure key-file rotation is atomic (write-to-temp-then-rename) and the underlying file system honors that ordering.

**Plain-language summary.** When `HLS_PERIODIC_REKEY` is set and the user supplied an `hls_key_info_file`, the muxer re-reads that file at every segment boundary. The re-read happens inside `hls_start`, before any segment writing — see `[libavformat/hlsenc.c:L1779-L1782]`. Without the flag, the file is read once during `hls_init` (or first segment start) and the key remains constant for the entire stream. The trigger predicate at `[libavformat/hlsenc.c:L1779]` is `if (!vs->encrypt_started || (c->flags & HLS_PERIODIC_REKEY))`, so the first segment always reads (when `encrypt_started == 0`) and every subsequent segment reads only when the flag is set.

The flag is at `[libavformat/hlsenc.c:L110]`. The re-read action is the call to `hls_encryption_start` at `[libavformat/hlsenc.c:L1781]`, which is the same function used at first-segment time. Failure to read the file returns `AVERROR(EINVAL)` from `hls_encryption_start` and bubbles up through the `goto fail` at `[libavformat/hlsenc.c:L1782]`.

| Condition | Branch | Resulting Behavior | Source Citation | Risk |
|---|---|---|---|---|
| `HLS_PERIODIC_REKEY` flag NOT set (default) | First segment: `!vs->encrypt_started` is true → `hls_encryption_start` called; subsequent segments skipped | Key read once, then constant for entire stream | `[libavformat/hlsenc.c:L110,L1779-L1781]` | LOW |
| `HLS_PERIODIC_REKEY` set, at every segment boundary | `(c->flags & HLS_PERIODIC_REKEY)` is true → `hls_encryption_start` called every segment | `hls_key_info_file` re-read; vs->key_uri / vs->key_string / vs->iv_string refreshed | `[libavformat/hlsenc.c:L1779-L1781]` | HIGH |
| Re-read succeeds, contents unchanged | `vs->key_uri` / `vs->key_string` / `vs->iv_string` re-populated with same values | Same `EXT-X-KEY:METHOD=AES-128` line emitted at next playlist publish — no client-visible change | `[libavformat/hlsenc.c:L1781,L1603-L1608]` | LOW |
| Re-read succeeds, contents changed | Key URI / key bytes / IV updated; segment encrypted with new key | New `EXT-X-KEY` line precedes the next segment entry in the playlist; clients must re-fetch the key URI | `[libavformat/hlsenc.c:L1603-L1608]` | HIGH |
| Re-read FAILS — missing key URI in info file | `hls_encryption_start` returns `AVERROR(EINVAL)` at L744 | `goto fail` in `hls_start` — segment write aborted | `[libavformat/hlsenc.c:L744,L1782]` | HIGH |
| Re-read FAILS — missing key file path in info file | `hls_encryption_start` returns `AVERROR(EINVAL)` at L749 | `goto fail` in `hls_start` — segment write aborted | `[libavformat/hlsenc.c:L749,L1782]` | HIGH |
| Re-read FAILS — key file unreadable (I/O error) | `hls_encryption_start` returns AVERROR from `avio_read` failure | `goto fail` in `hls_start` — segment write aborted | `[libavformat/hlsenc.c:L760-L765,L1782]` | HIGH |
| `c->encrypt` set instead of `c->key_info_file`, `HLS_PERIODIC_REKEY` set | `do_encrypt` only called when `c->encrypt_started == 0` (one-shot); flag does NOT trigger inline key rotation | Auto-generated key is one-shot; periodic rekey applies only to `key_info_file` mode | `[libavformat/hlsenc.c:L1784-L1792]` | LOW |

The interaction with auto-generated encryption (`hls_enc`) is important: the periodic-rekey flag only causes re-reads of an **external** `hls_key_info_file`. With `hls_enc=1` and `HLS_PERIODIC_REKEY` set, the auto-generated key is still produced only once (at `[libavformat/hlsenc.c:L1786-L1789]`, gated on `!c->encrypt_started`), and the flag has no observable effect.

---

## Decision — EXT-X-INDEPENDENT-SEGMENTS Emission

**Plain-language summary.** The `#EXT-X-INDEPENDENT-SEGMENTS` tag declares that every segment in the playlist begins with a self-decodable boundary — for video, an IDR keyframe. The tag is emitted in the playlist body (not the header) when both the `HLS_INDEPENDENT_SEGMENTS` flag is set **and** the variant has video. Audio-only variants do not emit the tag even when the flag is set, because the concept does not apply (every audio frame is independently decodable).

The flag is `HLS_INDEPENDENT_SEGMENTS = (1 << 13)` at `[libavformat/hlsenc.c:L111]`. The emission gate is at `[libavformat/hlsenc.c:L1597-L1599]`. Note that the emission predicate is **conjunction**: `vs->has_video && (hls->flags & HLS_INDEPENDENT_SEGMENTS)`.

A second interaction at `[libavformat/hlsenc.c:L2953-L2959]`: if both `HLS_SPLIT_BY_TIME` and `HLS_INDEPENDENT_SEGMENTS` are set, `HLS_INDEPENDENT_SEGMENTS` is forcibly cleared (with `AV_LOG_WARNING`) because non-keyframe cuts cannot guarantee segment independence.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_INDEPENDENT_SEGMENTS` flag NOT set (default) | No emission | Tag absent | `[libavformat/hlsenc.c:L111]` |
| `HLS_INDEPENDENT_SEGMENTS` set AND `vs->has_video` is true | `avio_printf(... "#EXT-X-INDEPENDENT-SEGMENTS\n")` in playlist body | Tag emitted once per playlist publish, after the EXTM3U/version/target-duration block | `[libavformat/hlsenc.c:L1597-L1599]` |
| `HLS_INDEPENDENT_SEGMENTS` set AND `vs->has_video` is false (audio-only) | No emission (predicate fails on `vs->has_video`) | Tag suppressed — concept N/A for audio-only | `[libavformat/hlsenc.c:L1597-L1599]` |
| `HLS_INDEPENDENT_SEGMENTS` set AND `HLS_SPLIT_BY_TIME` ALSO set | At `hls_init`, `HLS_INDEPENDENT_SEGMENTS` cleared from `hls->flags` and `AV_LOG_WARNING` logged | Tag never emitted (flag is no longer set) | `[libavformat/hlsenc.c:L2953-L2959]` |
| `HLS_INDEPENDENT_SEGMENTS` set | `EXT-X-VERSION` raised to 6 minimum | Tag is a version 6 feature; cascading version bump | `[libavformat/hlsenc.c:L1565-L1567]` |

The emission site is **inside** the playlist body (between the header and the segment list) and is unconditional across playlists once enabled — every publish during the run emits the tag (it is not a one-shot like `HLS_DISCONT_START`).

---

## Decision — EXT-X-I-FRAMES-ONLY Emission

**Plain-language summary.** When `hls_flags=iframes_only` is set, the muxer produces an I-frame-only playlist intended for trick-play (fast forward / rewind scrubbing in HLS players). The playlist contains only I-frame-aligned segments, and `#EXT-X-I-FRAMES-ONLY` declares this. The implementation pins `EXT-X-VERSION` to 4 (the minimum version supporting the tag), and the actual `#EXT-X-I-FRAMES-ONLY` line is written into the playlist header by `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L129-L130]` when its `iframe_mode` parameter is non-zero.

The flag is `HLS_I_FRAMES_ONLY = (1 << 14)` at `[libavformat/hlsenc.c:L112]`. The version pin happens at `[libavformat/hlsenc.c:L1561-L1563]`. The actual emission lives in the I-frames-only sub-playlist write path (`vs->iframes_only`) rather than the main path — see `hlsplaylist.c:L129-L130` where the `iframe_mode` parameter controls emission.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_I_FRAMES_ONLY` flag NOT set (default) | No emission; standard playlist path | `#EXT-X-I-FRAMES-ONLY` absent | `[libavformat/hlsenc.c:L112]` |
| `HLS_I_FRAMES_ONLY` set | `hls->version = 4` (in cascade, raises baseline) | Version pinned to 4 minimum | `[libavformat/hlsenc.c:L1561-L1563]` |
| `HLS_I_FRAMES_ONLY` set, writing iframes-only sub-playlist | `ff_hls_write_playlist_header(..., iframe_mode = 1)` | `avio_printf(out, "#EXT-X-I-FRAMES-ONLY\n")` emitted in playlist header | `[libavformat/hlsplaylist.c:L129-L130]` |
| `iframe_mode = 1` (I-frames-only playlist) | `ff_hls_write_file_entry` uses `en->video_keyframe_pos` / `en->video_keyframe_size` for `EXT-X-BYTERANGE` instead of `en->pos` / `en->size` | Byte-range points at the keyframe region of the segment, not the full segment | `[libavformat/hlsplaylist.c:L162-L165]` |
| Main playlist (non-iframes-only) emission | `iframe_mode = 0` passed to `ff_hls_write_playlist_header` | `#EXT-X-I-FRAMES-ONLY` line not emitted | `[libavformat/hlsplaylist.c:L129-L130]` |

The I-frames-only playlist coexists with the main playlist when the flag is set; both are published from `hls_window` with separate `vs->out` (main) and `hls->sub_m3u8_out` (I-frames-only) writers at `[libavformat/hlsenc.c:L1640-L1655]`.

---

## Decision — EXT-X-MAP Emission (fMP4 Init Reference)

**Plain-language summary.** Fragmented MP4 segments do not self-describe — they reference a separate initialization segment (`init.mp4` by default) containing the moov box (codec parameters, timescales, etc.). The playlist tells clients about this init via `#EXT-X-MAP:URI="<filename>"` written before the first `#EXTINF`. For MPEG-TS segments, no `EXT-X-MAP` is needed because PAT/PMT live inside each TS segment.

The init file is written by `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]`, called from `hls_window` at `[libavformat/hlsenc.c:L1611-L1614]` only when `hls->segment_type == SEGMENT_TYPE_FMP4`. When `HLS_SINGLE_FILE` is set (single-file mode + fMP4 — a valid combination), the init segment is part of the same output file as the media segments, so the URI is the segment filename itself with a byte-range pointing at the init region. When `HLS_SINGLE_FILE` is not set, the URI is `vs->fmp4_init_filename` (default `init.mp4`, configurable via `hls_fmp4_init_filename`).

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `hls->segment_type == SEGMENT_TYPE_MPEGTS` (default) | `ff_hls_write_init_file` NOT called | No `#EXT-X-MAP` line in playlist; TS segments self-describe via PAT/PMT | `[libavformat/hlsenc.c:L116,L1611]` |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and `i == start_index` (writing first segment of window) | `ff_hls_write_init_file` called with URI argument | `#EXT-X-MAP:URI="<filename>"` emitted before first `#EXTINF` | `[libavformat/hlsenc.c:L1611-L1614]`, `[libavformat/hlsplaylist.c:L134-L142]` |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and `HLS_SINGLE_FILE` set | URI = `en->filename` (segment's own URI); byterange written | `#EXT-X-MAP:URI="...",BYTERANGE="<size>@<offset>"` | `[libavformat/hlsenc.c:L1611-L1613]`, `[libavformat/hlsplaylist.c:L138-L140]` |
| `hls->segment_type == SEGMENT_TYPE_FMP4` and NOT `HLS_SINGLE_FILE` | URI = `vs->fmp4_init_filename` (default `init.mp4`) | `#EXT-X-MAP:URI="init.mp4"` (or user-configured name) | `[libavformat/hlsenc.c:L1612-L1613]`, `[libavformat/hlsplaylist.c:L137]` |
| `byterange_mode == 1` (single-file mode) at `ff_hls_write_init_file` | `byterange_mode` param non-zero → BYTERANGE clause emitted | `,BYTERANGE="<size>@<offset>"` appended to URI line | `[libavformat/hlsplaylist.c:L138-L140]` |
| `byterange_mode == 0` and fMP4 | No BYTERANGE clause | Plain `#EXT-X-MAP:URI="<filename>"\n` | `[libavformat/hlsplaylist.c:L137,L141]` |
| `i != start_index` (writing non-first segment of window) | `ff_hls_write_init_file` NOT called | Init reference appears only once at top of playlist | `[libavformat/hlsenc.c:L1611]` |

The `vs->fmp4_init_filename` field is initialized from the `hls_fmp4_init_filename` AVOption (default `"init.mp4"` per `[libavformat/hlsenc.c:L3143]`); see [`data-model.md`](data-model.md) for the field's role within `VariantStream`.

---

## Decision — EXT-X-BYTERANGE Emission (Per-Segment)

**Plain-language summary.** When byterange mode is active (`HLS_SINGLE_FILE` set or `max_seg_size > 0`), every `#EXTINF` line in the playlist is followed by `#EXT-X-BYTERANGE:<size>@<offset>`, telling the client which byte slice of the larger file holds this segment. For non-byterange mode, no such line is emitted (each segment is its own file, identified solely by its filename). The emission gate is the `byterange_mode` parameter passed to `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L162-L165]`. The byterange values selected — `size@pos` for full segments versus `video_keyframe_size@video_keyframe_pos` for I-frames-only segments — depend on the `iframe_mode` parameter at the same line.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `byterange_mode == 0` (standard one-segment-per-file mode) | No `#EXT-X-BYTERANGE` emission | Each segment identified by filename only | `[libavformat/hlsplaylist.c:L163]` (negative path) |
| `byterange_mode == 1` and `iframe_mode == 0` (main playlist) | `avio_printf(out, "#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n", size, pos)` | Byte-range covers full segment | `[libavformat/hlsplaylist.c:L163-L165]` |
| `byterange_mode == 1` and `iframe_mode == 1` (I-frames-only playlist) | `avio_printf(out, "#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n", video_keyframe_size, video_keyframe_pos)` | Byte-range covers only the I-frame region of the segment | `[libavformat/hlsplaylist.c:L163-L165]` |
| Emission position in playlist | Always immediately after `#EXTINF`, before `#EXT-X-PROGRAM-DATE-TIME` and before the segment filename line | Order-locked: `#EXTINF` → `#EXT-X-BYTERANGE` → `#EXT-X-PROGRAM-DATE-TIME` → `<filename>` | `[libavformat/hlsplaylist.c:L160-L196]` |
| `HLS_SINGLE_FILE` set, `byterange_mode == 1` | `pos` parameter passed to `ff_hls_write_file_entry` is `en->pos` (offset within single file) | Offset stable across publishes | `[libavformat/hlsenc.c:L1617]` |
| `max_seg_size > 0`, `byterange_mode == 1`, multi-file | `pos` is offset within the current size-capped file | Offset resets at each new file boundary | `[libavformat/hlsenc.c:L1617]` |
| `byterange_mode == 1` and writing fMP4 init via `ff_hls_write_init_file` | Init `EXT-X-MAP` line also emits `,BYTERANGE="<size>@<offset>"` clause | Init segment byterange declared on the `EXT-X-MAP` line | `[libavformat/hlsplaylist.c:L138-L140]` |

The byterange-mode value is computed identically at every call site (`(hls->flags & HLS_SINGLE_FILE) || (hls->max_seg_size > 0)` — see four occurrences at `[libavformat/hlsenc.c:L779,L1049,L1549,L2504]`), so the decision is consistent across the muxer's lifecycle phases.

---

## Decision — EXT-X-ENDLIST Emission

**Plain-language summary.** The `#EXT-X-ENDLIST` tag signals that the playlist is complete — no more segments will be appended. Clients interpret this as VOD or end-of-stream and stop reloading the playlist. The muxer emits it only when (a) `last == 1` is passed to `hls_window` (which happens only from `hls_write_trailer` at `[libavformat/hlsenc.c:L2851,L2855]`, never mid-stream) and (b) the `HLS_OMIT_ENDLIST` flag is not set. Live streams typically omit the tag (so clients keep polling), while VOD streams always emit it; the `HLS_OMIT_ENDLIST` flag exists for the case of intentionally truncated live recordings that should not be marked complete.

The emission is at `[libavformat/hlsenc.c:L1629-L1630]`. The `last` parameter is set to `1` only by `hls_write_trailer` at `[libavformat/hlsenc.c:L2851,L2855]`. The `HLS_OMIT_ENDLIST` flag is at `[libavformat/hlsenc.c:L102]` (`1 << 4`). The tag itself is written by `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]`, which emits the literal string `#EXT-X-ENDLIST\n`.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `last == 0` (any mid-stream `hls_window` call from `hls_write_packet`) | Predicate `last && ...` is false → no emission | Playlist remains "live" — clients continue reloading | `[libavformat/hlsenc.c:L1629]` |
| `last == 1` AND `HLS_OMIT_ENDLIST` flag set | Predicate is false (`(hls->flags & HLS_OMIT_ENDLIST) == 0` is false) → no emission | Live recording ended without `EXT-X-ENDLIST` — clients keep polling | `[libavformat/hlsenc.c:L102,L1629-L1630]` |
| `last == 1` AND `HLS_OMIT_ENDLIST` flag NOT set | `ff_hls_write_end_list(byterange_mode ? hls->m3u8_out : vs->out)` called | `#EXT-X-ENDLIST\n` emitted on last line of playlist | `[libavformat/hlsenc.c:L1629-L1630]`, `[libavformat/hlsplaylist.c:L201-L206]` |
| `last == 1` AND `HLS_OMIT_ENDLIST` NOT set AND `vs->vtt_m3u8_name` set (subtitle variant exists) | `ff_hls_write_end_list(hls->sub_m3u8_out)` ALSO called | Subtitle playlist also receives `#EXT-X-ENDLIST` | `[libavformat/hlsenc.c:L1651-L1652]` |
| Trailer's first `hls_window(s, 1, vs)` returns AVERROR_EXIT | Retry with `hls_window(s, 1, vs)` | Trailer attempts endlist publication twice in error path | `[libavformat/hlsenc.c:L2851-L2856]` |

The `last` parameter is `int`-typed throughout — `0` for in-progress, `1` for finalize. There is no intermediate value. The `HLS_OMIT_ENDLIST` flag is also tested by `hls_write_trailer` at `[libavformat/hlsenc.c:L2851]` to decide whether to emit the master-playlist endlist if applicable. See [`pipeline-orchestration.md`](pipeline-orchestration.md) for the trailer's full call sequence.

---

## Decision — `hls_init_time` Activation

**Plain-language summary.** The `hls_init_time` AVOption (default `0`, meaning disabled) lets the user request that the first several segments be shorter than `hls_time`, so that playback begins faster (the player needs at least 3 segments to start playing in most HLS implementations, so shorter initial segments translate directly to lower latency). After the initial list fills, the muxer reverts to the steady-state `hls_time` cadence. The activation predicate at `[libavformat/hlsenc.c:L2455]` is `vs->sequence - vs->nb_entries > hls->start_sequence && hls->init_time > 0`. The end-of-init transition rewrites `hls->recording_time = hls->time` at `[libavformat/hlsenc.c:L2459]` so subsequent cuts use the steady cadence.

The `hls_init_time` AVOption is at `[libavformat/hlsenc.c:L3124]` (option name `"hls_init_time"`, `.i64 = 0` default). The startup-time selection of `recording_time` is at `[libavformat/hlsenc.c:L2951]`: `hls->recording_time = hls->init_time && hls->max_nb_segments > 0 ? hls->init_time : hls->time;` — so `init_time` is only initially honored when `max_nb_segments > 0` (a sliding window is in effect).

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `hls->init_time == 0` (default, disabled) | At `hls_init`: `hls->recording_time = hls->time` (else branch of ternary) | All segments use `hls_time` cadence; no fast-start | `[libavformat/hlsenc.c:L2951,L3124]` |
| `hls->init_time > 0` AND `hls->max_nb_segments > 0` (sliding window with init_time) | At `hls_init`: `hls->recording_time = hls->init_time` (then branch) | Initial cuts use `hls_init_time` (shorter) until window fills | `[libavformat/hlsenc.c:L2951]` |
| `hls->init_time > 0` AND `hls->max_nb_segments == 0` (unbounded list) | At `hls_init`: `hls->recording_time = hls->time` (then branch of `&&` fails) | `init_time` ignored for unbounded list mode | `[libavformat/hlsenc.c:L2951]` |
| In `hls_write_packet`: `vs->sequence - vs->nb_entries > hls->start_sequence` (window has filled) AND `hls->init_time > 0` | `init_list_dur = hls->init_time * vs->nb_entries`; `after_init_list_dur = (...) * hls->time`; `hls->recording_time = hls->time`; `end_pts = init_list_dur + after_init_list_dur` | Transition to steady-state cadence; `end_pts` computed as init-window time + post-init time | `[libavformat/hlsenc.c:L2455-L2461]` |
| In `hls_write_packet`: window has NOT filled (initial segments still being produced) | Standard `end_pts = hls->recording_time * vs->number` (using `init_time`) | Shorter cuts continue | `[libavformat/hlsenc.c:L2453]` |
| `HLS_APPEND_LIST` flag set AND `hls->init_time > 0` | At `hls_init`: `AV_LOG_WARNING` "append_list mode does not support hls_init_time"; `hls->init_time = 0`; `hls->recording_time = hls->time` | `init_time` forcibly disabled; warning logged | `[libavformat/hlsenc.c:L3103-L3108]` |

The dual computation at `[libavformat/hlsenc.c:L2457-L2461]` — `init_list_dur + after_init_list_dur` — ensures monotonic, continuous `end_pts` advancement across the transition from init mode to steady-state mode (no time discontinuity at the boundary).

---

## Decision — `append_list` Mode (Disables `init_time`)

**Plain-language summary.** When `hls_flags=append_list` is set, the muxer parses an existing `.m3u8` playlist on startup and resumes from where it left off, rather than starting a fresh playlist. This is used to recover from a previous run's `hls_write_trailer` failure or to splice new segments into a long-running recording. Append mode triggers four side effects: it calls `parse_playlist` to repopulate `vs->segments` from the existing M3U8; it forces `vs->discontinuity = 1` so the next segment is marked discontinuous (because PTS continuity is unverifiable across a restart); it disables `hls_init_time` (existing list already has steady-state segments; shorter new segments would mismatch); and it logs a warning if `hls_init_time` was set.

The flag is `HLS_APPEND_LIST = (1 << 6)` at `[libavformat/hlsenc.c:L104]`. The behavior block is at `[libavformat/hlsenc.c:L3100-L3108]`. Note that this block executes inside the per-variant init loop in `hls_write_header` (around L3081-L3115), so each variant stream independently parses its own existing playlist.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| `HLS_APPEND_LIST` flag NOT set (default) | Standard start — fresh playlist; `vs->segments` empty; `vs->discontinuity = 0` | New M3U8 from `start_sequence` (default 0) | `[libavformat/hlsenc.c:L104]` |
| `HLS_APPEND_LIST` set | `parse_playlist(s, vs->m3u8_name, vs)` called | Existing M3U8 read; `vs->segments` repopulated; `vs->sequence` set from existing `#EXT-X-MEDIA-SEQUENCE + N` | `[libavformat/hlsenc.c:L3100-L3101]` |
| `HLS_APPEND_LIST` set | `vs->discontinuity = 1` set unconditionally | Next appended segment carries `discont = 1` → emits `#EXT-X-DISCONTINUITY` before its entry | `[libavformat/hlsenc.c:L3102]` |
| `HLS_APPEND_LIST` set AND `hls->init_time > 0` | `AV_LOG_WARNING` logged: "append_list mode does not support hls_init_time, hls_init_time value will have no effect" | User informed that `init_time` is being overridden | `[libavformat/hlsenc.c:L3103-L3105]` |
| `HLS_APPEND_LIST` set AND `hls->init_time > 0` | `hls->init_time = 0` and `hls->recording_time = hls->time` | `init_time` forcibly disabled; cadence reverts to `hls_time` | `[libavformat/hlsenc.c:L3106-L3107]` |
| `HLS_APPEND_LIST` set AND `hls->init_time == 0` | Predicate fails — no warning, no reassignment | No-op for `init_time` (already disabled) | `[libavformat/hlsenc.c:L3103]` |
| `HLS_APPEND_LIST` set AND existing M3U8 has `#EXT-X-PROGRAM-DATE-TIME` entries | `parse_playlist` populates `discont_program_date_time` accumulator carried into the next segment via `vs->last_segment->discont_program_date_time` | New segment's PDT re-anchored to existing playlist's last value, not to current `av_gettime` | `[libavformat/hlsenc.c:L1273-L1276]` |
| `HLS_APPEND_LIST` set AND `parse_playlist` fails (file missing, malformed) | `parse_playlist` returns AVERROR; appears to be ignored (no `if (ret < 0) goto fail`) | Append-list start may proceed with empty segment list — caller-side validation expected | `[libavformat/hlsenc.c:L3101]` (no error check shown) |

The unconditional `vs->discontinuity = 1` at `[libavformat/hlsenc.c:L3102]` is the safety mechanism: PTS continuity cannot be guaranteed across an HLS muxer restart, so the very next segment is always discontinuous. This is a deliberate over-conservative choice — a true continuous append would require the user to ensure PTS continuity externally, which the muxer cannot verify.

---

## Decision — Demuxer Key Type Resolution

**Plain-language summary.** The HLS demuxer (`libavformat/hls.c`) classifies every segment's encryption by parsing the `METHOD` attribute of `#EXT-X-KEY` tags in the playlist. Three states are possible: `KEY_NONE` (no encryption, default); `KEY_AES_128` (whole-segment AES-128-CBC encryption — every byte of the segment is encrypted with a single key/IV pair); `KEY_SAMPLE_AES` (per-sample encryption inside the TS payload, used for HLS Sample Encryption with MPEG-TS containers). The classification is per-segment because a single playlist can rotate keys mid-stream — a new `#EXT-X-KEY` line applies to all subsequent segments until another `#EXT-X-KEY` is encountered.

The enum is at `[libavformat/hls.c:L71-L75]`. The parsing site is `[libavformat/hls.c:L867-L880]`, where `#EXT-X-KEY:` lines are recognized, the `METHOD=` attribute is extracted via `ff_parse_key_value(ptr, handle_key_args, &info)`, and the result string is compared against the three known METHOD values.

| Condition | Branch | Resulting Behavior | Source Citation |
|---|---|---|---|
| Playlist contains NO `#EXT-X-KEY` line before a segment | `key_type` remains `KEY_NONE` (initial value at parser scope) | Segment fetched plaintext via standard `AVIO` open | `[libavformat/hls.c:L72,L870]` |
| `#EXT-X-KEY:METHOD=NONE` line encountered | `key_type = KEY_NONE` (explicit reset at L870) | Encryption disabled for subsequent segments | `[libavformat/hls.c:L72,L870]` |
| `#EXT-X-KEY:METHOD=AES-128` line encountered | `strcmp(info.method, "AES-128") == 0` → `key_type = KEY_AES_128` | Subsequent segments wrapped via `crypto:` URL protocol with whole-segment AES-128-CBC | `[libavformat/hls.c:L73,L872-L873]` |
| `#EXT-X-KEY:METHOD=SAMPLE-AES` line encountered | `strcmp(info.method, "SAMPLE-AES") == 0` → `key_type = KEY_SAMPLE_AES` | Per-sample encryption applied inside TS payload via `ff_hls_senc_decrypt_frame` (see `data-model.md` for `HLSCryptoContext` and `hls_sample_encryption.h:L42-L62`) | `[libavformat/hls.c:L74,L874-L875]` |
| `#EXT-X-KEY:METHOD=<unknown>` line encountered | Neither `strcmp` matches → `key_type` remains at its prior assignment (KEY_NONE from L870 reset) | Effectively treated as `KEY_NONE`; playback likely fails downstream | `[libavformat/hls.c:L870,L872-L875]` |
| `#EXT-X-KEY` line includes `IV=0x<hex>` | `ff_hex_to_data(iv, info.iv + 2)` populates `iv[16]`; `has_iv = 1` | Explicit IV used for decryption | `[libavformat/hls.c:L876-L879]` |
| `#EXT-X-KEY` line omits `IV=` | `has_iv` remains 0; IV derived from segment sequence number per RFC 8216 | Implicit big-endian sequence number IV | `[libavformat/hls.c:L876]` (negative path) |
| Multiple `#EXT-X-KEY` lines in playlist | Each line resets `key_type` at L870 then re-evaluates | Mid-playlist key rotation — each segment carries the key state in effect when its preceding `#EXT-X-KEY` was parsed | `[libavformat/hls.c:L867-L880]` |
| Segment's `key_type` stored in `struct segment::key_type` (per-segment, immutable post-parse) | At segment fetch, `KEY_NONE` → direct AVIO open; `KEY_AES_128` → `crypto:` wrapper; `KEY_SAMPLE_AES` → post-mux decryption | Decryption path is selected per-segment, not per-playlist | `[libavformat/hls.c:L83]` |

The parser's per-segment behavior is critical: `key_type` is captured into each `struct segment` at the moment that segment's URL is added to the playlist's segment list. Subsequent `#EXT-X-KEY` lines mutate the parser-local `key_type` variable but do not retroactively rewrite previously-captured segments. This matches RFC 8216's semantics, where each `#EXT-X-KEY` applies to subsequent segments until superseded.

---

## Cross-References

- **Enum value definitions and struct field semantics** — see [`data-model.md`](data-model.md):
  - `HLSFlags` enum (full table of 15 flags with bit positions): `data-model.md` §HLSFlags
  - `SegmentType` enum (`SEGMENT_TYPE_MPEGTS`, `SEGMENT_TYPE_FMP4`): `data-model.md` §SegmentType
  - `StartSequenceSourceType` enum (4 modes): `data-model.md` §StartSequenceSourceType
  - `PlaylistType` enum (muxer side: NONE/EVENT/VOD): `data-model.md` §PlaylistType (muxer-side)
  - Demuxer `KeyType` enum (KEY_NONE/KEY_AES_128/KEY_SAMPLE_AES): `data-model.md` §KeyType (demuxer)
  - `HLSContext`, `VariantStream`, `HLSSegment` field-by-field dictionaries
  - Demuxer `HLSContext`, `playlist`, `segment`, `variant`, `rendition` dictionaries

- **Process flow diagrams** that visualize how these decisions chain together — see `process-flows.md`:
  - Segment generation flow (incorporates segment-cut, second-level templating, encryption start)
  - Playlist update flow (incorporates version negotiation, target-duration, all EXT-X-* emissions)
  - Live sliding window flow (incorporates `hls_init_time` activation, `HLS_DELETE_SEGMENTS`)
  - Encryption key rotation flow (incorporates periodic rekey cadence)

- **Lifecycle and callback chain** that drives when these decisions fire — see `pipeline-orchestration.md`:
  - Phase init → `hls_init` runs version cascade, start-sequence resolution, init_time selection
  - Phase write_header → calls `hls_mux_init` for each variant
  - Phase write_packet loop → segment-cut decision evaluated per packet
  - Phase write_trailer → invokes `hls_window` with `last = 1` for endlist emission

- **External interface contracts** that the decisions in this document drive — see [`integration-interfaces.md`](integration-interfaces.md):
  - HTTP method selection feeds the HTTP backend
  - Key URI fetch driven by `KEY_AES_128` / `KEY_SAMPLE_AES`
  - fMP4 init segment delivered when `SEGMENT_TYPE_FMP4`

- **"MUST" / "MUST NOT" zero-deviation companion specification** — see [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md):
  - "MUST emit `EXTM3U` on line 1, `EXT-X-VERSION` on line 2"
  - "MUST set version 7 when `SEGMENT_TYPE_FMP4`"
  - "MUST NOT emit `EXT-X-INDEPENDENT-SEGMENTS` for audio-only variants"
  - "MUST place `EXT-X-DISCONTINUITY` immediately before the discontinuous segment's `EXTINF`"
  - "MUST reset `sequence = 0` when byterange mode is activated"
  - "MUST emit `EXT-X-ENDLIST` only when `last == 1` and `HLS_OMIT_ENDLIST` is clear"
  - "MUST recompute `EXT-X-PROGRAM-DATE-TIME` from system time on discontinuity"

- **Wire-format and binary-layout contracts** — see [`../api-contracts/data-contracts.md`](../api-contracts/data-contracts.md):
  - AVOption types/defaults/bounds for every option referenced in this document
  - Byterange offset/size binary representation (`int64_t @ int64_t`)
  - Encryption IV derivation (when explicit vs random-seed-derived)
  - MPEG-TS HLS Sample Encryption stream type constants

- **Processing-order constraints** that this document's decisions must respect — see [`../api-contracts/timing-dependencies.md`](../api-contracts/timing-dependencies.md):
  - Keyframe detection MUST precede segment-cut decision
  - Segment file MUST be fully written before playlist update
  - `EXT-X-TARGETDURATION` MUST be computed before first playlist publish
  - AES-128 key MUST be installed before encrypted segment write begins

- **External-system contracts** that bind downstream consumers — see [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md):
  - AES-128 key URI fetch protocol
  - HTTP `PUT` chunked-transfer behavior
  - `EXT-X-MEDIA` audio/subtitle rendition format
  - HLS Sample Encryption stream-type wire values


