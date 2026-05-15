# Functional Inventory — Components of the FFmpeg HLS Pipeline

This document is the canonical component inventory referenced by every Layer 2 (technical) and Layer 3 (api-contracts) document in this set. It enumerates the functional components of the FFmpeg HLS muxer and demuxer, with each component described first in plain language for library integrators and then in engineer-facing technical detail with full source citations.

All source references in this document are anchored to commit `566ad786`. See `../README.md` for the canonical commit-anchor banner and the citation-format specification.

## Overview

The FFmpeg HLS pipeline is two related subsystems that share several supporting modules. The **muxer** turns an application's stream of compressed audio, video, and subtitle frames into an HLS publication: a top-level playlist manifest file plus a sequence of fixed-length media segment files on disk or pushed to a remote server. The **demuxer** does the opposite — given a playlist URL, it fetches the manifest, fetches each referenced segment, and turns the result back into a stream of compressed frames that the application can decode.

Internally each side is factored into several focused components. On the muxer side these are: segment generation (chopping the input stream at keyframe boundaries into segment files), playlist construction (rewriting the manifest after each segment), encryption handling (optional AES-128 segment encryption with periodic key rotation), variant stream management (multiple bitrate ladders for adaptive playback), closed-captions tracks (embedded 608/708 caption advertisements), live versus VOD mode selection (sliding window versus complete recording), and discontinuity handling (timestamp/codec changes mid-stream). On the demuxer side: a probe that recognizes M3U8 magic bytes, a playlist parser that builds the in-memory representation of variants and segments, and a sample-encryption transform for the Apple Sample-AES variant.

Both sides share a small translation unit of playlist-tag writer helpers that emit the wire-format `#EXT-X-*` lines. That translation unit is also linked into the FFmpeg DASH muxer so both formats produce byte-identical HLS playlist output where their feature sets overlap. The components covered in this document are summarized in the table below.

| Component | Side | Section | Primary source file | Approximate size |
|---|---|---|---|---|
| Segment Generation | Muxer | §3 | `[libavformat/hlsenc.c]` | Core of `hls_write_packet` (≈200 LOC) |
| Playlist Construction | Muxer | §4 | `[libavformat/hlsenc.c]` + `[libavformat/hlsplaylist.c]` | `hls_window` (≈140 LOC) plus 206 LOC writers |
| Encryption Handling | Muxer | §5 | `[libavformat/hlsenc.c]` | `hls_encryption_start` + `do_encrypt` (≈100 LOC) |
| Variant Stream Management | Muxer | §6 | `[libavformat/hlsenc.c]` | Variant struct + map parser (≈300 LOC) |
| Closed Captions Track | Muxer | §7 | `[libavformat/hlsenc.c]` | Small struct + map parser (≈60 LOC) |
| Live vs VOD Mode Selection | Muxer | §8 | `[libavformat/hlsenc.c]` + `[libavformat/hlsplaylist.c]` | Cross-cutting concern (flag interactions) |
| Discontinuity Handling | Muxer | §9 | `[libavformat/hlsenc.c]` | Flag + per-segment field propagation |
| HLS Demuxer Probe | Demuxer | §10 | `[libavformat/hls.c]` | `hls_probe` (≈35 LOC) |
| HLS Demuxer Playlist Parser | Demuxer | §11 | `[libavformat/hls.c]` | `parse_playlist` (≈300 LOC) plus reads |
| HLS Sample Encryption | Demuxer | §12 | `[libavformat/hls_sample_encryption.c]` | 396 LOC complete transform |
| Playlist Tag Writers (shared) | Both | §13 | `[libavformat/hlsplaylist.c]` | 206 LOC of 8 writers |

A lifecycle roll-up table summarizing the muxer's standard FFmpeg lifecycle phases appears in §14.

### Reading order

This document is the canonical inventory; it is the *first* document a reader should consult when navigating the pipeline. The recommended traversal for each audience is:

- **Integrators** typically read components 1, 2, 3, 5, 6, and 8 (segment generation, playlist construction, encryption handling, variant stream management, live-vs-VOD selection) to form a system mental model, then drop into [`./inputs-outputs.md`](./inputs-outputs.md) for the AVOption table and the EXT-X-* tag inventory. Components 7, 9, 10, 11, and 12 (closed captions, discontinuity handling, demuxer components) are domain-specific and consulted only when those features are in scope.
- **Engineers** planning a port or refactor read all 11 components in order, then drop into the Layer 2 documents under [`../technical/`](../technical/) for decision tables, data-model dictionary, pipeline orchestration, and external interface reference.
- **Auditors** and **operators** typically read the Overview, then the Plain-Language tier of every component (skipping Technical Detail), then [`./exception-handling.md`](./exception-handling.md) for AVERROR mappings.

### Plain-language summary of the muxer's job

A muxer reader who wants the single-paragraph summary of what the HLS muxer does: it takes a stream of audio/video frames, chops them at video-keyframe boundaries every ~2 seconds, writes each chunk as a self-contained MPEG-TS or fMP4 file, and after every chunk rewrites a small text manifest (the M3U8 playlist) listing all known chunks. Optionally it encrypts each chunk with a rotating AES-128 key, splits the output into multiple parallel bitrate ladders for adaptive playback, advertises embedded captions and alternative audio/subtitle tracks, and emits discontinuity markers when bitstream parameters change. Streams can be live (sliding window of recent chunks), VOD (complete recording with end marker), or event (append-only timeshift).

### Plain-language summary of the demuxer's job

For the demuxer reader: it takes a playlist URL, downloads the playlist, parses the text to learn which segments exist and what URLs to fetch them from, downloads each segment in turn (re-fetching the playlist periodically for live streams to discover new segments), and runs each segment through an inner MPEG-TS or fMP4 demuxer to emit a stream of frames. For Sample-AES-encrypted streams it additionally decrypts the encrypted samples in each frame using a key fetched from a URL declared by the playlist.

## Component: Segment Generation

Segment generation is the muxer's primary job. The HLS publication model presents a continuous broadcast or recording as a series of short media files — typically two to ten seconds each — so a client player can request any one of them at any time and begin playback at that point. To make every segment a valid starting point, the HLS specification requires every segment to begin with a video keyframe. The muxer therefore watches every incoming frame, and when it sees a video keyframe that arrives after the configured target duration has elapsed since the previous segment boundary, it closes the segment file currently being written, opens a new one, and the next frames start filling that new file.

Once a segment file is closed, the muxer also records the segment's filename, duration, byte offset and size, optional encryption metadata, and discontinuity flag into a per-variant linked list. That linked list is the in-memory representation of the playlist manifest the muxer republishes after each new segment. Streams that never carry video, or streams where the application has set the flag asking the muxer to cut purely on elapsed time without waiting for keyframes, are also supported.

Integrators typically interact with segment generation through three options: the target segment length (`hls_time`, default 2 seconds), the maximum number of segments retained in a live playlist (`hls_list_size`, default 5), and the filename template for segment files (`hls_segment_filename`). The default behavior is a five-segment sliding window of two-second MPEG-TS segments — appropriate for low-latency live streaming. VOD publications typically set `hls_list_size=0` (unlimited) and may set `hls_playlist_type=vod` to mark the manifest as a complete recording; recording-with-delete deployments combine a non-zero `hls_list_size` with the `delete_segments` flag so old segment files are unlinked from disk as they roll off the live window.

The segment-cut decision is intentionally conservative. The muxer cuts only at video keyframes by default because non-keyframe-aligned segments cannot be decoded from their start — a client that joins mid-segment would have to wait for the next keyframe before producing any output. The `HLS_SPLIT_BY_TIME` flag relaxes this requirement for use cases where every frame is independently decodable (most audio-only HLS, intra-only video codecs) or where strict time alignment matters more than clean start frames. The cost of `HLS_SPLIT_BY_TIME` is that joining clients see a black screen until the next keyframe; the cost of *not* setting it is that segments may be longer than the configured `hls_time` when keyframes are sparse.

### Technical Detail

#### Packet entry and routing

- The packet pipeline entry point is `hls_write_packet` at `[libavformat/hlsenc.c:L2410]`. Each `AVPacket` delivered to the muxer through `av_write_frame` or `av_interleaved_write_frame` is dispatched into this callback for the variant the packet belongs to.
- Packet-to-variant routing iterates the `HLSContext::var_streams` array looking for the variant whose `streams` list contains the packet's `stream_index` (logic at `[libavformat/hlsenc.c:L2425-L2446]`). Subtitle packets (`AVMEDIA_TYPE_SUBTITLE`) are routed to the variant's parallel `vtt_avf` sub-muxer; all other packets go to `vs->avf` (the main TS/fMP4 sub-muxer).
- The reference stream for segment-cut decisions is identified by `vs->reference_stream_index` at `[libavformat/hlsenc.c:L152]`. By default, the reference is the variant's video stream when one is present; the local `is_ref_pkt` flag (set at L2476) is true only when `pkt->stream_index == vs->reference_stream_index`. Non-reference packets drive byterange accounting (`vs->size += pkt->size`) but do not advance the segment-cut clock.

#### Cut decision

- Whether the current packet is a candidate for cutting a new segment is decided by the local `can_split` flag at `[libavformat/hlsenc.c:L2473-L2475]`: the packet must either be a video keyframe (`pkt->flags & AV_PKT_FLAG_KEY` on a video stream) or the muxer must have the `HLS_SPLIT_BY_TIME` flag set (allowing cuts on elapsed time even between keyframes). The expression is `can_split = (st->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) && ((pkt->flags & AV_PKT_FLAG_KEY) || (hls->flags & HLS_SPLIT_BY_TIME))`.
- When `pkt->pts == AV_NOPTS_VALUE`, both `is_ref_pkt` and `can_split` are forced to zero at `[libavformat/hlsenc.c:L2478-L2479]`. A stream that never carries presentation timestamps will never trigger a time-based cut.
- A strictly positive PTS delta is also required at `[libavformat/hlsenc.c:L2500]` (`can_split = can_split && (pkt->pts - vs->end_pts > 0)`) so duplicate keyframe timestamps do not produce zero-duration segments.
- The cut-or-not comparison is performed at `[libavformat/hlsenc.c:L2501-L2502]` using `av_compare_ts(pkt->pts - vs->start_pts, st->time_base, end_pts, AV_TIME_BASE_Q) >= 0`. The `end_pts` value is `vs->end_pts + hls->time` (the accumulated end-of-segment PTS plus the configured target segment duration `hls->time`, default 2,000,000 µs = 2 s, defined as the `hls_time` AVOption at `[libavformat/hlsenc.c:L3123]`).
- The `hls_init_time` option at `[libavformat/hlsenc.c:L3124]` allows a *different* target duration for the first few segments of a publication. When set and the current sequence count is below the `hls_list_size` threshold, the muxer uses `init_time` instead of `time` as the cut target — useful for joining mid-broadcast scenarios where a quick first segment reduces time-to-first-frame.

#### Segment open and close

- Per-variant segment open is `hls_start` at `[libavformat/hlsenc.c:L1675]`. It opens the next segment file (via `hlsenc_io_open` at `[libavformat/hlsenc.c:L292]`, which transparently handles `file://`, `http://`, or any other registered protocol), resets the per-segment counters (`packets_written`, `start_pos`, `size`, `start_pts`, `end_pts`), and prepares the sub-muxer for the next segment.
- After the cut decision fires, the current segment is finalized: `av_write_frame(oc, NULL)` flushes any buffered packets at the sub-muxer level, the sub-muxer's `AVIOContext` is flushed and closed, and `hls_append_segment` at `[libavformat/hlsenc.c:L1042]` appends a record of the just-closed segment to the variant's linked list.
- Each segment record uses the `HLSSegment` struct defined at `[libavformat/hlsenc.c:L76-L94]`: `filename`, `sub_filename` (used for parallel WebVTT subtitle segment files), `duration`, `discont` (the per-segment discontinuity flag), `pos` and `size` (byterange offsets for byterange mode), `keyframe_pos` and `keyframe_size` (for `EXT-X-I-FRAMES-ONLY` mode), `var_stream_idx`, `key_uri` and `iv_string` (the encryption metadata captured at segment-close time), `next` (singly-linked-list pointer), `discont_program_date_time` (the wall-clock anchor for `EXT-X-PROGRAM-DATE-TIME`), plus a flexible-array `buf[]` tail used as backing storage for `filename` and `sub_filename`.

#### Meta-muxer pattern: child sub-muxer

- The actual bitstream format written into each segment file is owned by a child muxer — either MPEG-TS or fragmented MP4 — that the HLS muxer instantiates per variant. The HLS muxer is therefore a *meta-muxer*: it does not emit TS packets or MP4 boxes itself. The child format context is `vs->avf` at `[libavformat/hlsenc.c:L133]` (an `AVFormatContext` pointer), and the format is chosen by the `hls_segment_type` AVOption at `[libavformat/hlsenc.c:L3139-L3141]` (the option declaration at L3139 plus the `mpegts` and `fmp4` const aliases at L3140-L3141; values `SEGMENT_TYPE_MPEGTS = 0` or `SEGMENT_TYPE_FMP4 = 1`, defined at `[libavformat/hlsenc.c:L115-L118]`).
- The child sub-muxer is instantiated by `hls_mux_init` at `[libavformat/hlsenc.c:L773]`. For each variant, `hls_mux_init` calls `avformat_alloc_output_context2` with the chosen `AVOutputFormat` (the MPEG-TS muxer at `[libavformat/mpegtsenc.c:L2408]` or the fMP4-capable MP4 muxer), copies the relevant fields from the parent `AVFormatContext` into the child (`flags`, `avoid_negative_ts`, `interrupt_callback`, `max_delay`, `opaque`, `strict_std_compliance`, `metadata`), and registers each variant's `AVStream` array with the child via `avformat_new_stream` plus `avcodec_parameters_copy`. The child is opened with `avformat_write_header` inside `hls_start` for each segment.
- For TS segmenting the entire segment file is independent (no shared header): every segment carries its own MPEG-TS pat/pmt and is self-contained. For fragmented MP4 segmenting an initialization segment (default filename `"init.mp4"`, configurable via `hls_fmp4_init_filename` at `[libavformat/hlsenc.c:L3142]`) is written once at the head of the publication and referenced by every media segment through the `#EXT-X-MAP` tag in the playlist; the media segments themselves contain only fragment boxes (`moof`/`mdat`) without `moov`.

#### Audio-only and audio-first cases

- When the variant has audio but no video, the segment-cut logic falls back to time-based cutting on the audio stream. `vs->has_video` at `[libavformat/hlsenc.c:L136]` is set during `hls_init` based on the variant's stream array; when false, the `can_split` predicate becomes effectively `pkt->flags & AV_PKT_FLAG_KEY` — and since audio packets carry the key flag on every frame for most audio codecs, every audio packet is a candidate cut point and the time delta becomes the operative constraint.
- The `vs->start_pts_from_audio` field at `[libavformat/hlsenc.c:L139]` handles the variant-startup case where the first packet to arrive is audio (i.e., audio's `start_pts` is set first), but a later-arriving video packet has an *earlier* PTS. The audio-first start is lowered to the video PTS at `[libavformat/hlsenc.c:L2468-L2470]` so the variant's segment list begins at the earliest media time, not the earliest audio time.

#### Filename templating

Segment filenames are produced by expanding a template (the `hls_segment_filename` AVOption at `[libavformat/hlsenc.c:L3130]`). The template supports several substitution tokens:

| Token | Substitution | Notes |
|---|---|---|
| `%d` | Segment sequence number (an integer) | Required for filename uniqueness in live and multi-segment publications |
| `%v` | Variant logical name or zero-based variant index | Required for multi-variant publications |
| `%Y`, `%m`, `%d`, `%H`, `%M`, `%S` and other `strftime` tokens | Expanded as wall-clock fields when `use_localtime` is set | Per `strftime(3)` man page |
| `_%d` (suffix pattern `POSTFIX_PATTERN`) | Defined at `[libavformat/hlsenc.c:L74]` for default-suffix templates | Used when the user-supplied template lacks an explicit sequence-number placeholder |

The default template (when `hls_segment_filename` is unset) is the playlist filename with the `.m3u8` extension stripped and `_%d.ts` (or `_%d.m4s` for fMP4) appended. So a playlist `out.m3u8` produces segments `out_0.ts`, `out_1.ts`, ... by default.

Second-level templating flags (`HLS_SECOND_LEVEL_SEGMENT_INDEX`, `HLS_SECOND_LEVEL_SEGMENT_DURATION`, `HLS_SECOND_LEVEL_SEGMENT_SIZE` at `[libavformat/hlsenc.c:L107-L109]`) enable additional tokens that interpolate segment metadata into filenames; the typical use is `seg_%d_%t.ts` where `%t` is the segment's wall-clock timestamp.

#### Segment lifecycle quick-reference

A reader who needs the per-segment lifecycle in one table:

| Phase | Function | Source line |
|---|---|---|
| Decide to cut | `hls_write_packet` segment-cut block | `[libavformat/hlsenc.c:L2470-L2502]` |
| Finalize previous segment | `av_write_frame(NULL)` + `avio_close` | inside `hls_write_packet` post-cut block |
| Append metadata to variant list | `hls_append_segment` | `[libavformat/hlsenc.c:L1042]` |
| Publish playlist | `hls_window` | `[libavformat/hlsenc.c:L1531]` |
| Open new segment | `hls_start` | `[libavformat/hlsenc.c:L1675]` |
| Encrypt segment bytes (optional) | `do_encrypt` (via `hls_start` and per-write hooks) | `[libavformat/hlsenc.c:L641]` |

#### Cross-references

- The full segment-cut decision matrix (every condition, every outcome) lives in [`../technical/codec-logic.md`](../technical/codec-logic.md) as a decision table; this section explains *what* the decision is, the decision table enumerates *all* of *what makes the decision*.
- The Mermaid flowchart of the segment-generation control flow lives in [`../technical/process-flows.md`](../technical/process-flows.md).
- The internal callback chain (`hls_write_packet` → `hls_append_segment` → `sls_flags_filename_process` → `hls_window` → `hls_start`) is narrated in [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md).
- Field-level definitions of `HLSSegment`, `VariantStream::has_video`, `VariantStream::start_pts_from_audio`, `VariantStream::reference_stream_index`, etc. live in [`../technical/data-model.md`](../technical/data-model.md).
- The filename-template wire format and `strftime` token expansion contract lives in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).

## Component: Playlist Construction

After every new segment is closed, the muxer rewrites the playlist file so a client polling for updates discovers the new segment on its next refresh. The playlist is a small text manifest in the M3U8 format defined by the HLS specification: a fixed opening sequence of metadata tags identifying the playlist version, allowed cache behavior, target segment length, and starting sequence number, followed by one entry per segment listing the segment's duration and URL.

Playlist construction is split into a fixed-order pipeline of small writer helpers, each emitting one specific category of tag. The writer helpers live in a separate translation unit so the FFmpeg DASH muxer can share them — see §13 for that coupling. The muxer's own role is to drive the writers in the right order, compute aggregate fields like the target duration that must be set before any segment is listed, and route each writer's output to the correct destination (a file on disk, an HTTP `PUT` to a remote ingest endpoint, or a temporary file followed by an atomic rename).

From an integrator's perspective, playlist construction is the *visible* part of the HLS protocol: clients read the playlist file, parse the tag lines, and decide what to fetch next. Any mis-emission — a wrong target duration, a non-monotonic media-sequence number, a missing version line, a misplaced `#EXT-X-DISCONTINUITY` — is immediately observable in player behavior. The component is therefore the most contract-heavy part of the muxer; the corresponding zero-deviation rules are enumerated in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md), and the per-tag emission contracts in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).

The publish cadence is one playlist rewrite per closed segment. For a typical configuration (2-second target duration, 5-segment live window), this means a playlist rewrite every 2 seconds. The rewrite is full-file, not incremental — the muxer regenerates the entire playlist text from the in-memory segment list and overwrites (or HTTP-PUTs) the manifest file. This is intentional: HLS clients have no incremental-update mechanism; they re-parse the entire playlist on every refresh.

### Technical Detail

#### Publish entry point

- Per-variant playlist publish is the function `hls_window` at `[libavformat/hlsenc.c:L1531]`. It is called from `hls_write_packet` after each segment is appended (and once from `hls_write_trailer` at `[libavformat/hlsenc.c:L2727]` for the final flush). The function takes the parent `AVFormatContext`, a `last` flag indicating whether this is the trailer publish, and the `VariantStream` pointer.
- The function opens (or reuses) an `AVIOContext` for the playlist file, walks the variant's segment linked list to gather aggregate statistics (computing `EXT-X-TARGETDURATION`, the media-sequence integer, and per-segment cumulative metadata), then drives the writer helpers in the fixed order below.

#### Fixed-order top-of-playlist sequence

The order of the helper invocations is load-bearing — RFC 8216 requires several tags to appear in specific positions relative to each other. The eight ordered emissions are:

1. `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` — emits `#EXTM3U` on line 1 and `#EXT-X-VERSION:%d` on line 2. Always the first two lines of every playlist; `#EXTM3U` is the M3U8 magic line and `#EXT-X-VERSION` declares the highest version of the HLS spec the playlist conforms to (so a client knows which subsequent tags it must understand).
2. `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]` — emits (conditionally) `#EXT-X-ALLOW-CACHE` (when `allowcache != -1`, i.e., explicitly set by the user), `#EXT-X-TARGETDURATION:%d` (always), `#EXT-X-MEDIA-SEQUENCE:%lld` (always), `#EXT-X-PLAYLIST-TYPE:EVENT|VOD` (only when explicitly set via the `hls_playlist_type` option), and `#EXT-X-I-FRAMES-ONLY` (only when the `HLS_I_FRAMES_ONLY` flag is set).
3. Optional `#EXT-X-DISCONTINUITY` at top of segment list when `HLS_DISCONT_START` is set, emitted at `[libavformat/hlsenc.c:L1593-L1596]`. The emission is gated on `vs->discontinuity_set == 0` so the line is emitted exactly once at the start of the publication.
4. Optional `#EXT-X-INDEPENDENT-SEGMENTS` when the variant carries video and the `HLS_INDEPENDENT_SEGMENTS` flag is set, emitted at `[libavformat/hlsenc.c:L1597-L1599]`. The condition is `(vs->has_video) && (hls->flags & HLS_INDEPENDENT_SEGMENTS)`.
5. Per-segment `#EXT-X-KEY:METHOD=AES-128,URI="...",IV=0x...` line when the key has changed since the previous segment, emitted at `[libavformat/hlsenc.c:L1601-L1609]`. The muxer compares the segment's `key_uri` and `iv_string` against the previous-segment values and emits a fresh `#EXT-X-KEY` line only on change. Successive segments protected by the same key share a single preceding `#EXT-X-KEY` line.
6. Optional `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]` — emits `#EXT-X-MAP:URI="...",BYTERANGE="..."` exactly once at the head of the segment list when `segment_type=fmp4`. The URI is the fMP4 initialization-segment filename (default `init.mp4`); the optional `BYTERANGE` attribute is present when byterange mode is active.
7. Per-segment `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L144-L199]` — emits the per-segment block: optional `#EXT-X-DISCONTINUITY` (when `seg->discont` is set), the `#EXTINF:%f,` duration line, optional `#EXT-X-BYTERANGE:%lld@%lld` (byterange mode), optional `#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s` with timezone-offset suffix (when the `HLS_PROGRAM_DATE_TIME` flag is set), and finally the segment URL itself on its own line (optionally prefixed by `hls_base_url`).
8. Optional `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]` — emits `#EXT-X-ENDLIST` on the trailer publish when `HLS_OMIT_ENDLIST` is not set, emission gated at `[libavformat/hlsenc.c:L1629-L1630]`. The presence of this tag tells clients the playlist is complete and will not be appended to; its absence in a finished playlist (because `HLS_OMIT_ENDLIST` was set) keeps the playlist looking "live" even when no more segments will be added.

#### Aggregate computations

- The integer value emitted in the `#EXT-X-VERSION` line is negotiated by `hls_window` based on which features the muxer is using on this publish; the result is the maximum of all version-floor rules required by enabled features. The five possible outcomes (2, 3, 4, 6, 7) are determined by the cascading conditions at `[libavformat/hlsenc.c:L1551-L1571]`. The full negotiation matrix lives in [`../technical/codec-logic.md`](../technical/codec-logic.md); this document treats the version as a derived quantity. Higher versions enable additional tag types: byterange mode requires version 4, `EXT-X-INDEPENDENT-SEGMENTS` requires version 6, and fMP4 mode requires version 7.
- The `#EXT-X-TARGETDURATION` field is computed at `[libavformat/hlsenc.c:L1584-L1587]` as the `lrint` of the maximum segment duration in the current segment list. The HLS specification requires this field to be greater than or equal to every segment's duration, so the muxer is conservative and emits the maximum across all retained segments. A single oversized segment (e.g., one that ran long because a keyframe was late) bumps the target for the entire playlist.
- The `#EXT-X-MEDIA-SEQUENCE` value is computed at `[libavformat/hlsenc.c:L1539]` as `FFMAX(hls->start_sequence, vs->sequence - vs->nb_entries)`. The `FFMAX` guard prevents the sequence from dipping below the user-configured `start_sequence` floor (used in restart and append scenarios); `vs->sequence` advances on every closed segment.

#### Atomic playlist replacement

- When the `HLS_TEMP_FILE` flag is set, each playlist publish writes to `<m3u8_name>.tmp` first and renames atomically via `hls_rename_temp_file` at `[libavformat/hlsenc.c:L1300]`. The atomic rename protects clients that may be polling the playlist while it is being rewritten — a client GET that lands between the truncate and the final write would otherwise see a partial manifest.
- When `HLS_TEMP_FILE` is not set, the muxer truncates the playlist file in place and rewrites it. This is fine for `file://` URLs on local filesystems where the OS provides crash-consistency guarantees, but is risky for slow filesystems or HTTP destinations; the latter case is the primary motivation for the `HLS_TEMP_FILE` flag.
- For HTTP destinations, `hls_rename_temp_file` issues a `RENAME` request through the protocol layer. The HTTP protocol implementation translates this into a `MOVE` or `COPY+DELETE` operation depending on the configured method; the exact wire format is part of the integration contract documented in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).

#### Master playlist publication

- When the variant set produces a *master playlist* (the top-level manifest enumerating all variants), the master playlist is published separately from the per-variant playlists. Master-playlist publishing cadence is controlled by `master_pl_publish_rate` at `[libavformat/hlsenc.c:L3175]`; the master is rewritten only when the variant sequence number is divisible by that rate, so a player polling the master sees an update at a configurable cadence independent of segment cadence. The default value of `0` (publish-once) is appropriate when the variant set never changes after the publication starts.
- The master playlist is written by `hls_write_packet` itself by interleaving calls to `ff_hls_write_audio_rendition`, `ff_hls_write_subtitle_rendition`, and `ff_hls_write_stream_info` — see §6 for those helpers and the variant-stream management component.

#### Playlist file content map

A complete media playlist for a typical configuration consists of (in order):

| Line | Content | Source |
|---|---|---|
| 1 | `#EXTM3U` | `ff_hls_write_playlist_version` |
| 2 | `#EXT-X-VERSION:<n>` | `ff_hls_write_playlist_version` |
| 3 | `#EXT-X-ALLOW-CACHE:YES` (conditional) | `ff_hls_write_playlist_header` |
| 4 | `#EXT-X-TARGETDURATION:<seconds>` | `ff_hls_write_playlist_header` |
| 5 | `#EXT-X-MEDIA-SEQUENCE:<seq>` | `ff_hls_write_playlist_header` |
| 6 | `#EXT-X-PLAYLIST-TYPE:EVENT|VOD` (conditional) | `ff_hls_write_playlist_header` |
| 7 | `#EXT-X-I-FRAMES-ONLY` (conditional) | `ff_hls_write_playlist_header` |
| 8 | `#EXT-X-DISCONTINUITY` (conditional, once) | inline in `hls_window` |
| 9 | `#EXT-X-INDEPENDENT-SEGMENTS` (conditional) | inline in `hls_window` |
| 10 | `#EXT-X-MAP:URI=...,BYTERANGE=...` (conditional, fmp4 only) | `ff_hls_write_init_file` |
| 11+ | Per-segment block: `#EXT-X-KEY` (on key change), `#EXT-X-DISCONTINUITY` (per segment), `#EXTINF:...`, `#EXT-X-BYTERANGE` (conditional), `#EXT-X-PROGRAM-DATE-TIME` (conditional), segment URL | `ff_hls_write_file_entry` |
| Last | `#EXT-X-ENDLIST` (conditional) | `ff_hls_write_end_list` |

Each line ends with LF (`\n`), not CRLF. The playlist character set is UTF-8.

#### Master playlist file content map

A complete master playlist consists of (in order):

| Line | Content | Source |
|---|---|---|
| 1 | `#EXTM3U` | `ff_hls_write_playlist_version` |
| 2 | `#EXT-X-VERSION:<n>` | `ff_hls_write_playlist_version` |
| 3+ | Per closed-captions group: `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,...` | inline emission in `hls_write_packet` |
| ...+ | Per audio rendition: `#EXT-X-MEDIA:TYPE=AUDIO,...` | `ff_hls_write_audio_rendition` |
| ...+ | Per subtitle rendition: `#EXT-X-MEDIA:TYPE=SUBTITLES,...` | `ff_hls_write_subtitle_rendition` |
| ...+ | Per variant: `#EXT-X-STREAM-INF:...` followed by URL | `ff_hls_write_stream_info` |

#### Cross-references

- The decision matrix for every conditional emission (version negotiation, target-duration computation, discontinuity placement, byterange mode, `EXT-X-MAP` emission, etc.) is enumerated in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The Mermaid flowchart of the playlist-update control flow lives in [`../technical/process-flows.md`](../technical/process-flows.md).
- The processing-order constraints (target duration must be computed before any segment is listed, segment file must be fully written before playlist update, etc.) live in [`../api-contracts/timing-dependencies.md`](../api-contracts/timing-dependencies.md).
- The per-tag wire-format contracts (exact byte sequences for every `#EXT-X-*` line) live in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- The eight playlist tag writer functions are documented individually in §13 (Playlist Tag Writers).

## Component: Encryption Handling

The muxer can optionally encrypt every segment file using AES-128 in cipher-block-chaining mode. When encryption is enabled, three pieces of information are needed: the 16-byte encryption key itself (a secret known to authorized clients), a 16-byte initialization vector that perturbs the cipher's starting state, and the URL where authorized clients fetch the key over a separate channel (typically an authenticated HTTPS endpoint). The playlist advertises the key URL but not the key itself, so an eavesdropper who captures the playlist and segment files cannot decrypt them without first authenticating to the key endpoint.

The three values are supplied to the muxer through a small text file pointed at by the `hls_key_info_file` option (three lines: the URL clients will fetch, the local path to the raw key bytes the muxer should use, and an optional hex-encoded IV). The muxer also supports periodic key rotation — long-running streams can change their encryption key without restarting by replacing the key info file's contents at any time; the muxer re-reads the file before each segment and rotates without interrupting the segment stream.

Encryption is opt-in. By default the muxer publishes plaintext segments — appropriate for free-to-air broadcasts, public CDN distributions, and most VOD use cases. Encryption is engaged when the integrator wants per-client access control: the playlist URL is shared freely (or distributed through a discovery mechanism), but only clients that can authenticate to the key URL can play. The model is sometimes called "session-encryption" because the same key is used for all clients that successfully fetch it; for per-client encryption the integrator must use a dedicated DRM system instead, of which HLS is the playlist-and-segment substrate.

This component covers *segment-level* AES-128 CBC encryption (every byte of every segment file is encrypted). HLS also supports a second encryption scheme — *Sample Encryption* / Sample-AES — where individual samples inside an MPEG-TS segment are encrypted while the TS packet structure remains in clear. Sample Encryption is documented separately as §12 of this inventory; this section covers only the segment-level scheme.

### Technical Detail

#### Key info file format

- The encryption initialization function is `hls_encryption_start` at `[libavformat/hlsenc.c:L714]`. It parses the key info file (path from the `hls_key_info_file` AVOption at `[libavformat/hlsenc.c:L3133]`), reads the URL from line 1 into `vs->key_uri`, reads the 16 raw binary key bytes from the file path on line 2 into `vs->key_string`, and optionally parses a 32-hex-character IV from line 3 into `vs->iv_string`.
- The key info file is a plain UTF-8 text file with three newline-separated lines:
  1. The URL clients will fetch the key from (any URL the client can resolve — typically `https://...` with authentication).
  2. The local filesystem path the muxer should read the 16 raw binary key bytes from.
  3. *Optional* — a 32-hex-character IV. When absent, the IV is derived per-segment from the segment sequence number.
- The key file referenced on line 2 must contain exactly 16 bytes of raw binary key material — not hex, not base64. Mismatched size produces an `AVERROR(EINVAL)` from `hls_encryption_start`.
- The key info file is *not* uploaded to the publishing destination — it lives only on the muxer-side filesystem. The URL on line 1 is what *clients* will fetch; the muxer never serves the key.

#### Constants and storage

- The key and IV size is constant: `#define KEYSIZE 16` at `[libavformat/hlsenc.c:L70]`. Both the AES key and the IV are exactly 16 bytes.
- Per-variant storage for the encryption state lives in `VariantStream` at `[libavformat/hlsenc.c:L177-L180]`:
  - `key_file[LINE_BUFFER_SIZE + 1]` — the path read from line 2 of the key info file
  - `key_uri[LINE_BUFFER_SIZE + 1]` — the URL read from line 1
  - `key_string[KEYSIZE*2 + 1]` — the hex-encoded key value (stored as hex for emission compatibility, not as raw bytes)
  - `iv_string[KEYSIZE*2 + 1]` — the hex-encoded IV value
- The `encrypt_started` field at `[libavformat/hlsenc.c:L175]` is a per-variant boolean that tracks whether `hls_encryption_start` has successfully completed for the current variant. The first call sets it to 1; subsequent re-keying calls (in periodic-rekey mode) reuse the same flag to gate re-initialization of the AES context.

#### Per-segment encryption path

- Per-segment AES-128 CBC encryption is performed by the `do_encrypt` helper at `[libavformat/hlsenc.c:L641]`. It uses the FFmpeg AES library API: `av_aes_alloc` at `[libavutil/aes.h:L41]`, `av_aes_init` at `[libavutil/aes.h:L51]`, and `av_aes_crypt` at `[libavutil/aes.h:L63]`. The helper allocates an `AVAES` context, initializes it with the 16-byte key and the per-segment IV, runs the entire segment payload through `av_aes_crypt` in chained-CBC mode, and writes the resulting ciphertext to the segment file.
- The IV used for each segment is either (a) the user-supplied IV from line 3 of the key info file (when present, the same IV is used for every segment of the current key period), or (b) the segment sequence number formatted as a 16-byte big-endian integer (when no user IV is supplied — this matches RFC 8216 §5.2 default IV derivation).

#### Playlist tag emission

- The `#EXT-X-KEY` playlist tag is emitted inside `hls_window` at `[libavformat/hlsenc.c:L1601-L1609]` immediately before the first segment it protects. The tag format is `#EXT-X-KEY:METHOD=AES-128,URI="<url>"` with an optional `,IV=0x<32-hex-chars>` suffix when the IV is explicitly carried in the playlist rather than derived from the segment sequence number.
- Per-segment encryption metadata is carried on each `HLSSegment` record (struct at `[libavformat/hlsenc.c:L76-L94]`): the `key_uri` and `iv_string` fields are populated when the segment is appended so the playlist can emit a new `#EXT-X-KEY` line whenever the URI or IV changes between segments. The comparison-with-previous logic ensures the line is emitted only on change, not on every segment.

#### Periodic rekey

- When the `HLS_PERIODIC_REKEY` flag is set (AVOption const `periodic_rekey` at `[libavformat/hlsenc.c:L3157]`, enum value `HLS_PERIODIC_REKEY = (1 << 12)` defined at `[libavformat/hlsenc.c:L110]`), the muxer re-reads the key info file before every segment. If the file's contents have changed since the previous segment, the new key and IV take effect immediately and a new `#EXT-X-KEY` line is emitted in the playlist before the next segment entry.
- The rekey check is performed by re-invoking `hls_encryption_start` for each new segment; the function compares the freshly-read key against the stored `vs->key_string` and `vs->iv_string` to detect changes.
- Operationally, the integrator rotates keys by atomically replacing the key file (e.g., `cp newkey.bin /etc/hls/key.bin.new && mv /etc/hls/key.bin.new /etc/hls/key.bin`). The atomic move ensures the muxer never observes a half-written key file.

#### Built-in key generation

- A simpler encryption mode is also available: the `hls_enc` AVOption at `[libavformat/hlsenc.c:L3134]` enables built-in key generation when no key info file is supplied. The muxer derives a random 16-byte key, derives the IV from the segment sequence number, and writes both into a `.key` file alongside the segments — used primarily for self-contained demo or development setups.
- The `hls_enc_key` option at `[libavformat/hlsenc.c:L3135]` accepts a hex-encoded 16-byte key string directly (e.g., on the command line) instead of via a file; the `hls_enc_key_url` option at `[libavformat/hlsenc.c:L3136]` provides the URL clients will fetch; the `hls_enc_iv` option at `[libavformat/hlsenc.c:L3137]` provides an explicit IV. These options are an alternative to the key info file for ad-hoc encrypted-segment generation.

#### Sample Encryption is separate

- The HLS muxer does *not* implement Sample Encryption (Sample-AES) for outgoing streams — that workflow requires upstream encryption of the samples before they enter the muxer. The HLS *demuxer* does support Sample-AES decryption (see §12); the asymmetric support reflects the demuxer's need to play Apple-published Sample-AES content, paired with the absence of a standard outbound encryption path. The build system at `[libavformat/Makefile:L276-L277]` links `hls_sample_encryption.o` only into the demuxer.

#### Security threat model

The encryption design assumes the following threat model:

- **Adversary**: an attacker who can observe network traffic and capture segment files but cannot authenticate to the key endpoint.
- **Protection goal**: prevent the adversary from decoding the segment files into watchable content without first obtaining the key.
- **Out of scope**: defending against an attacker who has captured the key (e.g., a paying customer who shares their authenticated key fetch with a third party). This is a content-distribution-DRM problem and requires a higher-layer system.

The threat model implies several deployment requirements:

- The key endpoint MUST require authentication. If the playlist references `https://example.com/key.bin` and that URL is publicly accessible, the encryption provides no protection.
- The key endpoint MUST use HTTPS or an equivalent encrypted transport. Plaintext HTTP key fetches expose the key to the same network observer the encryption was supposed to defeat.
- The key file on the muxer side MUST have appropriate filesystem permissions. A key readable by all local users is no more secret than a publicly distributed one.
- The IV does not need to be secret — it only needs to be unique per encryption operation. The HLS specification permits the IV to be the segment sequence number, which the playlist already advertises in clear text.

#### Per-key-period segment grouping

Successive segments protected by the *same* key are grouped under a single `#EXT-X-KEY` line; the line appears once, before the first segment of the group, and applies to every following segment until either a new `#EXT-X-KEY` line appears or the playlist ends. The grouping is implemented by the comparison logic at `[libavformat/hlsenc.c:L1601-L1609]`: each segment's `key_uri` and `iv_string` are compared against the previous segment's, and a fresh `#EXT-X-KEY` line is emitted only on change.

This grouping is significant for periodic-rekey deployments: a rekey that happens between segments 5 and 6 produces a playlist with one `#EXT-X-KEY` line before segment 0 and another `#EXT-X-KEY` line before segment 6, with no extra noise on the intervening segments.

#### Compatibility with `hls_segment_type=fmp4`

Encryption is supported for both MPEG-TS segments and fragmented MP4 segments. The encryption path is the same — the segment bytes (whatever the segment format) are run through `do_encrypt` before being written to the destination. fMP4 segments encrypt their entire `moof`+`mdat` payload; the initialization segment (`init.mp4`) is unencrypted because it must be readable by the client to initialize the decoder.

#### Cross-references

- The key URI fetch protocol and the AES-128 crypto pipeline integration are documented in [`../technical/integration-interfaces.md`](../technical/integration-interfaces.md).
- The AES-128 wire-format contracts (the exact `#EXT-X-KEY` byte sequence, the IV derivation rule, the segment-encryption byte layout) live in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- The complete decision matrix for when `#EXT-X-KEY` is emitted (versus suppressed because the previous segment used the same key) is enumerated in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The `do_encrypt` and `hls_encryption_start` failure modes and their AVERROR returns are catalogued in [`./exception-handling.md`](./exception-handling.md).
- The Sample Encryption transform (for demuxing Apple Sample-AES streams) is documented in §12 (HLS Sample Encryption).

## Component: Variant Stream Management

Adaptive-bitrate playback is one of the defining features of HLS: a single source can be encoded at several quality tiers (for example 4K, 1080p, 720p, 480p) and the client player automatically picks the tier that best matches the available bandwidth, switching tiers at segment boundaries as bandwidth conditions change. The muxer supports this model by accepting a *variant stream map* — a small text string mapping each input stream into one of several variants. Each variant gets its own playlist file and its own series of segment files; a top-level *master playlist* lists all variants and lets clients discover them.

Variants can also be paired with separate audio-only or subtitle-only playlists (called *renditions*) using HLS group identifiers. A typical configuration produces one video-bearing variant playlist per quality tier, one audio-only rendition playlist per language, and one subtitle rendition playlist per language; the master playlist's `#EXT-X-STREAM-INF` entries reference the audio and subtitle groups so clients fetch the right audio and subtitle tracks alongside whichever video tier they are currently consuming.

The variant-stream-map syntax is compact but contract-heavy. A string like `v:0,a:0 v:1,a:1` declares two variants: variant 0 contains input stream 0 (video) and input stream 1 (audio); variant 1 contains input stream 2 (video) and input stream 3 (audio). Optional named groups (`a:0,agroup:aac_main`) link audio streams to renditions. The variant index assigned by the muxer (in source order) is preserved in the output filenames via the `%v` template token; for example a `hls_segment_filename` template of `vs%v/seg_%d.ts` produces `vs0/seg_0.ts`, `vs0/seg_1.ts`, … for variant 0 and `vs1/seg_0.ts`, `vs1/seg_1.ts`, … for variant 1.

Without `var_stream_map`, the muxer publishes a single variant containing all input streams. This is the simplest mode and the default; integrators typically engage variant streams only when they need adaptive bitrate, multiple audio languages, or a structured rendition layout. The master playlist is published only when the variant count is greater than one OR when `master_pl_name` is set explicitly.

### Technical Detail

#### Per-variant state

- The per-variant state is the `VariantStream` struct at `[libavformat/hlsenc.c:L120-L194]` (≈ 75 fields). Key fields include: `sequence` (current segment sequence number), `oformat` and `avf` (the child format and its `AVFormatContext` used as a sub-muxer to emit the actual TS or fMP4 bytes), `vtt_oformat` and `vtt_avf` (a parallel sub-muxer for WebVTT subtitle segments), `packets_written`, `init_range_length`, accumulators `total_size`/`total_duration`/`avg_bitrate`/`max_bitrate`, linked-list heads `segments`/`last_segment`/`old_segments`, file-name templates `basename`/`m3u8_name`/`vtt_basename`/`vtt_m3u8_name`/`fmp4_init_filename`/`base_output_dirname`, and group-membership strings `agroup`, `sgroup`, `ccgroup`, `varname`, `subtitle_varname`.
- The per-context arrays are on the parent `HLSContext` (struct at `[libavformat/hlsenc.c:L202-L267]`): `var_streams` (pointer to the array), `nb_varstreams` (count). The array is heap-allocated during `hls_init` to a size determined by parsing `var_stream_map`.
- The `var_stream_idx` field on `VariantStream` (at `[libavformat/hlsenc.c:L121]`) is the zero-based position of the variant in the `var_streams` array. It is exposed to filename templates via `%v` substitution; integrators rely on it for directory layout and CDN routing.
- The `streams` field on `VariantStream` (at `[libavformat/hlsenc.c:L182]`) is a per-variant pointer array of `AVStream *` pointing into the parent context's `s->streams[]` array. Each variant owns a subset of the parent's streams; the muxer's packet-routing logic (in `hls_write_packet`) walks this array to find the variant for each incoming packet.
- The `nb_streams` field on `VariantStream` (at `[libavformat/hlsenc.c:L185]`) is the variant's stream count; for single-variant publications it equals `s->nb_streams`.

#### Variant map parsing

- The user-supplied variant map is parsed by `parse_variant_stream_mapstring` at `[libavformat/hlsenc.c:L1998]`. The string is consumed from the `var_stream_map` AVOption at `[libavformat/hlsenc.c:L3172]`. Syntax follows the form `v:0,a:0 v:1,a:1` where each space-separated group becomes one variant and each colon-tagged token (`v:`, `a:`, `s:`) selects an input stream by index.
- Supported tokens inside a variant entry:
  - `v:<n>` — input stream `n` is a video stream, included in this variant
  - `a:<n>` — input stream `n` is an audio stream, included in this variant
  - `s:<n>` — input stream `n` is a subtitle stream, included in this variant
  - `agroup:<name>` — bind this variant's audio to the named rendition group (so `#EXT-X-STREAM-INF` emits `AUDIO="<name>"`)
  - `sgroup:<name>` — bind this variant's subtitles to the named subtitle rendition group
  - `ccgroup:<name>` — bind this variant's closed captions to the named CC group
  - `name:<name>` — assign a logical name to the variant, used by `%v` filename substitution
  - `language:<lang>` — for audio-only renditions, the `LANGUAGE=` attribute
  - `default:<0|1>` — for audio renditions, the `DEFAULT=YES|NO` attribute
- The parser populates each `VariantStream` record's `streams`, `nb_streams`, and group-name fields; error handling sets `ret = AVERROR(EINVAL)` and propagates to the caller for any malformed entry.

#### Master playlist publication

- When the variant set has more than one variant or when the `master_pl_name` AVOption at `[libavformat/hlsenc.c:L3174]` is set, the muxer also publishes a master playlist. The master playlist is built by `hls_write_packet`'s master-playlist branch which calls into the playlist tag writers:
  - `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` for the header.
  - Per closed-captions group: `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,...` lines emitted directly by the muxer at `[libavformat/hlsenc.c:L1398-L1406]`.
  - Per audio-only rendition: `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L40-L56]`. Emits `#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="<group>",NAME="<name>",DEFAULT=YES|NO,LANGUAGE="<lang>",CHANNELS="<N>",URI="<playlist-url>"`.
  - Per subtitle rendition: `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]`. Emits `#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="<group>",NAME="<name>",DEFAULT=YES|NO,LANGUAGE="<lang>",URI="<playlist-url>"`.
  - Per variant: `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]` — emits `#EXT-X-STREAM-INF:BANDWIDTH=%d,AVERAGE-BANDWIDTH=%d,RESOLUTION=WxH,CODECS="...",AUDIO="group_%s",CLOSED-CAPTIONS="%s",SUBTITLES="%s"` followed by the variant's playlist URL on the next line.

#### Master republishing cadence

- Master-playlist republishing cadence is controlled by `master_pl_publish_rate` at `[libavformat/hlsenc.c:L3175]`. The muxer rewrites the master only when the current variant sequence number is divisible by this rate, which lets a publisher refresh the master at a coarser cadence than per-variant playlists (the master rarely changes once all variants are known, so frequent rewrites would be wasteful).
- The default value of `0` is special: it means "publish the master once, then never again". For variant sets where the rendition layout is fixed at start time, this is appropriate; for dynamic configurations where renditions are added or removed mid-publication, a non-zero rate ensures the master is refreshed periodically.

#### Bandwidth accounting

- Each variant's bandwidth accounting (the `total_size`, `total_duration`, `avg_bitrate`, and `max_bitrate` fields at `[libavformat/hlsenc.c:L154-L157]`) is updated after each segment to reflect actual bytes-per-second observed. The master-playlist `BANDWIDTH` and `AVERAGE-BANDWIDTH` attributes use these values so the advertised values reflect real consumption, not nominal target bitrates.
- The `BANDWIDTH` field is the *peak* bitrate (worst-case per-segment): a player reading this number knows the network capacity required to sustain playback at this tier without buffering. The `AVERAGE-BANDWIDTH` field is the *average* across all observed segments: a player uses this for steady-state bandwidth estimation and tier-switch decisions.
- The computation runs in `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]`, which reads the accumulators from the `VariantStream` record and emits the resulting bandwidth integers as attributes on the `#EXT-X-STREAM-INF` line.

#### Variant filename templating

Variant-specific filenames are produced from templates containing the `%v` token, which the muxer expands to the variant's logical name (from `name:` in the map) or to its zero-based index when no name was provided. The expansion happens in `format_name` at `[libavformat/hlsenc.c:L240]` and related helpers. Common patterns:

- `out_%v.m3u8` → `out_0.m3u8`, `out_1.m3u8`, ... (per-variant playlist filenames)
- `vs%v/seg_%d.ts` → `vs0/seg_0.ts`, `vs0/seg_1.ts`, `vs1/seg_0.ts`, ... (per-variant directories with sequenced segment files)
- `vs%v/data.m3u8` → `vs0/data.m3u8`, `vs1/data.m3u8`, ... (named directories with fixed playlist filenames)

The `%v` token is required when more than one variant is configured; without it, all variants would write to the same filename and overwrite each other. The muxer validates this requirement during `hls_init` and returns `AVERROR(EINVAL)` if a multi-variant configuration uses a template lacking `%v`.

#### Group identifier model

HLS uses a group-identifier indirection to wire audio and subtitle renditions to video variants. The same audio rendition group can be referenced by multiple video variants, allowing clients to switch video bitrate independently of audio language selection. The model has three parts:

| Construct | Purpose | Example |
|---|---|---|
| Audio rendition | A standalone audio-only playlist with a `GROUP-ID` attribute | `agroup:aac_main` |
| Subtitle rendition | A standalone subtitle playlist with a `GROUP-ID` attribute | `sgroup:subs_main` |
| Video variant reference | A `#EXT-X-STREAM-INF` line with `AUDIO=` and/or `SUBTITLES=` attributes pointing to a group | `v:0,a:0,agroup:aac_main,sgroup:subs_main` |

The muxer enforces a soft contract: a video variant that names an audio group must have at least one corresponding audio rendition declared with that group ID, or the resulting playlist would reference a nonexistent group. This is validated during `hls_init` after the variant map is fully parsed.

#### Default subtitle/audio selection

The `default:` token on a rendition entry maps to the `DEFAULT=YES|NO` attribute on the emitted `#EXT-X-MEDIA` line. Players use this to choose which audio/subtitle track to play when the user has not explicitly selected one. Best practice is to mark exactly one entry per group as default; marking zero entries leaves the player free to pick any, marking multiple entries produces undefined client behavior.

#### Cross-references

- The `VariantStream` struct field dictionary (all 75 fields with name, type, and business meaning) lives in [`../technical/data-model.md`](../technical/data-model.md).
- The `var_stream_map` parsing decision matrix (which tokens combine validly, which combinations error) is enumerated in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The `EXT-X-STREAM-INF` and `EXT-X-MEDIA` wire-format contracts (exact byte sequences for every attribute) live in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- The `BANDWIDTH` annotation contract (peak versus average semantics, computation timing) lives in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- Closed-captions group binding is documented separately in §7 (Closed Captions Track).


## Component: Closed Captions Track

Closed captions in HLS are the embedded EIA-608 or CTA-708 caption streams carried inside a video stream's bitstream rather than transmitted as separate text files. They are not their own playlist; they are advertised in the master playlist using a `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` line that names a group, language, and instream identifier (such as `CC1`, `CC2`, `CC3`, `CC4`, or `SERVICE1` through `SERVICE63`). A video variant then references the same group identifier through the `CLOSED-CAPTIONS=` attribute of its `#EXT-X-STREAM-INF` line; a player consuming that variant knows which caption channel to extract from the video bitstream and which language label to display.

The muxer accepts a small per-captions-stream configuration mapping the group name, language, and instream identifier through the `cc_stream_map` option. Each entry produces one `#EXT-X-MEDIA` line in the master playlist. Individual video variants are wired to a particular caption group through their variant-stream-map entry, which causes the muxer to add a `CLOSED-CAPTIONS="<group>"` attribute to that variant's `#EXT-X-STREAM-INF` line so the client knows which caption channel and language label belong to which video tier.

This component is intentionally narrow: the muxer does not *encode* captions, *decode* captions, or *transcode* captions. The caption bytes remain inside the video bitstream as the upstream encoder placed them; the muxer's only job is to make their presence discoverable to a client through the master playlist. Integrators who need to add or modify caption content must do so upstream of the HLS muxer (typically inside the video encoder); integrators who want separate subtitle tracks (rather than embedded captions) should use the `subtitle_varname`/`s:` rendition mechanism instead.

The four classic EIA-608 channels are addressed as `CC1`, `CC2`, `CC3`, `CC4` (program 1's first through fourth caption channels). CTA-708 services are addressed as `SERVICE1` through `SERVICE63`. The integrator selects the channel by setting `instreamid:` to the appropriate literal in the `cc_stream_map` entry; the muxer copies the value through to the `INSTREAM-ID` attribute on the emitted `#EXT-X-MEDIA` line without validation, so any client-recognized identifier is acceptable.

### Technical Detail

#### Struct and arrays

- The `ClosedCaptionsStream` struct at `[libavformat/hlsenc.c:L196-L200]` carries three fields per closed-captions stream: `ccgroup` (the group ID referenced by variant `#EXT-X-STREAM-INF` lines), `instreamid` (the `INSTREAM-ID` attribute value, typically `CC1`..`CC4` or `SERVICE1`..`SERVICE63`), and `language` (the BCP-47 language tag, e.g., `en`, `es`, `fr`).
- Per-context arrays on `HLSContext`: `cc_streams` (heap-allocated array of `ClosedCaptionsStream`) and `nb_ccstreams` (count) at `[libavformat/hlsenc.c:L248-L249]`. Both are populated during `hls_init` from the parsed `cc_stream_map` string.

#### Map parsing

- The user-supplied configuration string is the `cc_stream_map` AVOption at `[libavformat/hlsenc.c:L3173]`. It is parsed by `parse_cc_stream_mapstring` at `[libavformat/hlsenc.c:L2144]`. Syntax is space-separated entries of the form `ccgroup:<group>,instreamid:<id>,language:<lang>`.
- The parser walks the input string token by token, identifies the `ccgroup:`, `instreamid:`, and `language:` prefixes, copies their values into the per-entry `ClosedCaptionsStream` fields, and appends one entry to `hls->cc_streams` per space-separated group in the input.
- Validation is minimal: the muxer trusts the integrator to supply a valid `INSTREAM-ID` (the value is emitted verbatim to the playlist). Mistyping `CC1` as `cc1` produces a playlist line with the lowercase form; some lenient players accept it, strict players ignore the rendition entirely.

#### Master-playlist emission

- The `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` emission loop runs inside the master-playlist writer block at `[libavformat/hlsenc.c:L1398-L1406]`. For each entry in `cc_streams` the muxer emits one line:

  ```text
  #EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,GROUP-ID="<ccgroup>",NAME="<instreamid>",LANGUAGE="<language>",INSTREAM-ID="<instreamid>"
  ```

  The `INSTREAM-ID` value is repeated as the `NAME` attribute by default; the literal printf at `[libavformat/hlsenc.c:L1400]` is the emission point.
- Each variant carries a `ccgroup` string field on its `VariantStream` record (defined at `[libavformat/hlsenc.c:L120-L194]`); the value is set during variant-stream-map parsing when an entry contains a `ccgroup:` token. Variant `#EXT-X-STREAM-INF` emission at `[libavformat/hlsenc.c:L1484-L1496]` resolves `vs->ccgroup` and passes it into `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]`, which appends the `CLOSED-CAPTIONS="<ccgroup>"` attribute to the variant's stream-info line.

#### What this component does NOT do

This component covers only the *advertisement* of closed captions in the master playlist. The actual caption bytes are not extracted, repackaged, or rewritten by the HLS muxer; they remain embedded inside the video stream as the upstream encoder placed them. The muxer's responsibility ends at the playlist advertisement, after which the client player extracts captions from the decoded video using its own EIA-608/CTA-708 parser.

If the upstream encoder did *not* emit caption bytes into the video bitstream, advertising them in the playlist produces a discoverable-but-empty caption track — the player will see the `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` line, attempt to extract captions, and find nothing. The muxer does not validate that captions are actually present; it advertises what the configuration says is present and trusts the encoder.

#### Relationship to subtitle renditions

Closed captions and subtitles are two distinct mechanisms in HLS:

- **Closed captions** are embedded in the video bitstream (EIA-608 / CTA-708) and advertised via `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS`. The video bytes are unchanged; the player extracts captions during video decode.
- **Subtitles** are separate playlist tracks (often WebVTT) advertised via `#EXT-X-MEDIA:TYPE=SUBTITLES`. The subtitle bytes are delivered in their own segment files referenced by a separate variant playlist.

The muxer supports both, through different option surfaces. Closed captions use `cc_stream_map`; subtitles use `var_stream_map` entries with `s:<index>` stream selectors plus the `subtitle_varname` option to assign a rendition name. They are emitted by different writer functions: closed captions through the inline emission at `[libavformat/hlsenc.c:L1398-L1406]`, subtitles through `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]`.

Integrators who want translated captions in multiple languages typically use both: closed captions for the primary language carried in the video (already encoded as 608/708), and subtitle renditions for additional languages.

#### Configuration example walkthrough

A typical `cc_stream_map` configuration is:

```text
ccgroup:cc,instreamid:CC1,language:en ccgroup:cc,instreamid:CC2,language:es
```

This declares two closed-caption renditions in the same group named `cc`: one English caption on channel `CC1`, one Spanish caption on channel `CC2`. The variant-stream-map for a video tier would then reference the same group:

```text
v:0,a:0,ccgroup:cc
```

This binds video stream 0 and audio stream 0 to a variant that advertises the `cc` closed-caption group. The resulting master playlist would contain two `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` lines (one per language) and the variant's `#EXT-X-STREAM-INF` line would include a `CLOSED-CAPTIONS="cc"` attribute.

#### Cross-references

- The `ClosedCaptionsStream` and `VariantStream::ccgroup` field definitions live in [`../technical/data-model.md`](../technical/data-model.md).
- The `EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` wire-format contract (exact attribute order, quoting rules, character set) lives in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- The decision matrix for which `#EXT-X-STREAM-INF` lines emit a `CLOSED-CAPTIONS` attribute (versus omit it because no `ccgroup:` was bound) is enumerated in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The subtitle rendition mechanism (the alternative to closed captions for multi-language text tracks) is documented in §6 (Variant Stream Management).

## Component: Live vs VOD Mode Selection

An HLS playlist comes in three flavors that correspond to three publishing models. A *live* playlist describes a sliding window of recent segments and grows on one end while old segments fall off the other; a client polls the playlist periodically and follows the moving window. A *video-on-demand* (VOD) playlist describes a complete, finished recording from beginning to end and is closed with an explicit end-of-list tag; once published it never changes. An *event* playlist is append-only — it grows over time as new segments are added but never deletes any segments, allowing late-joining clients to seek back to the beginning of the event.

The muxer selects the flavor through the `hls_playlist_type` option and through several flag interactions. Live mode is the default when the option is left unset — the playlist contains only the most recent few segments and older segments are deleted from the head as the configured sliding-window length is exceeded. Setting the option to `vod` or `event` selects the other two flavors, and three additional flags (`omit_endlist`, `append_list`, and `delete_segments`) interact with the choice in non-obvious ways.

The default "live" configuration (no `hls_playlist_type` set, 5-segment window, no end-list emission) is appropriate for low-latency broadcast streaming where clients enter and leave the stream at arbitrary times. The "VOD" configuration is appropriate for replays of finished events where every client will play from the beginning to the end. The "event" configuration is appropriate for time-shifted live publications where clients can rewind to the start of the event but the publication is still being appended to (sports games, live concerts).

The interaction between `hls_playlist_type`, `HLS_OMIT_ENDLIST`, `HLS_DELETE_SEGMENTS`, and `HLS_APPEND_LIST` is the most error-prone surface in the entire muxer. Integrators who confuse "live with deletion" with "VOD" or who set `omit_endlist` on a VOD publication produce playlists that misbehave on strict-mode clients. The decision matrix is enumerated exhaustively in [`../technical/codec-logic.md`](../technical/codec-logic.md).

### Technical Detail

#### Playlist type enumeration

- The playlist-type enumeration is defined at `[libavformat/hlsplaylist.h:L31-L36]`: `PLAYLIST_TYPE_NONE = 0`, `PLAYLIST_TYPE_EVENT = 1`, `PLAYLIST_TYPE_VOD = 2`, `PLAYLIST_TYPE_NB = 3` (the count sentinel).
- The AVOption that selects the type is `hls_playlist_type` at `[libavformat/hlsenc.c:L3162]` (default `PLAYLIST_TYPE_NONE`); user-facing aliases `event` and `vod` are bound at `[libavformat/hlsenc.c:L3163-L3164]`.
- When the type is `EVENT` or `VOD`, the playlist-header writer (`ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]`) emits an `#EXT-X-PLAYLIST-TYPE:EVENT` or `#EXT-X-PLAYLIST-TYPE:VOD` line. When the type is `NONE` (the live-mode default), no `#EXT-X-PLAYLIST-TYPE` line is emitted.

#### Sliding-window behavior

- Sliding-window behavior in live mode is governed by `hls_list_size` (AVOption at `[libavformat/hlsenc.c:L3125]`, default 5 segments). When the per-variant segment list exceeds this size, `hls_delete_old_segments` at `[libavformat/hlsenc.c:L531]` unlinks the oldest segment files from disk (or issues `HTTP DELETE` requests through `hls->http_delete` when configured) and trims the linked list. The `hls_delete_threshold` AVOption at `[libavformat/hlsenc.c:L3126]` controls how many segments remain queued for deletion before being unlinked.
- The interaction between `hls_list_size` and `hls_delete_threshold` is subtle: the playlist always contains at most `hls_list_size` entries, but the muxer may retain up to `hls_list_size + hls_delete_threshold` segment files on disk. The grace buffer of `hls_delete_threshold` segments protects in-flight clients that loaded a slightly-stale playlist and are now trying to fetch a segment that has just rolled out of the window. With the default `hls_delete_threshold=1`, the muxer keeps one extra segment around past the window edge before unlinking it.
- A value of `hls_list_size=0` means "unlimited entries" — the playlist grows without bound, no segments are deleted. This is the typical VOD configuration; pairing it with `hls_playlist_type=vod` ensures the playlist also emits the `#EXT-X-PLAYLIST-TYPE:VOD` and (on trailer) `#EXT-X-ENDLIST` markers.

#### Flag interactions

- The `HLS_OMIT_ENDLIST` flag (AVOption const `omit_endlist` at `[libavformat/hlsenc.c:L3150]`, decimal value `1 << 4`) suppresses the final `#EXT-X-ENDLIST` line even when in VOD mode. This is useful for streams that may resume later — clients see a VOD-shaped playlist but the server reserves the right to add more segments.
- The `HLS_DELETE_SEGMENTS` flag (AVOption const `delete_segments` at `[libavformat/hlsenc.c:L3147]`, decimal value `1 << 1`) controls whether segment files are unlinked from disk when they roll out of the live window. Without the flag, the muxer trims the playlist but leaves the segment files in place — useful when an external archival process consumes the segments before deletion would orphan them.
- The `HLS_APPEND_LIST` flag (AVOption const `append_list` at `[libavformat/hlsenc.c:L3152]`, decimal value `1 << 6`) activates append-to-existing-playlist mode. When set, the muxer reads the existing playlist file via `parse_playlist` at `[libavformat/hlsenc.c:L1162]`, picks up where it left off (preserving the existing sequence number), seeds the wall-clock anchor by parsing any prior `#EXT-X-PROGRAM-DATE-TIME` line at `[libavformat/hlsenc.c:L1226-L1244]`, and forces a discontinuity at the resume point (see §9). This mode is typically combined with `EVENT` playlist type to produce a server that survives restarts without breaking client playback.

#### Atomic publish

- Live versus VOD also affects how the playlist file is delivered: with `HLS_TEMP_FILE` set, every publish in live mode writes to a `.tmp` file and renames atomically (so polling clients never see a half-written manifest); without the flag the muxer writes directly.
- For VOD publications, the temp-file approach is less critical because the playlist changes only at publication end; for live publications it is essential because clients poll the playlist on a 1-to-3-second cadence and a non-atomic update has a high probability of being observed mid-write.

#### Configuration recipes by mode

The following recipes summarize the option-and-flag combinations integrators most commonly use for each playback model. Recipes are sourced from the AVOption defaults and the flag interactions documented above.

| Mode | `hls_playlist_type` | `hls_list_size` | `HLS_DELETE_SEGMENTS` | `HLS_OMIT_ENDLIST` | `HLS_APPEND_LIST` | Effect |
|---|---|---|---|---|---|---|
| Live broadcast (low latency) | unset (`NONE`) | 5 (default) | yes (recommended) | no | no | Sliding window of 5 recent segments; old segment files unlinked from disk as they roll off |
| Live broadcast (archive on disk) | unset (`NONE`) | 5 (default) | no | no | no | Sliding window of 5 segments in playlist; all segment files retained on disk for later archive |
| VOD playback | `vod` | 0 (unlimited) | n/a (no deletion) | no | no | Playlist grows from 0 to N segments, then `#EXT-X-ENDLIST` is emitted at trailer |
| Event timeshift | `event` | 0 (unlimited) | n/a | no | no | Append-only playlist; clients can rewind to start |
| Restart-resilient live | `event` | 0 (unlimited) | n/a | no | yes | Muxer reads prior playlist on startup and continues appending after a `#EXT-X-DISCONTINUITY` |
| Pseudo-live VOD | `vod` | 0 (unlimited) | n/a | yes | no | Playlist looks VOD-shaped but lacks `#EXT-X-ENDLIST`, so clients keep polling |

#### Implications for the demuxer

The demuxer's polling and refresh policy is keyed off the `#EXT-X-PLAYLIST-TYPE` tag (or its absence) and the `#EXT-X-ENDLIST` tag. A VOD playlist (`#EXT-X-PLAYLIST-TYPE:VOD`) is downloaded once and never re-fetched. An EVENT playlist (`#EXT-X-PLAYLIST-TYPE:EVENT`) is re-fetched until `#EXT-X-ENDLIST` appears, with the parser appending new segments to the existing in-memory list on each refresh. An unspecified-type playlist (no `#EXT-X-PLAYLIST-TYPE` tag) is treated as live and re-fetched on the cadence implied by `#EXT-X-TARGETDURATION`, with the parser detecting and respecting any sliding-window deletions on the publisher side.

The demuxer's behavior is therefore symmetrical with the muxer's choice. Misconfiguration on the muxer side (e.g., a VOD publication that the publisher forgot to mark as VOD) manifests as a stuck-polling demuxer that never reaches end-of-file.

#### Cross-references

- The full decision matrix for `hls_playlist_type` × `HLS_OMIT_ENDLIST` × `HLS_DELETE_SEGMENTS` × `HLS_APPEND_LIST` combinations is enumerated in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The Mermaid flowchart of the live sliding-window control flow lives in [`../technical/process-flows.md`](../technical/process-flows.md).
- The `EXT-X-ENDLIST` and `EXT-X-PLAYLIST-TYPE` invariants (when they are emitted, what their absence means) live in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).
- The append-list discontinuity handling cross-references with §9 (Discontinuity Handling).
- The demuxer's matching refresh-policy logic lives in §11 (HLS Demuxer Playlist Parser).

## Component: Discontinuity Handling

HLS playlists can declare *discontinuities* — points in the segment stream where one or more of the bitstream properties change abruptly. Typical reasons include: timestamps reset (e.g., a server restart that re-anchors PTS at zero), codec parameters change (e.g., switching from one input source to another), or content boundaries (e.g., the end of one program and the start of the next in a 24/7 channel). The HLS specification requires the playlist to mark these transitions with a `#EXT-X-DISCONTINUITY` tag so the client player flushes its decoder state and re-initializes its bitstream parsers at the boundary.

The muxer emits a discontinuity tag in two situations. First, when the publisher knows up-front that the playlist will be a continuation of a prior stream and the prior stream is no longer guaranteed to share the same bitstream parameters — typically signaled by enabling the `discont_start` flag or by enabling append-list mode. Second, when an individual segment has an internal discontinuity flag set, which propagates a `#EXT-X-DISCONTINUITY` line in front of that segment's entry in the playlist.

Discontinuities are intentionally coarse: a single `#EXT-X-DISCONTINUITY` line tells the client *something* changed at the boundary, but does not specify what. Clients respond by tearing down their decoder pipeline and re-initializing it from the next segment's codec parameters. This is expensive (it forces a re-keyframe-wait on the client side) but safe: a client that did not flush state across a boundary risks producing visual or audio artifacts. Integrators should therefore emit discontinuities only when an actual change occurs; spurious discontinuities cause client-side stuttering.

The `discont_program_date_time` mechanism is a related-but-separate concern: when wall-clock-anchored timestamps are in use (the `HLS_PROGRAM_DATE_TIME` flag), a discontinuity at the start of a resumed publication must also re-anchor the wall-clock so the resumed segments are not advertised as occurring at the original publication's wall-clock time. This is what `parse_playlist` does in append-list mode: it reads the *prior* publication's last `#EXT-X-PROGRAM-DATE-TIME` and uses it to seed the resumed publication's starting anchor.

### Technical Detail

#### Top-of-playlist discontinuity

- The `HLS_DISCONT_START` flag (AVOption const `discont_start` at `[libavformat/hlsenc.c:L3149]`, decimal value `1 << 3`) requests a `#EXT-X-DISCONTINUITY` line at the top of the playlist's segment list. The emission point is `[libavformat/hlsenc.c:L1593-L1596]`; emission is gated on the flag being set, the current sequence equaling `hls->start_sequence`, and the per-variant `discontinuity_set` field being zero (so the line is emitted exactly once).
- The `discontinuity_set` flag on `VariantStream` at `[libavformat/hlsenc.c:L150]` is the "we have already emitted the start-of-playlist discontinuity for this variant" memo. It is set to 1 after the first emission so the discontinuity is not re-emitted on every playlist refresh.

#### Per-segment discontinuity propagation

- A per-segment discontinuity is carried on the `HLSSegment::discont` field (struct at `[libavformat/hlsenc.c:L76-L94]`, declared `int` at L82). The propagation from `vs->discontinuity` (transient variant-level flag) to `en->discont` (persistent per-segment flag) happens inside `hls_append_segment` at `[libavformat/hlsenc.c:L1090-L1093]`:

  ```c
  if (vs->discontinuity) { en->discont = 1; vs->discontinuity = 0; }
  ```

  The variant-level flag is set in several places: by `parse_playlist` when resuming an existing list (append-list mode), and at any point in `hls_write_packet` where the muxer detects a timestamp gap exceeding a threshold.
- The per-segment emission lives inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L156-L158]`: when `insert_discont` (the writer's first parameter, populated from `en->discont`) is nonzero, `#EXT-X-DISCONTINUITY\n` is emitted on its own line *before* the `#EXTINF` line for that segment.
- The line MUST precede the segment entry it applies to — placing it after would cause clients to interpret it as applying to the *next* segment. The single-line-before-segment ordering is enforced by the writer function's call sequence.

#### Wall-clock re-anchoring

- When the `HLS_PROGRAM_DATE_TIME` flag (AVOption const `program_date_time` at `[libavformat/hlsenc.c:L3153]`, decimal value `1 << 7`) is enabled together with a discontinuity, the wall-clock anchor must be recomputed because the discontinuity may include a clock jump. The recompute path is in `parse_playlist` at `[libavformat/hlsenc.c:L1226-L1244]` (parsing the existing playlist's last `#EXT-X-PROGRAM-DATE-TIME` and converting it to a `discont_program_date_time` value via `av_timegm` and the per-segment duration). The new anchor is then emitted by `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L167-L193]` with millisecond precision in ISO-8601 format including a timezone offset.
- The wall-clock emission format is `#EXT-X-PROGRAM-DATE-TIME:YYYY-MM-DDTHH:MM:SS.mmm+HHMM` — millisecond precision, RFC-3339 / ISO-8601 with a numeric timezone offset (no `Z` suffix). The format string lives in `hlsplaylist.c`; see [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md) for the exact wire-format contract.

#### Append-list restart discontinuity

- The `HLS_APPEND_LIST` mode (see §8) automatically sets `vs->discontinuity = 1` after reading the existing playlist, which propagates to the next segment's `discont` flag and produces a `#EXT-X-DISCONTINUITY` line at the resume point. This is the muxer's default behavior on restart and ensures clients flush their decoder state at the boundary.
- The reasoning is conservative: the muxer cannot guarantee that the restarted publication shares the same encoder parameters as the prior one. Even when the restart is intended to be uninterrupted from the client's perspective, the safer behavior is to mark the boundary as a discontinuity and let clients re-initialize their decoders.

#### Emission conditions summary

The full set of conditions under which the muxer emits a `#EXT-X-DISCONTINUITY` line is:

| Condition | Location | When | One-shot? |
|---|---|---|---|
| `HLS_DISCONT_START` flag set, first publish | `[libavformat/hlsenc.c:L1593-L1596]` | At top of playlist when `vs->sequence == hls->start_sequence` and `vs->discontinuity_set == 0` | Yes — `discontinuity_set` is set to 1 after first emission |
| Per-segment `discont` flag set via append-list resume | `[libavformat/hlsplaylist.c:L156-L158]` | Before the segment entry's `#EXTINF` line | Yes — only the segment that has `en->discont == 1` triggers emission |
| Per-segment `discont` flag set via timestamp gap detection | `[libavformat/hlsplaylist.c:L156-L158]` | Before the segment entry's `#EXTINF` line | Yes — same per-segment emission |

The muxer never emits `#EXT-X-DISCONTINUITY` *between* multiple successive segments unless each one has its own `discont` flag set. The flag does not "cascade" through subsequent segments.

#### Client behavior expectations

When a client encounters `#EXT-X-DISCONTINUITY` in a playlist, the HLS specification requires the client to:

- Flush its decoder state for the affected stream (clear buffers, reset reference frames, drop accumulated PTS offsets).
- Re-initialize the decoder using the codec parameters of the next segment (extradata, sample rate, channel layout, etc.).
- Wait for the next keyframe in the new segment before producing decoded output (this can introduce a brief blackout on screen).
- Re-anchor wall-clock display time using the segment's `#EXT-X-PROGRAM-DATE-TIME` value if present, or compute it from the previous segment's wall-clock plus accumulated duration if not.

This client cost is the reason discontinuities should be emitted sparingly. Each one introduces a brief glitch on the client side; an over-eager publisher that emits discontinuities at every segment boundary effectively breaks adaptive playback.

#### Cross-references

- The decision matrix for when `#EXT-X-DISCONTINUITY` is emitted (top-of-playlist vs per-segment, with vs without `PROGRAM-DATE-TIME` recompute) lives in [`../technical/codec-logic.md`](../technical/codec-logic.md).
- The processing-order constraint "discontinuity line must precede the affected segment entry" is a Layer 3 contract documented in [`../api-contracts/timing-dependencies.md`](../api-contracts/timing-dependencies.md).
- The `HLSSegment::discont` and `HLSSegment::discont_program_date_time` field definitions live in [`../technical/data-model.md`](../technical/data-model.md).
- The append-list mode's interaction with discontinuity is also documented in §8 (Live vs VOD Mode Selection).


## Component: HLS Demuxer Probe

When an application opens an input file or URL through `avformat_open_input` and does not specify which demuxer to use, FFmpeg samples the first several kilobytes of the input and asks each registered demuxer "is this its format?" through a probe callback. The HLS demuxer's probe inspects the sampled bytes for the M3U8 magic line plus one or more HLS-specific tags that distinguish HLS playlists from generic plain-text M3U files used by other audio players.

A successful probe returns a confidence score; the demuxer with the highest score is selected. The HLS probe is deliberately strict about which tags it accepts as evidence — a file beginning with `#EXTM3U` but containing nothing else is not HLS (it could be a Winamp-style playlist), but a file containing `#EXT-X-STREAM-INF`, `#EXT-X-TARGETDURATION`, `#EXT-X-MEDIA-SEQUENCE`, or `#EXTINF` along with the magic line is unambiguously HLS.

The probe is the first line of defense against format misidentification. Without it, an application opening a Winamp `.m3u` playlist by name might find FFmpeg invoking the HLS demuxer, which would then attempt to parse a totally unrelated playlist format and fail in confusing ways. The probe's strictness ensures that the HLS demuxer is selected only when the input is unambiguously an HLS playlist.

Integrators rarely interact with the probe directly. The probe runs implicitly inside `avformat_open_input` and its result either selects the HLS demuxer or falls through to other candidates. Integrators who want to *force* HLS demuxing (e.g., when reading from a network source that does not have a recognizable file extension) can pass `-f hls` to `ffmpeg` or call `av_find_input_format("hls")` and pass the result to `avformat_open_input`'s `fmt` parameter.

### Technical Detail

#### Probe function

- The probe function is `hls_probe` at `[libavformat/hls.c:L2814]`. It inspects `AVProbeData::buf` (the sampled buffer) for the `#EXTM3U` magic line plus discriminating HLS-specific tags.
- The function returns one of three confidence scores:
  - `0` — the input does not look like HLS at all (no magic line, or magic line but no HLS-specific tags). The demuxer abstains; FFmpeg tries other candidates.
  - `AVPROBE_SCORE_MAX` — the input is unambiguously HLS (magic line plus at least one HLS-specific tag). The demuxer wins the probe.
  - Intermediate scores — currently the probe does not emit intermediate scores; it is binary.
- The probe is purely text-based and never reads the network. It operates only on the locally-sampled bytes already in `AVProbeData::buf`. The buffer is filled by libavformat's probe machinery before any demuxer's probe callback is invoked; the size depends on the input but is typically several kilobytes — enough to capture any reasonable M3U8 header.

#### Registration

- The demuxer is registered at `[libavformat/hls.c:L2900-L2912]` as `const FFInputFormat ff_hls_demuxer` with `.p.name = "hls"`, `.p.long_name = "Apple HTTP Live Streaming"`, `.read_probe = hls_probe` slot at `[libavformat/hls.c:L2907]`, plus a `.priv_data_size = sizeof(HLSContext)` allocation hint and a `.flags_internal = FF_INFMT_FLAG_INIT_CLEANUP` hint that calls `read_close` even on a `read_header` failure.
- The `FF_INFMT_FLAG_INIT_CLEANUP` flag is important for resilience: when `hls_read_header` fails partway through (e.g., after allocating the variant array but before parsing the playlist), the framework still calls `hls_close` to clean up. Without this flag, the partial state would leak.

#### Registration flags

Registration flags `.p.flags = AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`:

- `AVFMT_NOGENSEARCH` — disables the generic byte-offset search; HLS does not support byte-offset seeking into the abstract media timeline. Seeking is by segment index (and by extension, by timestamp once the index-to-time mapping is known), not by byte offset.
- `AVFMT_TS_DISCONT` — declares that timestamp discontinuities across segment boundaries are normal and expected (downstream timestamp consumers should not interpret them as errors). Without this flag, FFmpeg's generic frame-timestamp validation would emit warnings every time a discontinuity crossed.
- `AVFMT_NO_BYTE_SEEK` — declares that seeking is by segment index, not by byte offset. Together with `AVFMT_NOGENSEARCH`, this informs the seeking machinery to use the demuxer's own `read_seek` callback rather than attempting generic byte-search seeks.
- `AVFMT_SHOW_IDS` — declares that stream IDs assigned by the demuxer are meaningful and should be exposed to the application. HLS variants and renditions have stable IDs that map to specific playlist URLs; surfacing them lets applications correlate decoded streams with their playlist origin.

#### What happens after a successful probe

- Once the probe confirms the input looks like HLS, FFmpeg invokes `avformat_open_input`'s next step (allocating the demuxer's private data and calling `read_header`). The demuxer takes over with `hls_read_header` at `[libavformat/hls.c:L2144]` (see §11).
- The probe's confidence score is recorded on the `AVFormatContext` so applications can inspect it; in practice most applications check only whether the input was opened successfully, not the probe's confidence score.

#### Tag-recognition matrix

The probe's strictness derives from the set of tags it accepts as evidence of HLS. The recognition logic is essentially: "if the buffer contains `#EXTM3U` AND at least one of the following HLS-specific tags, return `AVPROBE_SCORE_MAX`":

| Tag prefix recognized | HLS-specific? | Reason |
|---|---|---|
| `#EXTM3U` alone | No — necessary but not sufficient | Generic M3U playlist marker, used by Winamp and many audio players |
| `#EXT-X-STREAM-INF` | Yes | HLS master-playlist tag declaring a variant stream |
| `#EXT-X-TARGETDURATION` | Yes | HLS media-playlist tag declaring max segment duration |
| `#EXT-X-MEDIA-SEQUENCE` | Yes | HLS media-playlist tag declaring the first segment's sequence number |
| `#EXTINF` | Partially — also used by some podcast playlists | The combination with `#EXTM3U` is treated as HLS-likely |

The probe does not look at file extension, MIME type, or URL pattern — only at the buffer content. This is appropriate because the same M3U8 file might be served with `Content-Type: application/octet-stream`, `application/vnd.apple.mpegurl`, or `audio/x-mpegurl` depending on the server, and the demuxer must work in all cases.

#### Demuxer registration vs `av_find_input_format`

The probe-based selection described above is what runs when `avformat_open_input` is called with `fmt = NULL` (the default). When the application instead explicitly passes a result from `av_find_input_format("hls")`, the probe step is skipped entirely and the HLS demuxer is forced. The probe is also bypassed when:

- The input URL has the `hls:` pseudo-protocol prefix (some applications use `hls:http://example.com/m.m3u8` for explicit selection).
- The application sets `AVFormatContext::iformat` directly before calling `avformat_open_input`.
- The `ffmpeg` CLI tool is invoked with `-f hls` on the input side.

In all forced cases, `hls_probe` is not called. The demuxer takes over directly with `hls_read_header`, which may then fail with `AVERROR_INVALIDDATA` if the input is not actually an HLS playlist — but the probe-based opt-out has been waived.

#### Cross-references

- The full demuxer lifecycle (probe → read_header → read_packet loop → read_close) is narrated in [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md).
- The probe-failure path (when the input is *not* HLS — what the demuxer does, what other candidates may be tried) is documented in [`./exception-handling.md`](./exception-handling.md).
- The detailed work of `hls_read_header` (and the parser it invokes) is documented in §11 (HLS Demuxer Playlist Parser).

## Component: HLS Demuxer Playlist Parser

After the probe confirms the input is HLS, the demuxer's main job is to translate the M3U8 manifest from text into in-memory data structures the rest of the demuxer reads from. The playlist parser scans the manifest line by line, recognizes every `#EXT-X-*` tag defined by the HLS specification (RFC 8216), and builds three linked structures: a list of *variants* (the alternative bitrate ladders advertised in the master playlist), a list of *renditions* (the alternative audio and subtitle tracks), and a list of *playlists* (each containing a list of segments with their URLs, durations, and encryption metadata).

The parser is reentrant: it can be called repeatedly to refresh a live playlist, picking up new segments at the bottom while preserving any state already learned from prior parses. This is essential for live mode where the demuxer must poll the playlist URL at the cadence implied by `#EXT-X-TARGETDURATION` and pick up newly added segments without restarting.

The demuxer architecture is layered: the parser builds the in-memory representation, the per-playlist sub-demuxer state machine fetches segments and demuxes them through nested MPEG-TS or fMP4 sub-demuxers, and the top-level `hls_read_packet` callback interleaves packets from all active playlists by PTS-smallest-first. From an integrator's perspective, the demuxer behaves like any other libavformat demuxer — `av_read_frame` returns the next packet, `av_seek_frame` seeks, etc. — but its internal complexity is significantly higher than a single-file demuxer because it must coordinate network I/O, segment-boundary state resets, and live-playlist refresh.

The demuxer is *read-only*: it does not transcode, re-encode, or modify segment bytes. Its output is raw `AVPacket` values that downstream decoders consume. Sample-AES-encrypted streams are the one exception: when the parser sees `#EXT-X-KEY:METHOD=SAMPLE-AES` on a segment, the demuxer invokes the sample-encryption transform (§12) to decrypt the affected samples in place before emitting them as decoded packets.

### Technical Detail

#### Lifecycle entry points

Lifecycle entry points (all in `[libavformat/hls.c]`):

- `hls_read_header` at `[libavformat/hls.c:L2144]` — invoked once by `avformat_open_input` after a successful probe. Calls the parser on the master playlist, opens sub-demuxers for each selected variant, and registers each elementary stream with the application.
- `hls_read_packet` at `[libavformat/hls.c:L2546]` — invoked by `av_read_frame`. Picks the next playlist whose accumulated PTS is smallest, reads the next packet from that playlist's sub-demuxer (re-fetching the next segment from the network if needed), and returns the packet to the application.
- `hls_read_seek` at `[libavformat/hls.c:L2709]` — invoked by `av_seek_frame`. Maps a target timestamp to a segment index, restarts the affected sub-demuxers at the matching segment, and clears the queued packet state.
- `hls_close` at `[libavformat/hls.c:L2127]` — invoked on close. Frees all parsed playlist data, closes sub-demuxers, and releases network state.

#### Parser function

- The `parse_playlist` function starts at `[libavformat/hls.c:L787]` and runs through approximately L1080. The line-by-line tag-dispatch loop body — where each parsed line is matched against a series of tag prefixes and dispatched to a per-tag handler that updates the in-progress `struct playlist` record — lives around `[libavformat/hls.c:L860-L989]`.
- The function takes the demuxer context, the playlist URL, an optional `playlist *` pointer (for re-parsing an existing media playlist) and an optional `AVIOContext *` (for parsing from an already-open input rather than re-opening the URL). The dual modes support the master-playlist-first-read and the live-playlist-refresh-loop respectively.
- The line-dispatch loop uses `av_strstart` to test each known tag prefix (`#EXT-X-STREAM-INF:`, `#EXT-X-MEDIA:`, `#EXT-X-TARGETDURATION:`, `#EXT-X-MEDIA-SEQUENCE:`, `#EXTINF:`, `#EXT-X-KEY:`, `#EXT-X-DISCONTINUITY`, `#EXT-X-ENDLIST`, `#EXT-X-PLAYLIST-TYPE:`, `#EXT-X-MAP:`, `#EXT-X-BYTERANGE:`, `#EXT-X-PROGRAM-DATE-TIME:`, `#EXT-X-ALLOW-CACHE:`, `#EXT-X-VERSION:`, `#EXT-X-START:`, etc.) and dispatches to the matching handler block.

#### Key data structures

- `struct segment` at `[libavformat/hls.c:L77-L87]` — per-segment record with `duration`, `url_offset`, `size`, `url`, `key`, `key_type`, `iv[16]`, `init_section`. The `key` and `iv` fields carry the AES-128 key material for the segment when `key_type` is `KEY_AES_128` or `KEY_SAMPLE_AES`; `init_section` references the optional `#EXT-X-MAP` initialization segment for fMP4 playlists.
- `struct playlist` at `[libavformat/hls.c:L102-L177]` — per-media-playlist state including `url`, `pb` (the playlist's own AVIOContext for fetching), `read_buffer`, `input`/`input_next` (segment fetchers), `parent` (back-pointer to the demuxer's `AVFormatContext`), `ctx` (the sub-demuxer's `AVFormatContext`), `n_segments`/`segments`, `cur_seq_no`/`last_seq_no` (current and last-known sequence numbers), `m3u8_hold_counters` (retry budget for unchanged live playlists), `key[16]`/`key_url`, ID3 state for ID3-timestamped audio streams, `audio_setup_info` (Sample-AES audio metadata), `n_renditions`/`renditions`, `n_init_sections`/`init_sections`.
- `struct rendition` at `[libavformat/hls.c:L185-L192]` — alternative audio/subtitle/closed-captions track with `type`, back-pointer `playlist`, `group_id`, `language`, `name`, `disposition` (matched against the master playlist's `#EXT-X-MEDIA` lines).
- `struct variant` at `[libavformat/hls.c:L194-L204]` — top-level master-playlist variant with `bandwidth`, list of `playlists`, plus `audio_group`/`video_group`/`subtitles_group` matching strings.
- `HLSContext` (demuxer top level) at `[libavformat/hls.c:L206+]` — owns the `AVClass`, the parent `AVFormatContext`, the variants array, the playlists array, the renditions array.

#### Enumerations

- Encryption-method enumeration at `[libavformat/hls.c:L71-L75]`: `KEY_NONE = 0`, `KEY_AES_128 = 1`, `KEY_SAMPLE_AES = 2`. Set on each `struct segment::key_type` field as the parser sees `#EXT-X-KEY:METHOD=AES-128` or `#EXT-X-KEY:METHOD=SAMPLE-AES` lines. `KEY_NONE` means the segment is unencrypted.
- Playlist-type enumeration (demuxer-side) at `[libavformat/hls.c:L91-L95]`: `PLS_TYPE_UNSPECIFIED = 0`, `PLS_TYPE_EVENT = 1`, `PLS_TYPE_VOD = 2`. Set on each `struct playlist::type` field from `#EXT-X-PLAYLIST-TYPE` lines. The demuxer uses this to decide refresh policy: VOD playlists are never re-parsed, EVENT playlists may be re-parsed for new segments, UNSPECIFIED playlists are treated as live and re-parsed on the cadence implied by `#EXT-X-TARGETDURATION`.

#### Live-mode refresh

- Live-mode refresh loop is bounded by `m3u8_hold_counters` (AVOption at `[libavformat/hls.c:L2878-L2879]`, default 1000). When the parser re-reads a live playlist and observes no new segments, it decrements the counter and retries after a delay; reaching zero terminates the stream with `AVERROR_EOF` (the server is no longer publishing).
- The retry delay scales with `#EXT-X-TARGETDURATION`: typical refresh cadence is half the target duration (e.g., 1 second between refreshes for a 2-second-target playlist). The exact computation reads the parsed target duration and applies a small jitter to avoid synchronized polling from many clients.

#### Specification reference

- The RFC 8216 specification URL is referenced in the file-level Doxygen comment at `[libavformat/hls.c:L26]` (the surrounding `@file` block documents the demuxer as "Apple HTTP Live Streaming demuxer" with the RFC URL on the adjacent line).
- The parser implementation aims for spec compliance; deviations or undefined-by-spec corner cases are sometimes documented inline as comments. Any new HLS spec revision must be cross-checked against the parser's tag-dispatch loop to ensure new tags are recognized.

#### Cross-references

- The full inventory of tags accepted by the parser is enumerated in [`./inputs-outputs.md`](./inputs-outputs.md) §10 (Demuxer Input Table).
- The demuxer's sub-demuxer integration (how each `struct playlist::ctx` is bound to an MPEG-TS or fMP4 demuxer) is documented in [`../technical/integration-interfaces.md`](../technical/integration-interfaces.md).
- The Mermaid sequence diagram of `hls_read_packet`'s playlist-selection logic lives in [`../technical/process-flows.md`](../technical/process-flows.md).
- The struct field dictionary for `segment`, `playlist`, `variant`, `rendition`, and the demuxer-side `HLSContext` lives in [`../technical/data-model.md`](../technical/data-model.md).

## Component: HLS Sample Encryption

HLS also supports a second encryption scheme called *Sample Encryption* or *Sample-AES*. Where ordinary HLS encryption (see §5) encrypts an entire segment file with AES-128 in CBC mode (so the segment is opaque without the key), Sample-AES encrypts only the *samples* — the audio frames or the encoded video samples inside a segment — leaving the MPEG transport-stream packet headers and the segment's structural framing in clear text. The result is that a Sample-AES-aware demuxer can parse the segment structure without the key (it can read the timing and stream layout) but the application's decoders cannot actually decode anything without the key.

The HLS demuxer supports Sample-AES decryption for H.264 video and for AAC, AC-3, and E-AC-3 audio. Sample-AES is signaled at multiple levels: the playlist tags the affected stream with `#EXT-X-KEY:METHOD=SAMPLE-AES`, the transport-stream PMT advertises a Sample-AES-specific stream type (a value in the range reserved by the HLS Sample Encryption specification), and for AAC streams an ID3 PRIV tag carrying audio setup information is prepended to each segment so the demuxer knows codec parameters before decoder negotiation.

From a system-design perspective, Sample-AES is fundamentally different from the segment-level AES-128 scheme. Segment-level AES-128 is a *transport* protection scheme: the segment file is opaque on disk and on the wire, and any HLS player without the key cannot even read the container. Sample-AES is a *codec* protection scheme: the container is fully readable and parseable, but the elementary-stream payloads embedded in it are encrypted. This means Sample-AES requires codec-aware decryption that walks individual NAL units (for H.264) or frame headers (for AAC/AC-3/E-AC-3) and decrypts the *payload* portion while preserving the *framing* bytes. The codec awareness is why the transform lives in its own translation unit separate from the generic AES helpers in `libavutil/aes.h`.

The Sample-AES specification cited in the header file's `@file` Doxygen block is Apple's "HLS Sample Encryption" specification — distinct from RFC 8216 — published at `https://developer.apple.com/library/ios/documentation/AudioVideo/Conceptual/HLS_Sample_Encryption`. Reproducing exact byte-for-byte behavior of this scheme requires careful attention to the per-codec encryption patterns (different sub-block selection rules for video vs audio).

### Technical Detail

#### Implementation file

- Implementation file: `[libavformat/hls_sample_encryption.c]` (396 lines). It implements three exported functions plus internal helpers for ADTS parsing, AC-3 parsing, and H.264 NAL unit walking.
- Header file: `[libavformat/hls_sample_encryption.h]` (65 lines) declaring the three exported functions and the two state structs.

#### Exported function prototypes

Exported function prototypes are at `[libavformat/hls_sample_encryption.h:L59-L63]`:

- `ff_hls_senc_read_audio_setup_info` at `[libavformat/hls_sample_encryption.c:L61]` — parses the audio setup blob delivered as ID3 metadata. Called when the demuxer encounters an ID3 PRIV tag with the HLS-specific owner identifier at the front of a Sample-AES audio segment.
- `ff_hls_senc_parse_audio_setup_info` at `[libavformat/hls_sample_encryption.c:L94]` — translates the parsed blob into AVStream codec parameters. Sets the AAC profile, channel configuration, and priming sample count on the relevant `AVStream`.
- `ff_hls_senc_decrypt_frame` at `[libavformat/hls_sample_encryption.c:L388]` — performs the actual per-frame Sample-AES decryption: walks NAL units for H.264 or audio frame headers for AAC/AC-3/E-AC-3 and decrypts the body region in-place. Returns 0 on success, `AVERROR(EINVAL)` if the per-codec framing parser cannot recognize the frame structure.

#### State structs

- Crypto state struct `HLSCryptoContext` at `[libavformat/hls_sample_encryption.h:L43-L47]` carries the AES context pointer (`AVAES *aes_ctx`) plus the 16-byte key and 16-byte IV buffers. The AES context is owned by the struct; the caller is responsible for initializing it with the key bytes before invoking `ff_hls_senc_decrypt_frame`.
- Audio setup struct `HLSAudioSetupInfo` at `[libavformat/hls_sample_encryption.h:L49-L56]` carries the codec identifier, codec tag, priming sample count, version, setup-data length, and a 10-byte (plus padding) setup-data buffer populated from the ID3 PRIV tag prepended to each AAC segment. The `priming` field is critical for AAC: it specifies how many samples at the start of the stream were synthesized as part of encoder lookahead and must be discarded before playback to avoid an audible "pop".

#### Constants

- Constants at `[libavformat/hls_sample_encryption.h:L40-L41]`: `HLS_MAX_ID3_TAGS_DATA_LEN = 138`, `HLS_MAX_AUDIO_SETUP_DATA_LEN = 10`. These bound the size of the ID3 PRIV payload the demuxer accepts; oversized payloads are rejected to prevent memory expansion attacks via a maliciously crafted stream.

#### MPEG-TS stream-type signaling

The MPEG transport-stream stream-type values that signal HLS Sample-AES in a PMT live at `[libavformat/mpegts.h:L177-L180]`:

| Stream Type | Value | Meaning |
|-------------|-------|---------|
| `STREAM_TYPE_HLS_SE_VIDEO_H264` | `0xdb` | H.264 video, Sample-AES encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_AAC` | `0xcf` | AAC audio, Sample-AES encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_AC3` | `0xc1` | AC-3 audio, Sample-AES encrypted |
| `STREAM_TYPE_HLS_SE_AUDIO_EAC3` | `0xc2` | Enhanced AC-3 audio, Sample-AES encrypted |

These values are read by the MPEG-TS sub-demuxer when parsing the PMT and dispatched to `ff_hls_senc_decrypt_frame` for each affected packet.

#### Demuxer integration

- The demuxer-side `struct playlist` (at `[libavformat/hls.c:L102-L177]`) carries the per-playlist `audio_setup_info` field which is populated by `ff_hls_senc_read_audio_setup_info` when an ID3 PRIV tag is encountered at segment boundaries.
- The MPEG-TS sub-demuxer, when it parses a PMT containing one of the four `STREAM_TYPE_HLS_SE_*` values, sets a per-stream flag indicating Sample-AES encryption. The HLS demuxer's read-packet path checks this flag for each emitted packet and, if set, calls `ff_hls_senc_decrypt_frame` before returning the packet to the application.
- Each segment's encryption key is taken from the `struct segment::key` and `struct segment::iv` fields populated by the parser when it sees `#EXT-X-KEY:METHOD=SAMPLE-AES,URI="...",IV=0x...` lines.

#### Build-system scoping (muxer vs demuxer)

- **Sample Encryption is a demuxer-only feature in this codebase.** The build system at `[libavformat/Makefile:L276-L277]` links `hls_sample_encryption.o` only into the HLS demuxer object set (`OBJS-$(CONFIG_HLS_DEMUXER) += hls.o hls_sample_encryption.o`), not into the HLS muxer (`OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`). The muxer does not natively *produce* Sample-AES-encrypted segments; for that workflow the publisher must use a separate Sample-AES encryptor and feed the encrypted segments through a generic file-publishing path.
- This asymmetry is significant: an integrator who needs to publish Sample-AES content cannot rely on FFmpeg's HLS muxer for the encryption step; they must arrange for an external Sample-AES tool to encrypt the segments and only use FFmpeg for the surrounding workflow.

#### Cross-references

- The data-contract reference for `HLSCryptoContext`, `HLSAudioSetupInfo`, and the `STREAM_TYPE_HLS_SE_*` values lives in [`../api-contracts/data-contracts.md`](../api-contracts/data-contracts.md).
- The integration contract for the Sample-AES transport scheme (what the demuxer expects from a Sample-AES-aware publisher) lives in [`../api-contracts/integration-contracts.md`](../api-contracts/integration-contracts.md).
- The full struct-field dictionary for `HLSCryptoContext` and `HLSAudioSetupInfo` lives in [`../technical/data-model.md`](../technical/data-model.md).


## Component: Playlist Tag Writers (Shared with DASH Muxer)

The actual `#EXT-X-*` lines of an M3U8 playlist file are not emitted by the muxer's main translation unit directly. Instead, the muxer calls into a small set of helper functions living in a separate translation unit, each of which emits one specific category of HLS tag. This factoring exists because the FFmpeg DASH muxer also publishes a side-channel M3U8 playlist alongside its primary DASH manifest (for clients that prefer HLS over DASH), and both muxers must produce byte-identical playlist output for the tags they share. Centralizing the writers guarantees both formats stay in lockstep.

This component is therefore a *shared library*, not a private muxer subroutine. The same code path executes whether the user is publishing through the HLS muxer or through the DASH muxer's HLS side-channel, which means: any change to the wire-format output of these helpers affects both formats simultaneously. The eight writer functions cover the full set of standard HLS tags the FFmpeg muxers produce.

From an integrator's perspective, this is the component that most directly defines what an HLS client will see "on the wire". Every byte of every M3U8 file produced by either muxer flows through one of these eight functions. Anyone porting the muxer to a different runtime must reproduce the exact byte-for-byte output of each writer; anyone validating an HLS publish for spec conformance can read the writers' source to enumerate the exact tags that appear and the exact format of their attributes.

The DASH muxer's reuse of these writers is the single most surprising cross-format dependency in the FFmpeg formats layer. It means a refactor of any writer function affects two visible output formats simultaneously — a critical fact for risk-scoping any change.

### Technical Detail

#### Translation unit and prototypes

- Translation unit: `[libavformat/hlsplaylist.c]` (206 lines) with prototypes in `[libavformat/hlsplaylist.h]` (65 lines).
- The eight exported writer functions are declared at `[libavformat/hlsplaylist.h:L38-L63]`. They are listed below in roughly the order they are called when publishing a typical multi-variant HLS stream.

#### The eight writer functions

1. `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` — emits `#EXTM3U` on line 1 and `#EXT-X-VERSION:%d` on line 2. Always the first two lines of every playlist. The version number is negotiated by the muxer's feature-detection logic and passed in as a parameter; the writer does not negotiate the version itself.

2. `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L40-L56]` — emits one `#EXT-X-MEDIA:TYPE=AUDIO,...` line per audio-only rendition. Attributes: `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE`, `CHANNELS`, `URI`. Called once per audio rendition during master-playlist generation; the renditions referenced by `EXT-X-STREAM-INF` lines must be declared first.

3. `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]` — emits one `#EXT-X-MEDIA:TYPE=SUBTITLES,...` line per subtitle rendition. Attributes: `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE`, `URI`. Called once per subtitle rendition during master-playlist generation.

4. `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]` — emits one `#EXT-X-STREAM-INF:...` line per variant in the master playlist. Attributes: `BANDWIDTH`, `AVERAGE-BANDWIDTH`, `RESOLUTION=WxH`, `FRAME-RATE`, `CODECS`, optional `AUDIO`/`SUBTITLES`/`CLOSED-CAPTIONS` group references. The variant's playlist URL follows on the next line. This is the writer whose output most directly determines a player's bitrate-ladder presentation.

5. `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]` — emits (after calling `ff_hls_write_playlist_version` internally for the version line) the fixed top-of-playlist tags: `#EXT-X-ALLOW-CACHE`, `#EXT-X-TARGETDURATION:%d`, `#EXT-X-MEDIA-SEQUENCE:%lld`, optional `#EXT-X-PLAYLIST-TYPE`, optional `#EXT-X-I-FRAMES-ONLY`. Called once at the top of each media playlist (per variant) before any segment entries.

6. `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]` — emits the `#EXT-X-MAP:URI="<file>",BYTERANGE="<size>@<offset>"` line that references the fragmented-MP4 initialization segment. Called once per playlist when `segment_type=fmp4`. For MPEG-TS playlists this writer is not called; an MPEG-TS playlist has no `#EXT-X-MAP` line because each TS segment is self-contained.

7. `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L144-L199]` — emits the per-segment block: optional `#EXT-X-DISCONTINUITY`, the `#EXTINF:%f,` duration line, optional `#EXT-X-BYTERANGE:%lld@%lld`, optional `#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s` with timezone-offset suffix, and the segment URL on its own line. Called once per segment listed in the playlist; this is the writer invoked most frequently during live publishing.

8. `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]` — emits the `#EXT-X-ENDLIST` line. Called once at the end of a finished playlist (VOD mode or trailer publish with `HLS_OMIT_ENDLIST` not set). When `HLS_OMIT_ENDLIST` is set, this writer is never called and the playlist remains "live" from the player's perspective.

#### DASH coupling

DASH coupling is established in the build system at `[libavformat/Makefile:L189]`:

```text
OBJS-$(CONFIG_DASH_MUXER) += dash.o dashenc.o hlsplaylist.o
```

`hlsplaylist.o` is linked into the DASH muxer's object set so when DASH is built with HLS-side-channel support enabled, it can call these writers directly. The HLS muxer's own linkage at `[libavformat/Makefile:L277]` is `OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`. Both muxers therefore depend on the same compiled artifact.

The DASH muxer calls into these writers when its `hls_playlist` option is set: it publishes a primary DASH `.mpd` manifest and an alternate HLS `.m3u8` master playlist plus per-variant media playlists, allowing a client to choose either protocol. Because both muxers share the writers, the M3U8 published by DASH is byte-identical (for the same source streams) to the M3U8 that the HLS muxer would have produced. This is by design and is the reason the writers were factored out in the first place.

#### Wire-format consequences

- **Versioning:** A change to `ff_hls_write_playlist_version` that bumps the default `EXT-X-VERSION` number affects both formats' minimum client compatibility level.
- **New tags:** A new tag added to `ff_hls_write_playlist_header` would appear in both HLS-muxer playlists and DASH-side-channel playlists immediately.
- **Attribute reordering:** Most clients tolerate reordering of attributes within a single tag line, but some older or strict implementations may not — any reordering inside the writers must be tested against both muxers' downstream consumers.

#### PlaylistType enumeration

The `PlaylistType` enumeration used by `ff_hls_write_playlist_header` is defined in the writer's own header at `[libavformat/hlsplaylist.h:L31-L36]`. It carries three values: `PLAYLIST_TYPE_NONE`, `PLAYLIST_TYPE_EVENT`, `PLAYLIST_TYPE_VOD` (plus the sentinel `PLAYLIST_TYPE_NB`). It is distinct from the demuxer-side playlist type at `[libavformat/hls.c:L91-L95]` despite carrying similar semantics — the demuxer and muxer use separate type spaces because the demuxer also defines a `PLS_TYPE_UNSPECIFIED` value to handle the case of a live playlist with no `#EXT-X-PLAYLIST-TYPE` tag, which has no equivalent on the muxer side (the muxer always knows whether it is publishing VOD, EVENT, or live).

#### Cross-references

- The shared status of this translation unit has integrator-facing implications, enumerated in [`./consumer-dependencies.md`](./consumer-dependencies.md) §6.
- The complete I/O catalog for each writer (input parameters, output bytes) lives in [`./inputs-outputs.md`](./inputs-outputs.md).
- The functional invariants each writer must preserve (e.g., `#EXTM3U` always on line 1) live in [`../api-contracts/functional-invariants.md`](../api-contracts/functional-invariants.md).
- The data-contract reference for each tag's attribute formats lives in [`../api-contracts/data-contracts.md`](../api-contracts/data-contracts.md).

## Lifecycle Roll-Up

The muxer's externally visible lifecycle is the standard FFmpeg muxer pattern: an init phase that validates options and allocates per-variant state, a write-header phase that opens the first segment per variant, a write-packet loop that handles one packet at a time and cuts segments at keyframe boundaries, a write-trailer phase that flushes any pending segment and publishes the final playlist (with `#EXT-X-ENDLIST` if applicable), and a deinit phase that frees all state. The five phases are bound into FFmpeg's muxer-dispatch table by the `FFOutputFormat ff_hls_muxer` registration record at the bottom of the muxer source file. Engineering detail on each phase and on the internal callback chain lives in [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md); this section is the quick locator map.

| Phase | Function | Source |
|---|---|---|
| Init | `hls_init` | `[libavformat/hlsenc.c:L2866]` |
| Write Header | `hls_write_header` | `[libavformat/hlsenc.c:L2301]` |
| Write Packet | `hls_write_packet` | `[libavformat/hlsenc.c:L2410]` |
| Write Trailer | `hls_write_trailer` | `[libavformat/hlsenc.c:L2727]` |
| Deinit | `hls_deinit` | `[libavformat/hlsenc.c:L2693]` |
| FFOutputFormat registration | `ff_hls_muxer` | `[libavformat/hlsenc.c:L3191-L3207]` |

### Muxer lifecycle in plain language

For a reader new to FFmpeg's muxer pattern, the muxer lifecycle is invoked by the application's calls to `avformat_write_header` / `av_write_frame` (or `av_interleaved_write_frame`) / `av_write_trailer` on the parent `AVFormatContext`. Each application call dispatches through the FFmpeg generic muxer machinery, which in turn invokes the format-specific callbacks on the muxer's registration record. The pattern is:

1. **Init** — runs once at the very start (before `avformat_write_header` even). Allocates and validates state.
2. **Write Header** — runs once after init. Opens the first segment file and emits any pre-segment metadata (e.g., the fMP4 init segment).
3. **Write Packet** — runs once per `AVPacket` delivered by the application. Each call may close the current segment and open a new one when the segment-cut criteria are met.
4. **Write Trailer** — runs once at the end. Finalizes the last segment, publishes the playlist with end-of-list marker (if VOD), and ensures everything is flushed.
5. **Deinit** — runs once after the trailer. Frees all allocated state.

The application is responsible for delivering packets in the correct order (typically using `av_interleaved_write_frame` to handle interleaving across multiple streams). The muxer is *stateful* in that it tracks segment boundaries, variant routing, and playlist state across multiple write-packet calls; the application is *stateless* relative to the muxer (it does not need to know which segment a packet will end up in).

The demuxer side has its own five-phase lifecycle bound by the `FFInputFormat ff_hls_demuxer` registration record:

| Phase | Function | Source |
|---|---|---|
| Probe | `hls_probe` | `[libavformat/hls.c:L2814]` |
| Read Header | `hls_read_header` | `[libavformat/hls.c:L2144]` |
| Read Packet | `hls_read_packet` | `[libavformat/hls.c:L2546]` |
| Read Seek | `hls_read_seek` | `[libavformat/hls.c:L2709]` |
| Read Close | `hls_close` | `[libavformat/hls.c:L2127]` |
| FFInputFormat registration | `ff_hls_demuxer` | `[libavformat/hls.c:L2900-L2912]` |

### Demuxer lifecycle in plain language

For the demuxer, the application's calls go through `avformat_open_input` / `av_read_frame` / `av_seek_frame` / `avformat_close_input`. The pattern is:

1. **Probe** — runs implicitly inside `avformat_open_input` when the application has not specified a format. Returns a confidence score indicating "this is HLS" or "this isn't HLS".
2. **Read Header** — runs once after probe success. Fetches and parses the playlist, opens sub-demuxers for each variant, registers each elementary stream with the application via `avformat_new_stream`.
3. **Read Packet** — runs once per `av_read_frame` call. Picks the next playlist's next packet (re-fetching the next segment if needed) and returns it. May trigger a live-mode playlist refresh.
4. **Read Seek** — runs once per `av_seek_frame` call. Maps the target timestamp to a segment index and restarts the affected sub-demuxers.
5. **Read Close** — runs once at the end. Frees the parsed playlist data, closes sub-demuxers, releases network state.

Note: Probe is not strictly a separate lifecycle phase from the application's perspective — it's part of `avformat_open_input`. But it is a separate callback on the registration record and runs in a distinct execution context (no allocated demuxer state yet), so it is documented as a phase here.

### Phase interaction matrix

The following table summarizes which functions in the muxer interact with which other functions, to help readers trace a particular flow end-to-end:

| Calling function | Calls (in order) |
|---|---|
| `hls_init` | option validation, `parse_variant_stream_mapstring`, `parse_cc_stream_mapstring`, `var_streams` allocation |
| `hls_write_header` | per-variant `hls_mux_init` → child `avformat_write_header`, optional `hls_encryption_start`, `hls_start` for first segment |
| `hls_write_packet` | routing, segment-cut decision, `av_write_frame(child, pkt)`, on cut: `av_write_frame(child, NULL)` → segment finalize → `hls_append_segment` → `hls_window` → `hls_start` |
| `hls_write_trailer` | per-variant final `av_write_frame(child, NULL)` flush → `hls_append_segment` (last segment) → `hls_window(last=1)` → master playlist final publish |
| `hls_deinit` | per-variant `avformat_free_context(child)`, free linked lists, free strings |

The complete control-flow diagrams (Mermaid `flowchart TB` for the muxer lifecycle DAG and a `classDiagram` for the meta-muxer parent/child relationship) live in [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md). Lifecycle coverage is duplicated here for navigational convenience; the authoritative narrative for each phase, the internal callback chain, and the segment-finalization sequencing live in [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md).

## Cross-Component Reference Map

Each component documented above is cross-referenced from multiple Layer 2 and Layer 3 documents in this set. The table below summarizes, per component, where the deeper coverage lives. This is the canonical mapping; integrators and engineers reading this inventory should use it to plan their reading order through the full documentation set.

| Component (§) | Layer 2 documents | Layer 3 documents |
|---|---|---|
| Segment Generation (§3) | `process-flows.md` (segment-generation flowchart), `codec-logic.md` (segment-cut decision table), `pipeline-orchestration.md` (lifecycle), `data-model.md` (HLSSegment dictionary) | `timing-dependencies.md` (keyframe-detect-then-cut sequence), `integration-contracts.md` (filename template format) |
| Playlist Construction (§4) | `process-flows.md` (playlist-update flowchart), `codec-logic.md` (version negotiation, conditional emissions), `pipeline-orchestration.md` (`hls_window` call sites) | `functional-invariants.md` (M3U8 line-1/2 order, target-duration ≥ max segment), `data-contracts.md` (timestamp unit conventions), `timing-dependencies.md` (target duration computed before first segment), `integration-contracts.md` (per-tag wire format) |
| Encryption Handling (§5) | `process-flows.md` (key rotation flow), `codec-logic.md` (key reload, IV derivation), `integration-interfaces.md` (AES pipeline) | `functional-invariants.md` (EXT-X-KEY precedes encrypted segment), `data-contracts.md` (KEYSIZE 16, IV format), `integration-contracts.md` (AES-128 key URI fetch) |
| Variant Stream Management (§6) | `codec-logic.md` (var_stream_map parsing matrix), `data-model.md` (VariantStream 75-field dictionary), `pipeline-orchestration.md` (per-variant init) | `functional-invariants.md` (EXT-X-STREAM-INF placement), `data-contracts.md` (BANDWIDTH integer format), `integration-contracts.md` (EXT-X-MEDIA + EXT-X-STREAM-INF wire format) |
| Closed Captions Track (§7) | `codec-logic.md` (cc_stream_map parsing), `data-model.md` (ClosedCaptionsStream struct) | `integration-contracts.md` (EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS wire format) |
| Live vs VOD Mode Selection (§8) | `codec-logic.md` (hls_playlist_type × flag interaction matrix), `process-flows.md` (sliding-window flowchart) | `functional-invariants.md` (EXT-X-ENDLIST emission rules) |
| Discontinuity Handling (§9) | `codec-logic.md` (discontinuity placement decision table), `process-flows.md` (append-list resume flow) | `functional-invariants.md` (discontinuity line placement), `timing-dependencies.md` (discontinuity-precedes-segment-entry constraint) |
| HLS Demuxer Probe (§10) | `pipeline-orchestration.md` (probe phase) | `integration-contracts.md` (M3U8 magic + HLS-tag signature) |
| HLS Demuxer Playlist Parser (§11) | `process-flows.md` (parse sequence), `pipeline-orchestration.md` (read-header / read-packet), `data-model.md` (demuxer playlist/variant/rendition structs), `integration-interfaces.md` (HTTP fetch) | `data-contracts.md` (per-tag attribute formats accepted), `timing-dependencies.md` (parse-before-emit ordering) |
| HLS Sample Encryption (§12) | `data-model.md` (HLSCryptoContext, HLSAudioSetupInfo), `integration-interfaces.md` (Sample-AES decrypt path) | `data-contracts.md` (STREAM_TYPE_HLS_SE_* values), `integration-contracts.md` (Sample-AES transport) |
| Playlist Tag Writers (§13) | `data-model.md` (PlaylistType enum) | `functional-invariants.md` (every per-tag invariant), `data-contracts.md` (every tag wire format), `integration-contracts.md` (downstream-consumer contracts) |

### Common reading paths

The following reading paths combine the inventory above with the cross-references to give complete coverage by audience:

**Path A — Integrator who needs to publish HLS:**

1. README.md (set-wide overview)
2. This document — §§3, 4, 5, 6, 8 (the core publishing components)
3. `inputs-outputs.md` (AVOption table)
4. `consumer-dependencies.md` (downstream consumers)
5. `exception-handling.md` (failure modes)

**Path B — Integrator who needs to consume HLS:**

1. README.md
2. This document — §§10, 11, 12 (demuxer components)
3. `inputs-outputs.md`
4. `exception-handling.md`

**Path C — Engineer planning a refactor or port:**

1. README.md
2. This document — all 11 components in order
3. All Layer 2 documents under `../technical/` (process-flows, codec-logic, data-model, pipeline-orchestration, integration-interfaces)
4. All Layer 3 documents under `../api-contracts/` (functional-invariants, data-contracts, timing-dependencies, integration-contracts)

**Path D — Auditor reviewing for spec compliance or security:**

1. README.md
2. This document — Overview and Plain-Language tiers of each component
3. `../api-contracts/functional-invariants.md` (the zero-deviation checklist)
4. `../api-contracts/integration-contracts.md` (external-system contracts)
5. `exception-handling.md` (error paths)

Each path lands the reader in the same authoritative material — only the entry sequence differs. Readers may freely jump between documents using the inline citations.

## Component Dependency Summary

The 11 components above are not independent. Their dependencies (which component logically requires which other component to be in place first) are summarized below. This is useful for refactor scoping: a refactor of a *root* component cascades to all its dependents, while a refactor of a *leaf* component is locally contained.

| Component | Depends on | Notes |
|---|---|---|
| Segment Generation (§3) | Variant Stream Management (§6) for routing | Each packet's variant must be known before segment generation can apply |
| Playlist Construction (§4) | Segment Generation (§3), Playlist Tag Writers (§13) | Playlist is generated from the segment list using the writer functions |
| Encryption Handling (§5) | Segment Generation (§3) | Encryption transforms segment bytes; runs inside segment-write path |
| Variant Stream Management (§6) | (root — no dependencies) | First component initialized; everything else builds on per-variant state |
| Closed Captions Track (§7) | Variant Stream Management (§6) | Closed-captions advertisements are bound to variants |
| Live vs VOD Mode Selection (§8) | Segment Generation (§3), Playlist Construction (§4) | Selection affects when segments are deleted and when end-list is emitted |
| Discontinuity Handling (§9) | Segment Generation (§3), Playlist Construction (§4) | Discontinuity flag attaches to segments and propagates to playlist |
| HLS Demuxer Probe (§10) | (root — independent of demuxer state) | Runs before any demuxer state is allocated |
| HLS Demuxer Playlist Parser (§11) | HLS Demuxer Probe (§10) | Parser runs after probe selects the demuxer |
| HLS Sample Encryption (§12) | HLS Demuxer Playlist Parser (§11) | Sample-AES is signaled in the playlist; decryption runs per packet from the parser's segment list |
| Playlist Tag Writers (§13) | (root — pure helper functions) | Stateless helpers called by Playlist Construction; also called by DASH muxer |

A refactor that changes the wire format of any tag emitted by Playlist Tag Writers (§13) is the highest-risk change in the muxer pipeline because both the HLS muxer and the DASH muxer publish through the same writers. A refactor of Segment Generation (§3) is the next highest-risk because it touches every packet, with cascading effects through Playlist Construction, Encryption Handling, Live vs VOD Mode Selection, and Discontinuity Handling.

This dependency summary is not a complete formal dependency graph — for that, see [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md), which contains the full Mermaid `flowchart TB` lifecycle DAG with per-component callout boxes.

## Document Navigation Footer

This document is the canonical *inventory* of HLS pipeline components. After reading it, the recommended next documents are:

- [`./inputs-outputs.md`](./inputs-outputs.md) — table format I/O capture for every component (every AVOption, every EXT-X-* tag, every AVPacket field)
- [`./consumer-dependencies.md`](./consumer-dependencies.md) — downstream consumers (players, CDN, DASH muxer reuse)
- [`./exception-handling.md`](./exception-handling.md) — failure scenarios with AVERROR code mappings
- [`../technical/`](../technical/) — Layer 2 engineer-facing technical reference (process flows, decision tables, data dictionary, lifecycle orchestration, integration interfaces)
- [`../api-contracts/`](../api-contracts/) — Layer 3 zero-deviation contracts (functional invariants, data contracts, timing dependencies, integration contracts)

Set-wide conventions (commit anchor, citation format, reading orders) are defined in [`../README.md`](../README.md).

