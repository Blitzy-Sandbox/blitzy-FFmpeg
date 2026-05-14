# Functional Invariants — HLS Pipeline API Contracts

> **Commit Anchor:** All source references in this document are anchored to commit `566ad786` (full hash `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). Line numbers cited as `[<path>:L<start>-L<end>]` are valid at this commit. See [`../README.md`](../README.md) for the documentation-set-wide commit-anchor convention and citation format.

---

## Overview

**Plain-language summary.** This document is the **zero-deviation behavior checklist** for the FFmpeg HLS muxer and demuxer. Every item below states a behavior that the existing pipeline guarantees — that is, a behavior some downstream consumer (an HLS-compliant media player, a CDN ingest endpoint, a sibling muxer that reuses HLS's playlist writers, or a regression test) depends on observing. A port, rewrite, or extension that violates any one of these invariants will break at least one external consumer, sometimes silently. The list is intended to double as a regression-testing checklist: any reviewer of a port can walk this document top to bottom and confirm each MUST/MUST NOT clause against the new implementation.

The format is uniform across the twenty-one invariants in this document. Each `## Invariant:` section opens with a plain-language **MUST** or **MUST NOT** statement (one or two sentences, no FFmpeg jargon), then a `### Technical detail` subsection with three labeled paragraphs: **Enforcement site** (the source line or lines that implement the invariant), **Mechanism** (how the code enforces it — sequential assignment, conditional, format-string convention), and **Violation impact** (what a port that breaks this invariant produces — a malformed playlist, a player rejection, a downstream parser failure).

**Reading guide.** Library integrators who only need to know *what* is guaranteed can read the MUST/MUST NOT statements and skip the technical-detail subsections. Engineers porting or refactoring the HLS pipeline should read both halves of each invariant — the MUST statement names the obligation, and the technical detail tells them which line of code currently fulfills it, so a faithful port can identify the equivalent enforcement site in the new implementation. Reviewers checking conformance can walk the `## Validation Checklist` at the bottom of this document as a single-page summary.

Companion references:

- [`./data-contracts.md`](data-contracts.md) — the data-shape contracts that several invariants below refer to (option types, struct layouts, timestamp unit conventions).
- [`./integration-contracts.md`](integration-contracts.md) — the external-system contracts (HTTP, AES-128, Sample Encryption, EXT-X-MEDIA rendition shape) that some of the invariants below are interpreted by.
- [`./timing-dependencies.md`](timing-dependencies.md) — the ordering contracts that several invariants below depend on (e.g., target-duration computed before first publish, segment file flushed before playlist update).
- [`../technical/codec-logic.md`](../technical/codec-logic.md) — the decision tables that implement the conditional emission rules referenced in many of the invariants.
- [`../technical/data-model.md`](../technical/data-model.md) — the canonical struct dictionary for every field referenced below.

The invariants are presented in the order specified by the agent prompt skeleton. The order is grouped roughly by topic: M3U8 wire-format invariants first (header order, version negotiation, segment naming, tag placement), then meta-muxer semantic invariants (PTS/DTS passthrough, extradata), then per-tag conditional-emission invariants, then registration-flag invariants, and finally a monotonicity invariant on `EXT-X-MEDIA-SEQUENCE`.

---

## Invariant: M3U8 Header Order

**MUST.** Every M3U8 playlist file produced by the HLS muxer MUST begin with the literal line `#EXTM3U` on line 1, followed immediately by `#EXT-X-VERSION:<n>` on line 2 — in that exact order and with no intervening content (no blank line, no comment, no other tag).

### Technical detail

**Enforcement site.** The function `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` emits exactly two lines, in order: `avio_printf(out, "#EXTM3U\n");` at `[libavformat/hlsplaylist.c:L36]` followed by `avio_printf(out, "#EXT-X-VERSION:%d\n", version);` at `[libavformat/hlsplaylist.c:L37]`. This function is invoked as the **first** action of `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L116]`, so every variant playlist begins with these two lines before any other header tag is written. The master playlist also calls `ff_hls_write_playlist_version` directly before writing any other tag (see `create_master_playlist` in `libavformat/hlsenc.c`).

**Mechanism.** Sequential `avio_printf` calls inside a single function with no conditional gating — the two emissions are always in the order shown, on every playlist write. There is no flag or option that suppresses either line or reorders them.

**Violation impact.** Per RFC 8216 §4.3.1.1, an HLS-compliant player MUST treat any playlist whose first line is not `#EXTM3U` as malformed. A port that reorders these two lines, that omits `#EXTM3U`, or that inserts another tag between them, produces playlists that compliant players reject outright — typically the player surface error is "not a valid M3U8 playlist" and stream playback never begins.

---

## Invariant: EXT-X-VERSION Negotiation

**MUST.** The integer `<n>` emitted in `#EXT-X-VERSION:<n>` MUST be computed by the cascading ladder in `hls_window` and MUST be one of exactly five values: `2`, `3`, `4`, `6`, or `7`. No other value is produced by the existing code, and a port MUST NOT emit any other value.

**MUST.** The ladder MUST be evaluated in the order encoded in `hls_window` so that feature-enabling conditions only raise the version, never lower it. Specifically: float durations (i.e., `HLS_ROUND_DURATIONS` NOT set) require version 3; byterange mode requires version 4; `HLS_I_FRAMES_ONLY` requires version 4; `HLS_INDEPENDENT_SEGMENTS` requires version 6; `SEGMENT_TYPE_FMP4` requires version 7.

### Technical detail

**Enforcement site.** The ladder is implemented as serial assignments at `[libavformat/hlsenc.c:L1551-L1571]` inside `hls_window`:

- `[libavformat/hlsenc.c:L1551]` — `hls->version = 2;` (initial value)
- `[libavformat/hlsenc.c:L1552-L1554]` — `if (!(hls->flags & HLS_ROUND_DURATIONS))` raises to `3`
- `[libavformat/hlsenc.c:L1556-L1559]` — `if (byterange_mode)` raises to `4` and additionally resets `sequence = 0`
- `[libavformat/hlsenc.c:L1561-L1563]` — `if (hls->flags & HLS_I_FRAMES_ONLY)` raises to `4`
- `[libavformat/hlsenc.c:L1565-L1567]` — `if (hls->flags & HLS_INDEPENDENT_SEGMENTS)` raises to `6`
- `[libavformat/hlsenc.c:L1569-L1571]` — `if (hls->segment_type == SEGMENT_TYPE_FMP4)` raises to `7`

The `byterange_mode` predicate is `(hls->flags & HLS_SINGLE_FILE) || (hls->max_seg_size > 0)`, defined at `[libavformat/hlsenc.c:L1549]`. The final `hls->version` value is passed to `ff_hls_write_playlist_header` at `[libavformat/hlsenc.c:L1590-L1591]`, which forwards it to `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L116]`.

**Mechanism.** Serial writes — each later block can only overwrite the version with an equal or higher value because the constants are monotonically non-decreasing in the order they appear (2 → 3 → 4 → 4 → 6 → 7). The ladder is order-independent in the sense that all conditions are evaluated unconditionally on each pass; it is order-dependent in the sense that the final value is the value of the last assignment whose condition fires. Because the constants are non-decreasing, the final value equals the maximum of all applicable constants. A port that re-orders the blocks would change the version only if it places a lower constant after a higher one (e.g., the `version = 3` block after the `version = 7` block), which would silently downgrade fMP4 playlists to version 3 — a wire-format violation.

**Violation impact.** A player negotiating against a version lower than what the playlist features require will reject the playlist. Specifically: `EXT-X-MAP` requires version ≥ 6 (or ≥ 7 for fMP4); `EXT-X-BYTERANGE` requires version ≥ 4; `EXT-X-I-FRAMES-ONLY` requires version ≥ 4; fMP4 segments require version ≥ 7. Emitting a version lower than these thresholds, while still emitting the tags that require them, produces playlists that a compliant player will refuse to play or will misinterpret. Emitting a version higher than necessary is benign but unnecessarily restrictive.

---

## Invariant: Segment Naming Convention

**MUST.** Segment filenames MUST follow the printf-style pattern `<basename>_<N>.<ext>`, where `<basename>` is the user-supplied filename stem (default `live` for streamed playlists; user-supplied via `hls_segment_filename`), `<N>` is the segment sequence number rendered by the `POSTFIX_PATTERN "_%d"` macro, and `<ext>` is the segment file extension (`.ts` for MPEG-TS, `.m4s` for fMP4).

**MUST NOT.** A port MUST NOT change the literal value of `POSTFIX_PATTERN` from `"_%d"` to any other format string (e.g., `"-%d"`, `"_%05d"`, `"_seg%d"`), nor remove the underscore separator, nor pad the sequence number to a fixed width.

### Technical detail

**Enforcement site.** The macro `#define POSTFIX_PATTERN "_%d"` is defined at `[libavformat/hlsenc.c:L74]`. It is consumed by filename construction logic in `hls_start` and related functions when expanding the user's `hls_segment_filename` template into per-segment paths. The `%d` format specifier produces a decimal integer with no padding, no leading zeros, and no fixed width.

**Mechanism.** Preprocessor macro substituted at compile time. The macro is a single literal string with two characters (`_` followed by `%d` consumed by `snprintf`/`av_strlcatf`-style printf chains).

**Violation impact.** External tooling that parses segment filenames to recover the sequence number relies on the exact `_<N>.<ext>` shape — e.g., a CDN log-analysis script that groups segments by underscore-prefixed sequence, or a player that reads the on-disk filename to recover stream-position information when the M3U8 is truncated. A port that emits filenames as, for example, `live-0.ts`, `live-1.ts` (with a dash instead of an underscore) would break any consumer relying on the underscore convention. Zero-padding the integer (`live_00000.ts`) would change the lexicographic-order property of the filenames; existing tooling that relies on lexicographic order matching numeric order would silently mis-order segments at the 10-segment boundary, the 100-segment boundary, and so on.

---

## Invariant: EXT-X-DISCONTINUITY Placement

**MUST.** `#EXT-X-DISCONTINUITY` MUST appear in exactly two and only two positions within an M3U8 playlist:

1. **Sequence-start position** — immediately after the `#EXT-X-MEDIA-SEQUENCE` line in the playlist header, before any `#EXTINF` segment-entry line, when the `HLS_DISCONT_START` flag is set, the playlist is being published for the first time (i.e., `sequence == hls->start_sequence`), and `vs->discontinuity_set` is `0`.
2. **Per-segment position** — immediately before the `#EXTINF:<duration>` line of an affected segment entry, when the segment's `HLSSegment::discont` field is `1`.

**MUST NOT.** `#EXT-X-DISCONTINUITY` MUST NOT appear in any other position within the playlist, and MUST NOT appear at all unless one of the two conditions above holds.

### Technical detail

**Enforcement site — sequence-start position.** At `[libavformat/hlsenc.c:L1593-L1596]`, after `ff_hls_write_playlist_header` returns, the muxer gates an `avio_printf` of `"#EXT-X-DISCONTINUITY\n"` on the conjunction `(hls->flags & HLS_DISCONT_START) && sequence==hls->start_sequence && vs->discontinuity_set==0` and, after the emission, sets `vs->discontinuity_set = 1` so subsequent publishes do not re-emit at sequence-start.

**Enforcement site — per-segment position.** Inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L156-L158]`, the writer emits `"#EXT-X-DISCONTINUITY\n"` when its `insert_discont` argument is non-zero, then unconditionally continues to the `#EXTINF` line at `[libavformat/hlsplaylist.c:L160]` or `[libavformat/hlsplaylist.c:L162]`. The `insert_discont` argument is supplied by the caller from `en->discont` at `[libavformat/hlsenc.c:L1616]`. The `en->discont` flag is set in `hls_append_segment` at `[libavformat/hlsenc.c:L1090-L1093]`, which copies `vs->discontinuity` into `en->discont` and then clears `vs->discontinuity` so the flag is consumed exactly once.

**Mechanism.** Two distinct conditional-emission sites, each gated by a different variable. The two sites are mutually independent — a single playlist can carry both a sequence-start discontinuity and one or more per-segment discontinuities in the same publish. The `discont_program_date_time` field at `[libavformat/hlsenc.c:L1273-L1275]` propagates a wall-clock anchor across a discontinuity boundary when `HLS_PROGRAM_DATE_TIME` is active.

**Violation impact.** HLS players use `#EXT-X-DISCONTINUITY` to invalidate their decoder state at the segment boundary it precedes — they flush the audio/video decoders, reset timestamp anchors, and re-establish PSI/PMT (for MPEG-TS) or `moov` (for fMP4). A misplaced or missing `#EXT-X-DISCONTINUITY` causes the player to either (a) reset decoder state where no discontinuity exists, producing a visible jump or audio glitch, or (b) fail to reset where a discontinuity does exist, producing decode errors, frozen video, or PTS/DTS divergence between the muxer and the player. A port that emits `#EXT-X-DISCONTINUITY` after the affected segment's `#EXTINF` line, instead of before it, anchors the decoder reset to the wrong segment.

---

## Invariant: PTS/DTS Passthrough Semantics

**MUST NOT.** The HLS muxer MUST NOT rewrite, remap, normalize, or otherwise modify the `pkt->pts`, `pkt->dts`, or `pkt->duration` fields of any `AVPacket` it forwards to its child sub-muxer. Timestamp semantics are owned exclusively by the child format (MPEG-TS muxer for `SEGMENT_TYPE_MPEGTS`, fMP4 muxer for `SEGMENT_TYPE_FMP4`).

### Technical detail

**Enforcement site.** The HLS muxer is a **meta-muxer** that wraps a child `AVFormatContext *avf` per `VariantStream`. The field is declared at `[libavformat/hlsenc.c:L133]`. In `hls_write_packet`, the packet is forwarded to the child via `ff_write_chained` at `[libavformat/hlsenc.c:L2679]`; the call takes the original `pkt` pointer with its timestamp fields unmodified.

**Mechanism.** `ff_write_chained` is FFmpeg's standard mechanism for passing a packet from a parent format context to a child format context without timestamp rewrites — it performs unit-conversion and stream-index remapping but does not alter `pts`/`dts`/`duration` semantics. The HLS muxer's `hls_write_packet` does not call `av_packet_rescale_ts` or otherwise modify the packet's timestamp fields before the `ff_write_chained` call. The only timestamp-related work `hls_write_packet` performs is tracking `vs->start_pts` and `vs->end_pts` for **segment-cut decision logic** — these values are read-only consumers of `pkt->pts`, not writers of it.

**Violation impact.** A port that rewrites `pkt->pts` or `pkt->dts` before forwarding to the child sub-muxer would change the synchronization contract between audio and video streams in the resulting TS or fMP4 segment. Downstream consumers (HLS players, transcoders, ID3-based analytics) would observe drift between audio and video, gaps or overlaps at segment boundaries, or — for ID3-timestamped audio-only streams — wall-clock-misaligned events. The cumulative effect over a long live stream can be tens of seconds of A/V drift.

---

## Invariant: Extradata Injection (AVFMT_GLOBALHEADER)

**MUST.** Callers (encoders feeding the HLS muxer) MUST populate `AVCodecParameters::extradata` and `AVCodecParameters::extradata_size` for every stream before passing the streams to the HLS muxer. The HLS muxer declares this requirement via the `AVFMT_GLOBALHEADER` flag on its `AVOutputFormat` registration.

### Technical detail

**Enforcement site.** The `AVFMT_GLOBALHEADER` flag is set in the muxer's flag bitfield at `[libavformat/hlsenc.c:L3198]` as part of the literal `FFOutputFormat ff_hls_muxer` initializer: `.p.flags = AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS`.

**Mechanism.** `AVFMT_GLOBALHEADER` is an `AVOutputFormat::flags` bit defined in the public FFmpeg API. The FFmpeg framework propagates this requirement to the encoder layer via the `AV_CODEC_FLAG_GLOBAL_HEADER` codec-flag bit: when an encoder is bound to an output context whose format carries `AVFMT_GLOBALHEADER`, the encoder is expected to emit codec parameters (sequence parameter set, picture parameter set, AAC AudioSpecificConfig, etc.) as `extradata` on the `AVCodecParameters`, rather than inlining them into the first frame's bitstream. The HLS muxer's child sub-muxer then uses `extradata` to construct the necessary box-level headers (`avcC`, `hvcC`, `esds`, etc. for fMP4; PMT entries for MPEG-TS).

**Violation impact.** A port that drops `AVFMT_GLOBALHEADER` would relax the requirement on the encoder side. Encoders that emit inline headers (i.e., codec parameters embedded in the first frame's bitstream) would produce playable MPEG-TS segments (because TS muxing tolerates inline headers — the PMT can be re-derived from the elementary-stream bytestream) but invalid fMP4 segments. fMP4 requires `avcC`/`hvcC` boxes inside the `moov` of the initialization segment, which the muxer constructs from `extradata`. Without `extradata`, the init segment is malformed and the playlist is unplayable.

---

## Invariant: EXT-X-TARGETDURATION ≥ Max Segment Duration

**MUST.** The integer value emitted as `#EXT-X-TARGETDURATION:<n>` in the playlist header MUST be greater than or equal to the duration of every segment listed in the playlist, where the segment duration is `lrint`'d to the nearest integer second.

### Technical detail

**Enforcement site.** Inside `hls_window` at `[libavformat/hlsenc.c:L1584-L1587]`, the muxer recomputes `target_duration` on every playlist publish by scanning the live segment list:

```c
for (en = vs->segments; en; en = en->next) {
    if (target_duration <= en->duration)
        target_duration = lrint(en->duration);
}
```

The local variable `target_duration` is initialized to `0` at `[libavformat/hlsenc.c:L1535]` (the declaration `int target_duration = 0;`) and is passed to `ff_hls_write_playlist_header` at `[libavformat/hlsenc.c:L1590-L1591]`. The header writer emits the line `#EXT-X-TARGETDURATION:<n>` at `[libavformat/hlsplaylist.c:L120]`.

**Mechanism.** A linear pass over the per-variant `HLSSegment` linked-list (starting from `vs->segments`) tracking the maximum. The comparison `target_duration <= en->duration` uses `<=` rather than `<`, so equal-duration segments still trigger the `lrint` update — this ensures the integer value emitted in the playlist is the `lrint` of the maximum `double` segment duration, not the running `target_duration` from a prior publish.

**Violation impact.** RFC 8216 §4.4.3.1 requires the `EXT-X-TARGETDURATION` value to be greater than or equal to the duration of every segment. A player that observes a violation will either reject the playlist as malformed or, more commonly, will use the (too-small) `EXT-X-TARGETDURATION` as its reload-poll interval, polling the playlist more frequently than the muxer produces new segments — wasting bandwidth and server load without changing the stream's playback experience. Some players will assume each segment is at most `EXT-X-TARGETDURATION` seconds long and will misallocate their internal buffer, producing under-buffering for the longer-than-advertised segments.

---

## Invariant: EXT-X-ENDLIST Emission Condition

**MUST.** `#EXT-X-ENDLIST` MUST be emitted in a playlist if and only if both of the following hold simultaneously:

1. The publish is the **final** playlist write — that is, the `last` argument to `hls_window` is non-zero, which happens exclusively from `hls_write_trailer`.
2. The `HLS_OMIT_ENDLIST` flag is **NOT** set.

**MUST NOT.** `#EXT-X-ENDLIST` MUST NOT appear in any intermediate (live-mode) playlist publish, regardless of flag settings.

### Technical detail

**Enforcement site.** Inside `hls_window` at `[libavformat/hlsenc.c:L1629-L1630]`, the muxer gates the call to `ff_hls_write_end_list` on the conjunction `last && (hls->flags & HLS_OMIT_ENDLIST) == 0`. The writer itself is `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]`, which emits the literal text `"#EXT-X-ENDLIST\n"` at `[libavformat/hlsplaylist.c:L205]`.

**Mechanism.** The `last` argument to `hls_window` is `0` for every call from `hls_write_packet` (the per-segment publish path) and `1` for every call from `hls_write_trailer` (the end-of-stream path). The `HLS_OMIT_ENDLIST` flag is a user-settable bit at `[libavformat/hlsenc.c:L102]` exposed via the `hls_flags` AVOption. The two conditions are conjuncted with `&&`, so if either is false the line is not emitted.

**Violation impact.** Emitting `#EXT-X-ENDLIST` on a live stream's intermediate playlist tells the player "this stream is over — stop polling for more segments." Players that observe `#EXT-X-ENDLIST` will close their reload-poll loop and either stop playback at the last segment or fail to discover newly published segments. A port that omits the `last &&` guard or drops the `HLS_OMIT_ENDLIST` check would convert every live-stream playlist publish into a one-shot VOD playlist from the player's perspective. Conversely, a port that forgets to emit `#EXT-X-ENDLIST` on the final write of a VOD playlist (when `HLS_OMIT_ENDLIST` is not set) leaves the playlist looking live, causing players to poll indefinitely for non-existent updates.

---

## Invariant: EXT-X-INDEPENDENT-SEGMENTS Conditional Emission

**MUST.** `#EXT-X-INDEPENDENT-SEGMENTS` MUST be emitted in a playlist if and only if both of the following hold:

1. The variant has at least one video stream (`vs->has_video` is non-zero).
2. The `HLS_INDEPENDENT_SEGMENTS` flag is set in `hls->flags`.

**MUST NOT.** `#EXT-X-INDEPENDENT-SEGMENTS` MUST NOT be emitted in audio-only or subtitle-only variants, regardless of the flag setting.

### Technical detail

**Enforcement site.** At `[libavformat/hlsenc.c:L1597-L1599]`, the muxer gates the `avio_printf` of `"#EXT-X-INDEPENDENT-SEGMENTS\n"` on the conjunction `vs->has_video && (hls->flags & HLS_INDEPENDENT_SEGMENTS)`. The line is emitted into the variant playlist (or the byterange-mode shared `m3u8_out` AVIOContext) only when both halves are true. The `HLS_INDEPENDENT_SEGMENTS` flag itself is defined at `[libavformat/hlsenc.c:L111]`.

**Mechanism.** A single boolean conjunction inside `hls_window`, evaluated after the header has been written but before the segment-entry loop. The emission is unconditional within the truth of the conjunction — there is no further per-segment gating.

**Cross-invariant side effect.** When this tag is emitted, the EXT-X-VERSION negotiation invariant (above) requires the version to be at least 6 — the cascade at `[libavformat/hlsenc.c:L1565-L1567]` raises `hls->version` to `6` whenever `HLS_INDEPENDENT_SEGMENTS` is set, independently of whether `vs->has_video` is true. This means an audio-only variant with `HLS_INDEPENDENT_SEGMENTS` set will declare version 6 but will not emit the `#EXT-X-INDEPENDENT-SEGMENTS` line — a permitted but unusual configuration.

**Violation impact.** A port that emits `#EXT-X-INDEPENDENT-SEGMENTS` on audio-only or subtitle-only variants does not break compliance per RFC 8216 (the tag is just less meaningful), but causes some players to expend extra effort attempting to validate the per-segment independence claim. A port that omits the tag when both conditions hold loses the player's ability to seek-to-segment without back-references, increasing seek latency for users.

---

## Invariant: EXT-X-I-FRAMES-ONLY Pins Version to 4

**MUST.** When `HLS_I_FRAMES_ONLY` is set, the playlist MUST declare `#EXT-X-VERSION:N` with `N >= 4`. The existing muxer pins this to exactly `4` (which may be raised further by other features in the same ladder).

**MUST.** When `HLS_I_FRAMES_ONLY` is set, the playlist header MUST contain the line `#EXT-X-I-FRAMES-ONLY`.

### Technical detail

**Enforcement site — version pin.** Inside the version-negotiation cascade at `[libavformat/hlsenc.c:L1561-L1563]`, the muxer raises `hls->version` to `4` when `(hls->flags & HLS_I_FRAMES_ONLY)` is non-zero. This is the third block in the cascade; later blocks (`HLS_INDEPENDENT_SEGMENTS` → `6`; `SEGMENT_TYPE_FMP4` → `7`) may raise the version further, so the final value is always `>= 4`.

**Enforcement site — header emission.** The actual `#EXT-X-I-FRAMES-ONLY` line is emitted inside `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L129-L131]`, gated on the `iframe_mode` argument, which the caller supplies as `hls->flags & HLS_I_FRAMES_ONLY` at `[libavformat/hlsenc.c:L1591]`.

**Mechanism.** Two distinct enforcement sites — the version pin at the negotiation cascade and the line emission inside the header writer. Both share the same predicate (`HLS_I_FRAMES_ONLY` flag), so they fire together. The `HLS_I_FRAMES_ONLY` flag itself is defined at `[libavformat/hlsenc.c:L112]`.

**Violation impact.** Per RFC 8216 §4.4.3.6, `EXT-X-I-FRAMES-ONLY` requires version 4 or higher. A port that emits the line at version 2 or 3 produces playlists that compliant players reject. The converse — pinning version to 4 without emitting the line — is silently functional (the playlist works, just at higher-than-needed version), but breaks the symmetry that downstream tooling assumes (i.e., that any version-4 HLS playlist with `EXT-X-I-FRAMES-ONLY` is a trick-play playlist).

---

## Invariant: EXT-X-MAP Emitted Only for fMP4

**MUST.** `#EXT-X-MAP:URI="..."` MUST be emitted in a playlist if and only if `hls->segment_type == SEGMENT_TYPE_FMP4` AND the entry being processed is the **first** segment in the per-variant segment list (i.e., `en == vs->segments`).

**MUST NOT.** `#EXT-X-MAP` MUST NOT be emitted for MPEG-TS segments (TS has no initialization-segment concept — the PAT/PMT is repeated periodically within each segment).

**MUST NOT.** `#EXT-X-MAP` MUST NOT be emitted more than once per playlist (subsequent segment iterations skip it).

### Technical detail

**Enforcement site.** At `[libavformat/hlsenc.c:L1611-L1614]`, the muxer gates the call to `ff_hls_write_init_file` on the conjunction `(hls->segment_type == SEGMENT_TYPE_FMP4) && (en == vs->segments)`. When the conjunction is true, `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]` emits `#EXT-X-MAP:URI="<filename>"` (and an optional `,BYTERANGE="<size>@<offset>"` clause when byterange mode is active). The URI is taken from `vs->fmp4_init_filename` (default `"init.mp4"` — see [`./data-contracts.md`](data-contracts.md)) or, in single-file byterange mode, from `en->filename`.

**Mechanism.** A boolean conjunction inside the per-segment loop in `hls_window`. The second predicate `(en == vs->segments)` is true only on the first iteration of the loop because `vs->segments` is the head pointer of the singly-linked segment list. Subsequent iterations carry `en != vs->segments` and skip the emission.

**Cross-invariant side effect.** The presence of `#EXT-X-MAP` in a playlist also implies the EXT-X-VERSION invariant — version 7 is forced for fMP4 mode (see the fMP4 invariant below at `[libavformat/hlsenc.c:L1569-L1571]`). The combination of `#EXT-X-MAP` and version-7 is what a player uses to recognize an fMP4 playlist.

**Violation impact.** Emitting `#EXT-X-MAP` for MPEG-TS segments would cause a player to attempt to read a non-existent initialization segment, producing a fetch error before any segment is played. Omitting `#EXT-X-MAP` for fMP4 segments leaves the player without the `moov` box, so segment fetches succeed but decoder initialization fails — typically the player reports "no compatible codec" or fails silently with a blank screen. Emitting `#EXT-X-MAP` on every segment iteration (rather than only the first) bloats the playlist and confuses some players' init-segment caching heuristics.

---

## Invariant: EXT-X-BYTERANGE Emitted Only for Byterange Mode

**MUST.** `#EXT-X-BYTERANGE:<size>@<offset>` MUST be emitted in a playlist for a segment entry if and only if `byterange_mode` is true. `byterange_mode` is true if and only if at least one of the following is set: `HLS_SINGLE_FILE` flag, or `hls->max_seg_size > 0`.

### Technical detail

**Enforcement site.** Inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L163-L165]`, the writer emits `"#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n"` when its `byterange_mode` argument is non-zero. The format is `<size>@<offset>` where `<size>` and `<offset>` are both `int64_t` (printed via the `PRId64` format-string macro). The arguments come from `iframe_mode ? video_keyframe_size : size` for the size value and `iframe_mode ? video_keyframe_pos : pos` for the offset — see the `iframe_mode` invariant below.

The `byterange_mode` predicate is computed at three call sites that should evaluate identically:
- `[libavformat/hlsenc.c:L1549]` inside `hls_window` (used to gate the playlist emission)
- `[libavformat/hlsenc.c:L1049]` inside `hls_append_segment`
- `[libavformat/hlsenc.c:L779]` inside `hls_mux_init`
- `[libavformat/hlsenc.c:L2504]` inside `hls_write_packet`

All four sites use the identical formula `(hls->flags & HLS_SINGLE_FILE) || (hls->max_seg_size > 0)`.

**Mechanism.** A single `if` gate inside the per-segment writer. The `byterange_mode` argument is passed through from `hls_window` and reflects the conjunction defined above.

**Cross-invariant side effect.** When `byterange_mode` is true, the EXT-X-VERSION negotiation cascade at `[libavformat/hlsenc.c:L1556-L1559]` raises the version to `4` and additionally resets `sequence = 0` — see the EXT-X-VERSION invariant. Resetting `sequence` to `0` for byterange playlists is unusual (normal live playlists have a monotonically advancing `EXT-X-MEDIA-SEQUENCE`), but it reflects the convention that byterange playlists are inherently single-file and the sequence counter loses its sliding-window semantics.

**Violation impact.** Emitting `#EXT-X-BYTERANGE` without setting either `HLS_SINGLE_FILE` or `max_seg_size > 0` would associate byterange directives with multi-file segments, confusing the player about where each segment lives — the player would attempt to read at the declared offset within the wrong file. Conversely, omitting `#EXT-X-BYTERANGE` in single-file mode leaves the player without offset information, so it cannot locate individual segments within the shared file.

---

## Invariant: EXT-X-KEY METHOD=AES-128 Line Precedes Encrypted Segments

**MUST.** Each encrypted-segment range in the playlist MUST be preceded by an `#EXT-X-KEY:METHOD=AES-128,URI="<uri>"` line (optionally with `,IV=0x<hex>` appended), in that exact format and METHOD value.

**MUST.** The `#EXT-X-KEY` line MUST be re-emitted whenever the per-segment `en->key_uri` differs from the previously-emitted URI for this playlist, or whenever the per-segment `en->iv_string` differs from the previously-emitted IV.

**MUST NOT.** Any METHOD value other than `AES-128` MUST NOT be emitted by the existing muxer's encryption path (the muxer supports only AES-128 in CBC mode for full-segment encryption — Sample Encryption uses a different code path via `libavformat/hls_sample_encryption.c` and is not gated by this invariant).

### Technical detail

**Enforcement site.** Inside the per-segment loop in `hls_window` at `[libavformat/hlsenc.c:L1601-L1609]`, the muxer gates the `avio_printf` calls on the conjunction `(hls->encrypt || hls->key_info_file) && (!key_uri || strcmp(en->key_uri, key_uri) || av_strcasecmp(en->iv_string, iv_string))`. When the conjunction holds, the muxer emits:

- `"#EXT-X-KEY:METHOD=AES-128,URI=\"%s\""` with `en->key_uri` at `[libavformat/hlsenc.c:L1603]` (note: this line carries no trailing newline yet)
- If `*en->iv_string` is non-empty, `",IV=0x%s"` with `en->iv_string` at `[libavformat/hlsenc.c:L1604-L1605]`
- `"\n"` to terminate the line at `[libavformat/hlsenc.c:L1606]`
- Updates the loop-local `key_uri` and `iv_string` trackers at `[libavformat/hlsenc.c:L1607-L1608]` so subsequent segments compare against the just-emitted values

**Mechanism.** Serial `avio_printf` calls inside the per-segment iteration, with stateful tracking via two loop-local pointers (`key_uri`, `iv_string`) that hold the most-recently-emitted values. The `strcmp`/`av_strcasecmp` comparisons detect URI or IV changes between segments, which is the trigger for re-emission. The first segment in the loop always emits because the trackers are initialized to `NULL` (`!key_uri` is true).

**Violation impact.** Emitting a METHOD value other than `AES-128` (e.g., `SAMPLE-AES`, `NONE`) for the muxer's full-segment-encryption code path is a contract violation — a player would either reject the playlist or attempt the wrong decryption algorithm, producing zero-byte or corrupt audio/video output. Forgetting to re-emit the `#EXT-X-KEY` line after a key URI change leaves all subsequent segments referencing the stale key, so the player would attempt to decrypt with the wrong key and fail. Emitting `#EXT-X-KEY` after the `#EXTINF` of the affected segment (rather than before it) anchors the key to the wrong segment range.

---

## Invariant: fMP4 Forces Version 7

**MUST.** When `hls->segment_type == SEGMENT_TYPE_FMP4`, the playlist MUST declare `#EXT-X-VERSION:7`. The existing muxer pins this unconditionally — fMP4 is the **last** assignment in the version-negotiation cascade.

**MUST NOT.** A port MUST NOT emit fMP4 segments with `#EXT-X-VERSION` less than `7`.

### Technical detail

**Enforcement site.** The final block in the version-negotiation cascade at `[libavformat/hlsenc.c:L1569-L1571]` executes `hls->version = 7;` unconditionally when `hls->segment_type == SEGMENT_TYPE_FMP4`. Because this is the last assignment in the cascade (after the `HLS_ROUND_DURATIONS`, `byterange_mode`, `HLS_I_FRAMES_ONLY`, and `HLS_INDEPENDENT_SEGMENTS` blocks), it overrides every prior assignment. A configuration that enables both `HLS_INDEPENDENT_SEGMENTS` (which would set version to 6) and `SEGMENT_TYPE_FMP4` lands at version 7 — fMP4 always wins.

**Mechanism.** Single unconditional assignment inside the cascade. The placement of this block at the end of the cascade is the mechanism — moving it earlier would allow subsequent blocks to override it (but no subsequent block currently exists that would raise the version above 7).

**Violation impact.** Per RFC 8216 §8, fMP4 segments are not valid in HLS playlists below version 7. A port that emits fMP4 segments with `#EXT-X-VERSION:6` (or lower) produces playlists that compliant players reject. The fMP4-specific tags (`#EXT-X-MAP` referencing the initialization segment with `init.mp4` URI) are also unparseable on pre-version-7 players, compounding the failure mode.

---

## Invariant: EXT-X-PLAYLIST-TYPE Conditional Emission

**MUST.** `#EXT-X-PLAYLIST-TYPE:EVENT` MUST be emitted if and only if `hls->pl_type == PLAYLIST_TYPE_EVENT` (set via the `hls_playlist_type` AVOption with value `event`).

**MUST.** `#EXT-X-PLAYLIST-TYPE:VOD` MUST be emitted if and only if `hls->pl_type == PLAYLIST_TYPE_VOD` (set via the `hls_playlist_type` AVOption with value `vod`).

**MUST NOT.** `#EXT-X-PLAYLIST-TYPE` MUST NOT be emitted when `hls->pl_type == PLAYLIST_TYPE_NONE` (the default, indicating live mode with no explicit type declaration).

### Technical detail

**Enforcement site.** Inside `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L124-L128]`:

```c
if (playlist_type == PLAYLIST_TYPE_EVENT) {
    avio_printf(out, "#EXT-X-PLAYLIST-TYPE:EVENT\n");
} else if (playlist_type == PLAYLIST_TYPE_VOD) {
    avio_printf(out, "#EXT-X-PLAYLIST-TYPE:VOD\n");
}
```

The `playlist_type` argument is supplied by the caller from `hls->pl_type` at `[libavformat/hlsenc.c:L1591]`. The `PlaylistType` enum is defined in `[libavformat/hlsplaylist.h]` with values `PLAYLIST_TYPE_NONE` (default, no tag emitted), `PLAYLIST_TYPE_EVENT`, `PLAYLIST_TYPE_VOD`, and `PLAYLIST_TYPE_NB` (sentinel — never emitted).

**Mechanism.** An `if`/`else if` ladder with no terminal `else` — the `PLAYLIST_TYPE_NONE` (and any unrecognized value) falls through both branches without emitting any tag.

**Violation impact.** Emitting `#EXT-X-PLAYLIST-TYPE:VOD` on a live stream tells the player the stream is finite (even before `#EXT-X-ENDLIST` is reached), which causes some players to refuse to seek back to live or to mis-estimate the stream's duration. Emitting `#EXT-X-PLAYLIST-TYPE:EVENT` on an unbounded live stream is the closest tolerable substitute, but it still constrains the player's behavior — `EVENT` means "the playlist may grow but segments are never removed", which is incompatible with `HLS_DELETE_SEGMENTS` mode. Omitting the tag (the default for `PLAYLIST_TYPE_NONE`) is always safe.

---

## Invariant: EXT-X-ALLOW-CACHE Conditional Emission

**MUST.** `#EXT-X-ALLOW-CACHE:<value>` MUST be emitted if and only if the `hls_allow_cache` AVOption is set to exactly `0` or `1`. Any other value (including the default `-1`, which means "user did not set this option") MUST suppress the emission.

**MUST.** When emitted with `allowcache == 0`, the value MUST be the literal string `NO`. When emitted with `allowcache == 1`, the value MUST be the literal string `YES`.

### Technical detail

**Enforcement site.** Inside `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L117-L119]`:

```c
if (allowcache == 0 || allowcache == 1) {
    avio_printf(out, "#EXT-X-ALLOW-CACHE:%s\n", allowcache == 0 ? "NO" : "YES");
}
```

The `allowcache` argument is supplied by the caller from `hls->allowcache` at `[libavformat/hlsenc.c:L1590-L1591]`. The default `-1` value comes from the AVOption table at `[libavformat/hlsenc.c:L3128]` (the row `{"hls_allow_cache", ..., {.i64 = -1}, ...}`).

**Mechanism.** A two-value explicit check using OR. The default value `-1` falls outside the check, so the line is not emitted in the default configuration. Any non-zero, non-one user-provided value (e.g., `2`, `-5`) also suppresses emission — the muxer does not validate the value against the {0, 1} set; it simply skips emission for out-of-range values.

**Violation impact.** `EXT-X-ALLOW-CACHE` was deprecated in HLS version 7 but is still emitted for backward compatibility with old players. A port that emits the tag for any value other than 0 or 1 (e.g., always emitting `EXT-X-ALLOW-CACHE:YES` regardless of user setting) overrides the user's intent. A port that omits the tag entirely (even when the user sets `allowcache=0`) silently loses the user's caching directive.

---

## Invariant: LF Line Terminator on All Playlist Writes

**MUST.** Every line in every M3U8 playlist file written by the HLS muxer MUST be terminated with `\n` (LF, single byte `0x0A`), not `\r\n` (CRLF).

**MUST NOT.** A port MUST NOT introduce `\r\n` line terminators into M3U8 output.

### Technical detail

**Enforcement site.** This invariant is enforced uniformly by every `avio_printf` format string in `[libavformat/hlsplaylist.c]` and the in-line `avio_printf` calls in `[libavformat/hlsenc.c]` that write to the playlist AVIOContext. Every format string ending a line uses `\n` and never `\r\n`. Visible at:

- `[libavformat/hlsplaylist.c:L36-L37]` — `#EXTM3U\n` and `#EXT-X-VERSION:%d\n`
- `[libavformat/hlsplaylist.c:L47-L55]` — audio rendition lines all terminated with `\n` or `\"\n`
- `[libavformat/hlsplaylist.c:L65-L75]` — subtitle rendition lines (same pattern)
- `[libavformat/hlsplaylist.c:L107]` — stream-info closing `"\n%s\n\n"`
- `[libavformat/hlsplaylist.c:L118]` — `#EXT-X-ALLOW-CACHE:%s\n`
- `[libavformat/hlsplaylist.c:L120-L121]` — `#EXT-X-TARGETDURATION:%d\n` and `#EXT-X-MEDIA-SEQUENCE:%"PRId64"\n`
- `[libavformat/hlsplaylist.c:L125-L127]` — `#EXT-X-PLAYLIST-TYPE:EVENT\n` and `#EXT-X-PLAYLIST-TYPE:VOD\n`
- `[libavformat/hlsplaylist.c:L130]` — `#EXT-X-I-FRAMES-ONLY\n`
- `[libavformat/hlsplaylist.c:L141]` — `\n` terminating `#EXT-X-MAP` line
- `[libavformat/hlsplaylist.c:L157]` — `#EXT-X-DISCONTINUITY\n`
- `[libavformat/hlsplaylist.c:L160, L162]` — `#EXTINF:%ld,\n` and `#EXTINF:%f,\n`
- `[libavformat/hlsplaylist.c:L164]` — `#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n`
- `[libavformat/hlsplaylist.c:L191]` — `#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s\n`
- `[libavformat/hlsplaylist.c:L196]` — file URL line `%s\n`
- `[libavformat/hlsplaylist.c:L205]` — `#EXT-X-ENDLIST\n`

The same `\n`-only convention is enforced at the muxer-side `avio_printf` sites in `hls_window`:

- `[libavformat/hlsenc.c:L1594]` — `#EXT-X-DISCONTINUITY\n`
- `[libavformat/hlsenc.c:L1598]` — `#EXT-X-INDEPENDENT-SEGMENTS\n`
- `[libavformat/hlsenc.c:L1603]` — start of `#EXT-X-KEY:METHOD=AES-128,URI=...` (no trailing newline yet)
- `[libavformat/hlsenc.c:L1605]` — `,IV=0x%s` (no trailing newline yet)
- `[libavformat/hlsenc.c:L1606]` — `\n` to close the EXT-X-KEY line

**Mechanism.** Code-review-enforced convention with no central macro. There is no helper function that enforces line termination — each `avio_printf` call independently includes `\n` in its format string. A reviewer of any new line emission must confirm `\n` (not `\r\n`) at the end of the format string.

**Violation impact.** RFC 8216 §4.1 specifies the playlist line-grammar with LF as the explicit line terminator. While many real-world HLS players tolerate CRLF (treating `\r\n` as a single line break), the reference grammar does not, and mixing the two terminators produces parser ambiguity for tags that contain embedded quotes (e.g., `#EXT-X-KEY:URI="..."`) — some lenient parsers split on `\r` inside the quoted URI, producing a malformed parse. The simpler failure mode is on strict parsers (FFmpeg's own HLS demuxer, for example, treats `\r` as part of the content) which would see `\r` as part of the URI string and fail to fetch the resource.

---

## Invariant: Demuxer ID3-Timestamped Streams Use 1/90000 Time Base

**MUST.** When the HLS demuxer detects an ID3 `PRIV` timestamp tag (an apple-defined HLS-ID3 timestamping convention) on a stream, it MUST set the stream's `time_base` to `1/90000` with a 33-bit PTS wrap value.

### Technical detail

**Enforcement site.** Inside `set_stream_info_from_inner_stream` (or equivalent stream-info propagation function) at `[libavformat/hls.c:L2065-L2066]`:

```c
if (pls->is_id3_timestamped) /* custom timestamps via id3 */
    avpriv_set_pts_info(st, 33, 1, MPEG_TIME_BASE);
```

The `MPEG_TIME_BASE` macro is defined at `[libavformat/hls.c:L56]` as `90000`. The `avpriv_set_pts_info` call sets `st->pts_wrap_bits = 33`, `st->time_base.num = 1`, and `st->time_base.den = 90000`. The `is_id3_timestamped` flag on the `playlist` struct is set elsewhere in the demuxer when an ID3 `PRIV` frame with the apple HLS timestamp owner ID is observed.

**Mechanism.** A conditional dispatch with two branches: ID3-timestamped streams use the 90 kHz MPEG-2 time base (matching the in-band timestamps recovered from the ID3 PRIV payload); non-ID3-timestamped streams (the `else` branch at `[libavformat/hls.c:L2067-L2068]`) use the inner stream's native time base. The 33-bit wrap value matches the MPEG-2 33-bit PCR.

**Violation impact.** Using the sub-format's native time base (typically 44100 or 48000 Hz for audio-only HLS streams) for an ID3-timestamped stream produces drifting timestamps because the ID3 PRIV payload encodes the timestamp in 90 kHz units. A port that omits the conditional and always uses the sub-format's time base would observe each ID3 timestamp at an arithmetically scaled value (e.g., a 90000-tick PTS would be interpreted as a 90000-sample PTS at 44100 Hz, which is ~2.04 seconds — off by a factor of 2.04× from the intended 1.0 second). The cumulative drift makes ID3-anchored wall-clock alignment impossible.

---

## Invariant: Demuxer Format Flags Cluster

**MUST.** The HLS demuxer's `FFInputFormat` registration MUST declare exactly these flags: `AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`.

**MUST NOT.** A port MUST NOT add or remove any flag from this set without an explicit contract update.

### Technical detail

**Enforcement site.** At `[libavformat/hls.c:L2904]`, the literal initializer of `const FFInputFormat ff_hls_demuxer` sets `.p.flags = AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`. The full registration block spans `[libavformat/hls.c:L2900-L2912]`.

**Mechanism.** Literal initializer in the demuxer struct. Each flag is a bit defined in `[libavformat/avformat.h]`:

- `AVFMT_NOGENSEARCH` — the demuxer does not support generic (byte-level) searching for sync points; the demuxer's own `read_seek` callback is used.
- `AVFMT_TS_DISCONT` — the demuxer's output stream may contain timestamp discontinuities (across segment boundaries); downstream consumers (e.g., the player or transcoder) must handle resets.
- `AVFMT_NO_BYTE_SEEK` — byte-level seek is not supported; the demuxer's `read_seek` accepts only timestamp-based seeks.
- `AVFMT_SHOW_IDS` — stream IDs (from PMT in MPEG-TS sub-streams) are exposed to callers, which is useful for stream selection and metadata reporting.

**Violation impact.** Dropping `AVFMT_NO_BYTE_SEEK` would allow callers (e.g., `ffmpeg` CLI invoked with `-ss <byte-offset>`) to attempt byte-level seeks via the generic `av_seek_frame` path, which the demuxer does not support — the seek would land at an arbitrary position in an arbitrary segment, producing decoder errors or crashes. Dropping `AVFMT_TS_DISCONT` would cause downstream consumers to assume monotonic timestamps across the entire stream, mis-handling segment-boundary jumps (especially across `#EXT-X-DISCONTINUITY` boundaries). Dropping `AVFMT_NOGENSEARCH` would let the framework attempt generic search at sync points the demuxer cannot interpret. Dropping `AVFMT_SHOW_IDS` would suppress stream-ID reporting, making programmatic stream selection unreliable.

---

## Invariant: Muxer Format Flags Cluster

**MUST.** The HLS muxer's `FFOutputFormat` registration MUST declare exactly these flags: `AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS`.

**MUST NOT.** A port MUST NOT add or remove any flag from this set without an explicit contract update.

### Technical detail

**Enforcement site.** At `[libavformat/hlsenc.c:L3198]`, the literal initializer of `const FFOutputFormat ff_hls_muxer` sets `.p.flags = AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS`. The full registration block spans `[libavformat/hlsenc.c:L3191-L3207]`.

**Mechanism.** Literal initializer in the muxer struct. Each flag is a bit defined in `[libavformat/avformat.h]`:

- `AVFMT_NOFILE` — the muxer does not require the framework to open a top-level file at the playlist URL; instead, the muxer opens its own `AVIOContext`s for each segment and playlist file via `hlsenc_io_open`. The framework MUST NOT auto-open a file when this flag is set.
- `AVFMT_GLOBALHEADER` — encoders feeding the muxer MUST place codec parameters in `extradata` rather than inline in the bitstream. See the Extradata Injection invariant above.
- `AVFMT_NODIMENSIONS` — the muxer does not require video streams to have valid width/height at registration time; dimensions are derived later from the bitstream (or, for fMP4, from `extradata`).

**Violation impact.** Dropping `AVFMT_NOFILE` would cause the framework to open the playlist URL as a top-level file, which conflicts with the HLS muxer's own AVIOContext management. The framework's auto-opened file would never receive any writes (the muxer writes to its own contexts), but the file handle would consume resources and cause the framework to attempt a flush-and-close at the wrong moment. Dropping `AVFMT_GLOBALHEADER` would lift the extradata requirement on encoders, producing invalid fMP4 segments (see the Extradata Injection invariant). Dropping `AVFMT_NODIMENSIONS` would cause the framework to reject streams with missing dimensions at registration, breaking real-world workflows where dimensions are only known after decoding the first frame.

---

## Invariant: EXT-X-MEDIA-SEQUENCE Monotonic

**MUST.** The `<n>` value emitted in successive `#EXT-X-MEDIA-SEQUENCE:<n>` lines for the same variant playlist MUST be monotonically non-decreasing across successive publishes of a live playlist. For a VOD playlist published once, the value is constant.

### Technical detail

**Enforcement site.** The `sequence` value emitted in the playlist header is computed in `hls_window` at `[libavformat/hlsenc.c:L1539]`:

```c
int64_t sequence = FFMAX(hls->start_sequence, vs->sequence - vs->nb_entries);
```

The local `sequence` variable is then passed to `ff_hls_write_playlist_header` at `[libavformat/hlsenc.c:L1590-L1591]`, which emits the line at `[libavformat/hlsplaylist.c:L121]`. When `byterange_mode` is true, the version-negotiation cascade additionally resets `sequence = 0` at `[libavformat/hlsenc.c:L1558]` (byterange mode is single-file and not a live sliding window).

**Mechanism.** The visible `sequence` value is the MAX of two monotonically advancing counters:

1. `hls->start_sequence` is the initial sequence value, set once at muxer-init time from the `hls_start_number_source` option (one of `HLS_START_SEQUENCE_AS_START_NUMBER`, `_AS_SECONDS_SINCE_EPOCH`, `_AS_FORMATTED_DATETIME`, `_AS_MICROSECONDS_SINCE_EPOCH`). It does not change after init.
2. `vs->sequence - vs->nb_entries` is the sequence number of the OLDEST segment currently in the live window. `vs->sequence` advances by 1 each time `hls_append_segment` adds a new segment; `vs->nb_entries` is the number of segments currently in the list. As segments are evicted by the sliding-window mechanism (in `hls_delete_old_segments`), `vs->nb_entries` decreases by 1 while `vs->sequence` continues to advance — so the difference `vs->sequence - vs->nb_entries` advances by 1 each time a segment is evicted.

Both counters advance monotonically, and their MAX advances monotonically. The reset to `0` in byterange mode is not a "decrease" of a live sequence — byterange mode is a one-shot publish, and the reset happens once before the first publish, not between publishes.

**Violation impact.** A player polling a live playlist tracks the `EXT-X-MEDIA-SEQUENCE` to detect new segments. If the value ever decreases between two successive polls, the player assumes a new stream session has begun — it discards its current buffer, resets its decoder, and restarts playback from the new sequence base. This produces a visible playback glitch (rebuffer + position jump). Players that re-implement this assumption strictly may refuse to play a playlist with a decreasing sequence. A port that does not preserve the MAX semantic would, for example, emit `EXT-X-MEDIA-SEQUENCE` based on `vs->sequence - vs->nb_entries` alone, which can briefly dip below `hls->start_sequence` immediately after init when `vs->nb_entries` is still zero or near zero. The `FFMAX` floor at `hls->start_sequence` prevents this.

---

## Cross-References

The invariants above are deliberately phrased as obligations on the *behavior* of the HLS pipeline. The companion documents below give the supporting context that a reviewer of a port needs to understand each invariant in depth:

| Companion Document | Why It Matters for This Document |
|---|---|
| [`./data-contracts.md`](data-contracts.md) | Defines the data shapes that several invariants reference: `AVOption` types/defaults/bounds for `hls_allow_cache` (allowing -1/0/1 distinction), the `HLSSegment` field layout for the per-segment discontinuity invariant, the `HLSCryptoContext` binary layout that the EXT-X-KEY invariant depends on, the timestamp-unit conventions referenced by the ID3-timestamped time-base invariant. |
| [`./integration-contracts.md`](integration-contracts.md) | The external-system contracts that interpret the invariants — for example, the EXT-X-KEY METHOD=AES-128 invariant becomes an AES-128 key-fetch contract for the AVIOContext layer; the EXT-X-MEDIA rendition lines become a player-side rendition-discovery contract. |
| [`./timing-dependencies.md`](timing-dependencies.md) | The ordering rules that several invariants depend on — for example, the EXT-X-TARGETDURATION invariant requires target_duration to be computed before the playlist header is written; the EXT-X-MAP invariant requires the fMP4 initialization segment file to be written before the playlist references it. |
| [`../technical/codec-logic.md`](../technical/codec-logic.md) | The decision tables that implement the conditional-emission rules referenced in many of the invariants above (HLS_VERSION cascade, byterange-mode predicate, fMP4 segment-type effects). |
| [`../technical/data-model.md`](../technical/data-model.md) | The canonical struct dictionary for every field referenced above (`HLSContext::flags`, `VariantStream::has_video`, `HLSSegment::discont`, `HLSSegment::key_uri`, `HLSSegment::iv_string`, etc.). |
| [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md) | The lifecycle narrative that explains *when* each invariant is checked or enforced — for example, the EXT-X-ENDLIST invariant fires only on the final `hls_window` call, which is initiated by `hls_write_trailer`. |

---

## Validation Checklist

The checklist below is a single-page summary of all twenty-one invariants in this document, intended as a tick-through review aid for engineers verifying a port or refactor. Each item references the source location that enforces the invariant in the existing code at commit `566ad786`.

- [ ] **M3U8 Header Order.** Every M3U8 begins with `#EXTM3U\n` on line 1 and `#EXT-X-VERSION:<n>\n` on line 2. (`[libavformat/hlsplaylist.c:L36-L37]`)
- [ ] **EXT-X-VERSION Negotiation.** The version is exactly one of `{2, 3, 4, 6, 7}`, computed by the monotonic cascade. (`[libavformat/hlsenc.c:L1551-L1571]`)
- [ ] **Segment Naming.** Segment filenames follow `<basename>_<N>.<ext>` per the `POSTFIX_PATTERN "_%d"` macro. (`[libavformat/hlsenc.c:L74]`)
- [ ] **EXT-X-DISCONTINUITY Placement.** The tag appears only at sequence-start (when `HLS_DISCONT_START` is set) or per-segment (when `en->discont` is true). (`[libavformat/hlsenc.c:L1593-L1596]` and `[libavformat/hlsplaylist.c:L156-L158]`)
- [ ] **PTS/DTS Passthrough.** Packets forwarded to the child sub-muxer carry unmodified `pkt->pts`/`pkt->dts`/`pkt->duration`. (`[libavformat/hlsenc.c:L2679]`)
- [ ] **AVFMT_GLOBALHEADER.** The muxer requires encoders to supply `AVCodecParameters::extradata`. (`[libavformat/hlsenc.c:L3198]`)
- [ ] **EXT-X-TARGETDURATION.** The integer value emitted is `lrint(max(en->duration))` over the live segment list. (`[libavformat/hlsenc.c:L1584-L1587]`)
- [ ] **EXT-X-ENDLIST.** Emitted only on the final write (`last == 1`) AND when `HLS_OMIT_ENDLIST` is unset. (`[libavformat/hlsenc.c:L1629-L1630]`)
- [ ] **EXT-X-INDEPENDENT-SEGMENTS.** Emitted only when `vs->has_video` AND `HLS_INDEPENDENT_SEGMENTS` are both true. (`[libavformat/hlsenc.c:L1597-L1599]`)
- [ ] **EXT-X-I-FRAMES-ONLY.** Pins `EXT-X-VERSION` to at least 4 and emits the `#EXT-X-I-FRAMES-ONLY` header line. (`[libavformat/hlsenc.c:L1561-L1563]` and `[libavformat/hlsplaylist.c:L129-L131]`)
- [ ] **EXT-X-MAP for fMP4.** Emitted only when `segment_type == SEGMENT_TYPE_FMP4` and only on the first segment iteration. (`[libavformat/hlsenc.c:L1611-L1614]`)
- [ ] **EXT-X-BYTERANGE.** Emitted only when `byterange_mode` is true, i.e., `HLS_SINGLE_FILE` set OR `max_seg_size > 0`. (`[libavformat/hlsplaylist.c:L163-L165]` and `[libavformat/hlsenc.c:L1549]`)
- [ ] **EXT-X-KEY METHOD=AES-128.** The METHOD value is always literal `AES-128`; the line is re-emitted on URI or IV change. (`[libavformat/hlsenc.c:L1601-L1609]`)
- [ ] **fMP4 → Version 7.** When `segment_type == SEGMENT_TYPE_FMP4`, `EXT-X-VERSION` is exactly 7 (the final assignment in the cascade). (`[libavformat/hlsenc.c:L1569-L1571]`)
- [ ] **EXT-X-PLAYLIST-TYPE.** Emitted only for `PLAYLIST_TYPE_EVENT` or `PLAYLIST_TYPE_VOD`; never for `PLAYLIST_TYPE_NONE`. (`[libavformat/hlsplaylist.c:L124-L128]`)
- [ ] **EXT-X-ALLOW-CACHE.** Emitted only when `allowcache == 0` (as `NO`) or `allowcache == 1` (as `YES`); the default `-1` suppresses. (`[libavformat/hlsplaylist.c:L117-L119]` and `[libavformat/hlsenc.c:L3128]`)
- [ ] **LF Line Terminator.** Every playlist line ends with `\n`, never `\r\n`. (`[libavformat/hlsplaylist.c]` throughout: `L36-L37, L107, L118, L120-L121, L125-L127, L130, L141, L157, L160, L162, L164, L191, L196, L205`)
- [ ] **ID3-Timestamped Time Base.** Streams with `pls->is_id3_timestamped` get `avpriv_set_pts_info(st, 33, 1, MPEG_TIME_BASE)` with `MPEG_TIME_BASE == 90000`. (`[libavformat/hls.c:L2065-L2066]` and `[libavformat/hls.c:L56]`)
- [ ] **Demuxer Flags Cluster.** `.p.flags == AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`. (`[libavformat/hls.c:L2904]`)
- [ ] **Muxer Flags Cluster.** `.p.flags == AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS`. (`[libavformat/hlsenc.c:L3198]`)
- [ ] **EXT-X-MEDIA-SEQUENCE Monotonic.** The value `FFMAX(hls->start_sequence, vs->sequence - vs->nb_entries)` is monotonically non-decreasing across successive live publishes. (`[libavformat/hlsenc.c:L1539]`)

A port that satisfies every check mark above preserves the wire-format and behavioral contracts of the FFmpeg HLS pipeline as observable at commit `566ad786`. Reviewers should treat each unchecked item as a porting blocker and demand either an explanation (e.g., "we intentionally relaxed this; here is the wire-format precedent") or a fix before approval.
