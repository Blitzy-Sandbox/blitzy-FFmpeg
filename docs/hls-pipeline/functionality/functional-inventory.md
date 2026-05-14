# Functional Inventory — Components of the FFmpeg HLS Pipeline

This document is the canonical component inventory referenced by every Layer 2 (technical) and Layer 3 (api-contracts) document in this set. It enumerates the functional components of the FFmpeg HLS muxer and demuxer, with each component described first in plain language for library integrators and then in engineer-facing technical detail with full source citations.

All source references in this document are anchored to commit `566ad786`. See `../README.md` for the canonical commit-anchor banner and the citation-format specification.

## Overview

The FFmpeg HLS pipeline is two related subsystems that share several supporting modules. The **muxer** turns an application's stream of compressed audio, video, and subtitle frames into an HLS publication: a top-level playlist manifest file plus a sequence of fixed-length media segment files on disk or pushed to a remote server. The **demuxer** does the opposite — given a playlist URL, it fetches the manifest, fetches each referenced segment, and turns the result back into a stream of compressed frames that the application can decode.

Internally each side is factored into several focused components. On the muxer side these are: segment generation (chopping the input stream at keyframe boundaries into segment files), playlist construction (rewriting the manifest after each segment), encryption handling (optional AES-128 segment encryption with periodic key rotation), variant stream management (multiple bitrate ladders for adaptive playback), closed-captions tracks (embedded 608/708 caption advertisements), live versus VOD mode selection (sliding window versus complete recording), and discontinuity handling (timestamp/codec changes mid-stream). On the demuxer side: a probe that recognizes M3U8 magic bytes, a playlist parser that builds the in-memory representation of variants and segments, and a sample-encryption transform for the Apple Sample-AES variant.

Both sides share a small translation unit of playlist-tag writer helpers that emit the wire-format `#EXT-X-*` lines. That translation unit is also linked into the FFmpeg DASH muxer so both formats produce byte-identical HLS playlist output where their feature sets overlap. The components covered in this document are summarized in the table below.

| Component | Side | Section |
|---|---|---|
| Segment Generation | Muxer | §3 |
| Playlist Construction | Muxer | §4 |
| Encryption Handling | Muxer | §5 |
| Variant Stream Management | Muxer | §6 |
| Closed Captions Track | Muxer | §7 |
| Live vs VOD Mode Selection | Muxer | §8 |
| Discontinuity Handling | Muxer | §9 |
| HLS Demuxer Probe | Demuxer | §10 |
| HLS Demuxer Playlist Parser | Demuxer | §11 |
| HLS Sample Encryption | Demuxer | §12 |
| Playlist Tag Writers (shared) | Both | §13 |

A lifecycle roll-up table summarizing the muxer's standard FFmpeg lifecycle phases appears in §14.

## Component: Segment Generation

Segment generation is the muxer's primary job. The HLS publication model presents a continuous broadcast or recording as a series of short media files — typically two to ten seconds each — so a client player can request any one of them at any time and begin playback at that point. To make every segment a valid starting point, the HLS specification requires every segment to begin with a video keyframe. The muxer therefore watches every incoming frame, and when it sees a video keyframe that arrives after the configured target duration has elapsed since the previous segment boundary, it closes the segment file currently being written, opens a new one, and the next frames start filling that new file.

Once a segment file is closed, the muxer also records the segment's filename, duration, byte offset and size, optional encryption metadata, and discontinuity flag into a per-variant linked list. That linked list is the in-memory representation of the playlist manifest the muxer republishes after each new segment. Streams that never carry video, or streams where the application has set the flag asking the muxer to cut purely on elapsed time without waiting for keyframes, are also supported.

### Technical detail

- The packet pipeline entry point is `hls_write_packet` at `[libavformat/hlsenc.c:L2410]`. Each `AVPacket` delivered to the muxer through `av_write_frame` or `av_interleaved_write_frame` is dispatched into this callback for the variant the packet belongs to.
- Whether the current packet is a candidate for cutting a new segment is decided by the local `can_split` flag at `[libavformat/hlsenc.c:L2473-L2475]`: the packet must either be a video keyframe (`pkt->flags & AV_PKT_FLAG_KEY` on a video stream) or the muxer must have the `HLS_SPLIT_BY_TIME` flag set (allowing cuts on elapsed time even between keyframes).
- When `pkt->pts == AV_NOPTS_VALUE`, `can_split` is forced to zero at `[libavformat/hlsenc.c:L2478-L2479]`. A stream that never carries presentation timestamps will never trigger a time-based cut.
- The cut-or-not comparison is performed at `[libavformat/hlsenc.c:L2501]` using `av_compare_ts` against the accumulated `vs->end_pts` plus the configured target segment duration (`hls->time`, default 2,000,000 µs = 2 s, defined as the `hls_time` AVOption at `[libavformat/hlsenc.c:L3123]`).
- A strictly positive PTS delta is also required at `[libavformat/hlsenc.c:L2500]` (`pkt->pts - vs->end_pts > 0`) so duplicate keyframe timestamps do not produce zero-duration segments.
- Per-variant segment open is `hls_start` at `[libavformat/hlsenc.c:L1675]`. It opens the next segment file (via `hlsenc_io_open` at `[libavformat/hlsenc.c:L292]`, which transparently handles `file://`, `http://`, or any other registered protocol), resets the per-segment counters (`packets_written`, `start_pos`, `size`, `start_pts`, `end_pts`), and prepares the sub-muxer for the next segment.
- After the cut decision fires, segment finalization appends a record of the just-closed segment to the variant's linked list via `hls_append_segment` at `[libavformat/hlsenc.c:L1042]`. Records use the `HLSSegment` struct defined at `[libavformat/hlsenc.c:L76-L94]`: `filename`, `sub_filename`, `duration`, `discont`, `pos`, `size`, `keyframe_pos`, `keyframe_size`, `var_stream_idx`, `key_uri`, `iv_string`, `next` (singly-linked-list pointer), `discont_program_date_time`, plus a flexible-array `buf[]` tail.
- The actual bitstream format written into each segment file is owned by a child muxer — either MPEG-TS or fragmented MP4 — that the HLS muxer instantiates per variant. The HLS muxer is therefore a *meta-muxer*: it does not emit TS packets or MP4 boxes itself. The child format context is `vs->avf` at `[libavformat/hlsenc.c:L133]` (an `AVFormatContext` pointer), and the format is chosen by the `hls_segment_type` AVOption at `[libavformat/hlsenc.c:L3130-L3132]` (values `SEGMENT_TYPE_MPEGTS = 0` or `SEGMENT_TYPE_FMP4 = 1`, defined at `[libavformat/hlsenc.c:L115-L118]`).
- For TS segmenting the entire segment file is independent (no shared header); for fragmented MP4 segmenting an initialization segment (default filename `"init.mp4"`, configurable via `hls_fmp4_init_filename` at `[libavformat/hlsenc.c:L3142]`) is written once and referenced by every media segment through the `#EXT-X-MAP` tag in the playlist.

## Component: Playlist Construction

After every new segment is closed, the muxer rewrites the playlist file so a client polling for updates discovers the new segment on its next refresh. The playlist is a small text manifest in the M3U8 format defined by the HLS specification: a fixed opening sequence of metadata tags identifying the playlist version, allowed cache behavior, target segment length, and starting sequence number, followed by one entry per segment listing the segment's duration and URL.

Playlist construction is split into a fixed-order pipeline of small writer helpers, each emitting one specific category of tag. The writer helpers live in a separate translation unit so the FFmpeg DASH muxer can share them — see §13 for that coupling. The muxer's own role is to drive the writers in the right order, compute aggregate fields like the target duration that must be set before any segment is listed, and route each writer's output to the correct destination (a file on disk, an HTTP `PUT` to a remote ingest endpoint, or a temporary file followed by an atomic rename).

### Technical detail

- Per-variant playlist publish is the function `hls_window` at `[libavformat/hlsenc.c:L1531]`. It is called from `hls_write_packet` after each segment is appended (and once from `hls_write_trailer` at `[libavformat/hlsenc.c:L2727]` for the final flush).
- The fixed-order top-of-playlist sequence is:
  1. `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` — emits `#EXTM3U` on line 1 and `#EXT-X-VERSION:%d` on line 2.
  2. `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]` — emits (conditionally) `#EXT-X-ALLOW-CACHE`, `#EXT-X-TARGETDURATION:%d`, `#EXT-X-MEDIA-SEQUENCE:%lld`, `#EXT-X-PLAYLIST-TYPE:EVENT|VOD`, and `#EXT-X-I-FRAMES-ONLY`.
  3. Optional `#EXT-X-DISCONTINUITY` at top of segment list when `HLS_DISCONT_START` is set, emitted at `[libavformat/hlsenc.c:L1593-L1596]`.
  4. Optional `#EXT-X-INDEPENDENT-SEGMENTS` when the variant carries video and the `HLS_INDEPENDENT_SEGMENTS` flag is set, emitted at `[libavformat/hlsenc.c:L1597-L1599]`.
  5. Per-segment `#EXT-X-KEY:METHOD=AES-128,URI="...",IV=0x...` line when the key has changed since the previous segment, emitted at `[libavformat/hlsenc.c:L1601-L1609]`.
  6. Optional `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]` — emits `#EXT-X-MAP:URI="...",BYTERANGE="..."` exactly once at the head of the segment list when `segment_type=fmp4`.
  7. Per-segment `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L144-L199]` — emits `#EXT-X-DISCONTINUITY` (when `seg->discont` is set), `#EXTINF:%f,`, optional `#EXT-X-BYTERANGE:%lld@%lld`, optional `#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s`, and the segment URL itself.
  8. Optional `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]` — emits `#EXT-X-ENDLIST` on the trailer publish when `HLS_OMIT_ENDLIST` is not set, emission gated at `[libavformat/hlsenc.c:L1629-L1630]`.
- The integer value emitted in the `#EXT-X-VERSION` line is negotiated by `hls_window` based on which features the muxer is using on this publish; the result is the maximum of all version-floor rules required by enabled features. The negotiation matrix lives in `../technical/codec-logic.md`; this document treats the version as a derived quantity.
- The `#EXT-X-TARGETDURATION` field is computed at `[libavformat/hlsenc.c:L1584-L1587]` as the `lrint` of the maximum segment duration in the current segment list. The HLS specification requires this field to be greater than or equal to every segment's duration, so the muxer is conservative and emits the maximum.
- When the `HLS_TEMP_FILE` flag is set, each playlist publish writes to `<m3u8_name>.tmp` first and renames atomically via `hls_rename_temp_file` at `[libavformat/hlsenc.c:L1300]`. The atomic rename protects clients that may be polling the playlist while it is being rewritten.
- When the variant set produces a *master playlist* (the top-level manifest enumerating all variants), the master playlist is published separately from the per-variant playlists. Master-playlist publishing cadence is controlled by `master_pl_publish_rate` at `[libavformat/hlsenc.c:L3175]`; the master is rewritten only when the variant sequence number is divisible by that rate, so a player polling the master sees an update at a configurable cadence independent of segment cadence. The master playlist is written by `hls_write_packet` itself by interleaving calls to `ff_hls_write_audio_rendition`, `ff_hls_write_subtitle_rendition`, and `ff_hls_write_stream_info` — see §6 for those helpers.

## Component: Encryption Handling

The muxer can optionally encrypt every segment file using AES-128 in cipher-block-chaining mode. When encryption is enabled, three pieces of information are needed: the 16-byte encryption key itself (a secret known to authorized clients), a 16-byte initialization vector that perturbs the cipher's starting state, and the URL where authorized clients fetch the key over a separate channel (typically an authenticated HTTPS endpoint). The playlist advertises the key URL but not the key itself, so an eavesdropper who captures the playlist and segment files cannot decrypt them without first authenticating to the key endpoint.

The three values are supplied to the muxer through a small text file pointed at by the `hls_key_info_file` option (three lines: the URL clients will fetch, the local path to the raw key bytes the muxer should use, and an optional hex-encoded IV). The muxer also supports periodic key rotation — long-running streams can change their encryption key without restarting by replacing the key info file's contents at any time; the muxer re-reads the file before each segment and rotates seamlessly.

### Technical detail

- The encryption initialization function is `hls_encryption_start` at `[libavformat/hlsenc.c:L714]`. It parses the key info file (path from the `hls_key_info_file` AVOption at `[libavformat/hlsenc.c:L3136]`), reads the URL from line 1 into `vs->key_uri`, reads the 16 raw binary key bytes from the file path on line 2 into `vs->key_string`, and optionally parses a 32-hex-character IV from line 3 into `vs->iv_string`.
- The key and IV size is constant: `#define KEYSIZE 16` at `[libavformat/hlsenc.c:L70]`. Both the AES key and the IV are exactly 16 bytes.
- Per-segment AES-128 CBC encryption is performed by the `do_encrypt` helper at `[libavformat/hlsenc.c:L641]`. It uses the FFmpeg AES library API: `av_aes_alloc` at `[libavutil/aes.h:L41]`, `av_aes_init` at `[libavutil/aes.h:L51]`, and `av_aes_crypt` at `[libavutil/aes.h:L63]`.
- The `#EXT-X-KEY` playlist tag is emitted inside `hls_window` at `[libavformat/hlsenc.c:L1601-L1609]` immediately before the first segment it protects. The tag format is `#EXT-X-KEY:METHOD=AES-128,URI="<url>"` with an optional `,IV=0x<32-hex-chars>` suffix when the IV is explicitly carried in the playlist rather than derived from the segment sequence number.
- Per-segment encryption metadata is carried on each `HLSSegment` record (struct at `[libavformat/hlsenc.c:L76-L94]`): the `key_uri` and `iv_string` fields are populated when the segment is appended so the playlist can emit a new `#EXT-X-KEY` line whenever the URI or IV changes between segments.
- When the `HLS_PERIODIC_REKEY` flag is set (AVOption const `periodic_rekey` at `[libavformat/hlsenc.c:L3157]`, decimal value `1 << 11`), the muxer re-reads the key info file before every segment. If the file's contents have changed since the previous segment, the new key and IV take effect immediately and a new `#EXT-X-KEY` line is emitted in the playlist before the next segment entry.
- A simpler encryption mode is also available: the `hls_enc` AVOption at `[libavformat/hlsenc.c:L3134]` enables built-in key generation when no key info file is supplied. The muxer derives a random 16-byte key, derives the IV from the segment sequence number, and writes both into a `.key` file alongside the segments — used primarily for self-contained demo or development setups.
- See `../technical/integration-interfaces.md` for the key URI fetch protocol and `../api-contracts/integration-contracts.md` for the AES-128 wire-format contracts.

## Component: Variant Stream Management

Adaptive-bitrate playback is one of the defining features of HLS: a single source can be encoded at several quality tiers (for example 4K, 1080p, 720p, 480p) and the client player automatically picks the tier that best matches the available bandwidth, switching tiers seamlessly on segment boundaries as bandwidth conditions change. The muxer supports this model by accepting a *variant stream map* — a small text string mapping each input stream into one of several variants. Each variant gets its own playlist file and its own series of segment files; a top-level *master playlist* lists all variants and lets clients discover them.

Variants can also be paired with separate audio-only or subtitle-only playlists (called *renditions*) using HLS group identifiers. A typical configuration produces one video-bearing variant playlist per quality tier, one audio-only rendition playlist per language, and one subtitle rendition playlist per language; the master playlist's `#EXT-X-STREAM-INF` entries reference the audio and subtitle groups so clients fetch the right audio and subtitle tracks alongside whichever video tier they are currently consuming.

### Technical detail

- The per-variant state is the `VariantStream` struct at `[libavformat/hlsenc.c:L120-L194]` (75 fields). Key fields include: `sequence` (current segment sequence number), `oformat` and `avf` (the child format and its `AVFormatContext` used as a sub-muxer to emit the actual TS or fMP4 bytes), `vtt_oformat` and `vtt_avf` (a parallel sub-muxer for WebVTT subtitle segments), `packets_written`, `init_range_length`, accumulators `total_size`/`total_duration`/`avg_bitrate`/`max_bitrate`, linked-list heads `segments`/`last_segment`/`old_segments`, file-name templates `basename`/`m3u8_name`/`vtt_basename`/`vtt_m3u8_name`/`fmp4_init_filename`/`base_output_dirname`, and group-membership strings `agroup`, `sgroup`, `ccgroup`, `varname`, `subtitle_varname`.
- The per-context arrays are on the parent `HLSContext` (struct at `[libavformat/hlsenc.c:L202-L267]`): `var_streams` (pointer to the array), `nb_varstreams` (count).
- The user-supplied variant map is parsed by `parse_variant_stream_mapstring` at `[libavformat/hlsenc.c:L1998]`. The string is consumed from the `var_stream_map` AVOption at `[libavformat/hlsenc.c:L3172]`. Syntax follows the form `v:0,a:0 v:1,a:1` where each space-separated group becomes one variant and each colon-tagged token (`v:`, `a:`, `s:`) selects an input stream by index.
- When the variant set has more than one variant or when the `master_pl_name` AVOption at `[libavformat/hlsenc.c:L3174]` is set, the muxer also publishes a master playlist. The master playlist is built by `hls_write_packet`'s master-playlist branch which calls into the playlist tag writers:
  - `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` for the header.
  - Per closed-captions group: `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,...` lines emitted directly by the muxer at `[libavformat/hlsenc.c:L1398-L1406]`.
  - Per audio-only rendition: `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L40-L56]`.
  - Per subtitle rendition: `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]`.
  - Per variant: `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]` — emits `#EXT-X-STREAM-INF:BANDWIDTH=%d,AVERAGE-BANDWIDTH=%d,RESOLUTION=WxH,CODECS="...",AUDIO="group_%s",CLOSED-CAPTIONS="%s",SUBTITLES="%s"` followed by the variant's playlist URL.
- Master-playlist republishing cadence is controlled by `master_pl_publish_rate` at `[libavformat/hlsenc.c:L3175]`. The muxer rewrites the master only when the current variant sequence number is divisible by this rate, which lets a publisher refresh the master at a coarser cadence than per-variant playlists (the master rarely changes once all variants are known, so frequent rewrites would be wasteful).
- Each variant's bandwidth accounting (the `total_size`, `total_duration`, `avg_bitrate`, and `max_bitrate` fields at `[libavformat/hlsenc.c:L154-L157]`) is updated after each segment to reflect actual bytes-per-second observed. The master-playlist `BANDWIDTH` and `AVERAGE-BANDWIDTH` attributes use these values so the advertised values reflect real consumption, not nominal target bitrates.


## Component: Closed Captions Track

Closed captions in HLS are the embedded EIA-608 or CTA-708 caption streams carried inside a video stream's bitstream rather than transmitted as separate text files. They are not their own playlist; they are advertised in the master playlist using a `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` line that names a group, language, and instream identifier (such as `CC1`, `CC2`, `CC3`, `CC4`, or `SERVICE1` through `SERVICE63`). A video variant then references the same group identifier through the `CLOSED-CAPTIONS=` attribute of its `#EXT-X-STREAM-INF` line; a player consuming that variant knows which caption channel to extract from the video bitstream and which language label to display.

The muxer accepts a small per-captions-stream configuration mapping the group name, language, and instream identifier through the `cc_stream_map` option. Each entry produces one `#EXT-X-MEDIA` line in the master playlist. Individual video variants are wired to a particular caption group through their variant-stream-map entry, which causes the muxer to add a `CLOSED-CAPTIONS="<group>"` attribute to that variant's `#EXT-X-STREAM-INF` line so the client knows which caption channel and language label belong to which video tier.

### Technical detail

- The `ClosedCaptionsStream` struct at `[libavformat/hlsenc.c:L196-L200]` carries three fields per closed-captions stream: `ccgroup`, `instreamid`, `language`.
- Per-context arrays on `HLSContext`: `cc_streams` and `nb_ccstreams` at `[libavformat/hlsenc.c:L248-L249]`.
- The user-supplied configuration string is the `cc_stream_map` AVOption at `[libavformat/hlsenc.c:L3173]`. It is parsed by `parse_cc_stream_mapstring` at `[libavformat/hlsenc.c:L2144]`. Syntax is space-separated entries of the form `ccgroup:<group>,instreamid:<id>,language:<lang>`.
- The `#EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS` emission loop runs inside the master-playlist writer block at `[libavformat/hlsenc.c:L1398-L1406]`. For each entry in `cc_streams` the muxer emits one line:

  ```text
  #EXT-X-MEDIA:TYPE=CLOSED-CAPTIONS,GROUP-ID="<ccgroup>",NAME="<instreamid>",LANGUAGE="<language>",INSTREAM-ID="<instreamid>"
  ```

  The `INSTREAM-ID` value is repeated as the `NAME` attribute by default; the literal printf at `[libavformat/hlsenc.c:L1400]` is the emission point.
- Each variant carries a `ccgroup` string field on its `VariantStream` record (defined at `[libavformat/hlsenc.c:L120-L194]`); the value is set during variant-stream-map parsing when an entry contains a `ccgroup:` token. Variant `#EXT-X-STREAM-INF` emission at `[libavformat/hlsenc.c:L1484-L1496]` resolves `vs->ccgroup` and passes it into `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]`, which appends the `CLOSED-CAPTIONS="<ccgroup>"` attribute to the variant's stream-info line.
- This component covers only the *advertisement* of closed captions in the master playlist. The actual caption bytes are not extracted, repackaged, or rewritten by the HLS muxer; they remain embedded inside the video stream as the upstream encoder placed them. The muxer's responsibility ends at the playlist advertisement, after which the client player extracts captions from the decoded video using its own EIA-608/CTA-708 parser.

## Component: Live vs VOD Mode Selection

An HLS playlist comes in three flavors that correspond to three publishing models. A *live* playlist describes a sliding window of recent segments and grows on one end while old segments fall off the other; a client polls the playlist periodically and follows the moving window. A *video-on-demand* (VOD) playlist describes a complete, finished recording from beginning to end and is closed with an explicit end-of-list tag; once published it never changes. An *event* playlist is append-only — it grows over time as new segments are added but never deletes any segments, allowing late-joining clients to seek back to the beginning of the event.

The muxer selects the flavor through the `hls_playlist_type` option and through several flag interactions. Live mode is the default when the option is left unset — the playlist contains only the most recent few segments and older segments are deleted from the head as the configured sliding-window length is exceeded. Setting the option to `vod` or `event` selects the other two flavors, and three additional flags (`omit_endlist`, `append_list`, and `delete_segments`) interact with the choice in non-obvious ways.

### Technical detail

- The playlist-type enumeration is defined at `[libavformat/hlsplaylist.h:L31-L36]`: `PLAYLIST_TYPE_NONE = 0`, `PLAYLIST_TYPE_EVENT = 1`, `PLAYLIST_TYPE_VOD = 2`, `PLAYLIST_TYPE_NB = 3` (the count sentinel).
- The AVOption that selects the type is `hls_playlist_type` at `[libavformat/hlsenc.c:L3162]` (default `PLAYLIST_TYPE_NONE`); user-facing aliases `event` and `vod` are bound at `[libavformat/hlsenc.c:L3163-L3164]`.
- When the type is `EVENT` or `VOD`, the playlist-header writer (`ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]`) emits an `#EXT-X-PLAYLIST-TYPE:EVENT` or `#EXT-X-PLAYLIST-TYPE:VOD` line. When the type is `NONE` (the live-mode default), no `#EXT-X-PLAYLIST-TYPE` line is emitted.
- Sliding-window behavior in live mode is governed by `hls_list_size` (AVOption at `[libavformat/hlsenc.c:L3125]`, default 5 segments). When the per-variant segment list exceeds this size, `hls_delete_old_segments` at `[libavformat/hlsenc.c:L531]` unlinks the oldest segment files from disk (or issues `HTTP DELETE` requests through `hls->http_delete` when configured) and trims the linked list. The `hls_delete_threshold` AVOption at `[libavformat/hlsenc.c:L3126]` controls how many segments remain queued for deletion before being unlinked.
- The `HLS_OMIT_ENDLIST` flag (AVOption const `omit_endlist` at `[libavformat/hlsenc.c:L3150]`, decimal value `1 << 4`) suppresses the final `#EXT-X-ENDLIST` line even when in VOD mode. This is useful for streams that may resume later — clients see a VOD-shaped playlist but the server reserves the right to add more segments.
- The `HLS_APPEND_LIST` flag (AVOption const `append_list` at `[libavformat/hlsenc.c:L3152]`, decimal value `1 << 6`) activates append-to-existing-playlist mode. When set, the muxer reads the existing playlist file via `parse_playlist` at `[libavformat/hlsenc.c:L1162]`, picks up where it left off (preserving the existing sequence number), seeds the wall-clock anchor by parsing any prior `#EXT-X-PROGRAM-DATE-TIME` line at `[libavformat/hlsenc.c:L1226-L1244]`, and forces a discontinuity at the resume point (see §9). This mode is typically combined with `EVENT` playlist type to produce a server that survives restarts without breaking client playback.
- Live versus VOD also affects how the playlist file is delivered: with `HLS_TEMP_FILE` set, every publish in live mode writes to a `.tmp` file and renames atomically (so polling clients never see a half-written manifest); without the flag the muxer writes directly.

## Component: Discontinuity Handling

HLS playlists can declare *discontinuities* — points in the segment stream where one or more of the bitstream properties change abruptly. Typical reasons include: timestamps reset (e.g., a server restart that re-anchors PTS at zero), codec parameters change (e.g., switching from one input source to another), or content boundaries (e.g., the end of one program and the start of the next in a 24/7 channel). The HLS specification requires the playlist to mark these transitions with a `#EXT-X-DISCONTINUITY` tag so the client player flushes its decoder state and re-initializes its bitstream parsers at the boundary.

The muxer emits a discontinuity tag in two situations. First, when the publisher knows up-front that the playlist will be a continuation of a prior stream and the prior stream is no longer guaranteed to share the same bitstream parameters — typically signaled by enabling the `discont_start` flag or by enabling append-list mode. Second, when an individual segment has an internal discontinuity flag set, which propagates a `#EXT-X-DISCONTINUITY` line in front of that segment's entry in the playlist.

### Technical detail

- The `HLS_DISCONT_START` flag (AVOption const `discont_start` at `[libavformat/hlsenc.c:L3149]`, decimal value `1 << 3`) requests a `#EXT-X-DISCONTINUITY` line at the top of the playlist's segment list. The emission point is `[libavformat/hlsenc.c:L1593-L1596]`; emission is gated on the flag being set, the current sequence equaling `hls->start_sequence`, and the per-variant `discontinuity_set` field being zero (so the line is emitted exactly once).
- A per-segment discontinuity is carried on the `HLSSegment::discont` field (struct at `[libavformat/hlsenc.c:L76-L94]`, declared `int` at L82). The propagation from `vs->discontinuity` (transient variant-level flag) to `en->discont` (persistent per-segment flag) happens inside `hls_append_segment` at `[libavformat/hlsenc.c:L1090-L1093]`:

  ```c
  if (vs->discontinuity) { en->discont = 1; vs->discontinuity = 0; }
  ```

  The variant-level flag is set in several places: by `parse_playlist` when resuming an existing list (append-list mode), and at any point in `hls_write_packet` where the muxer detects a timestamp gap exceeding a threshold.
- The per-segment emission lives inside `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L156-L158]`: when `insert_discont` (the writer's first parameter, populated from `en->discont`) is nonzero, `#EXT-X-DISCONTINUITY\n` is emitted on its own line *before* the `#EXTINF` line for that segment.
- When the `HLS_PROGRAM_DATE_TIME` flag (AVOption const `program_date_time` at `[libavformat/hlsenc.c:L3153]`, decimal value `1 << 7`) is enabled together with a discontinuity, the wall-clock anchor must be recomputed because the discontinuity may include a clock jump. The recompute path is in `parse_playlist` at `[libavformat/hlsenc.c:L1226-L1244]` (parsing the existing playlist's last `#EXT-X-PROGRAM-DATE-TIME` and converting it to a `discont_program_date_time` value via `av_timegm` and the per-segment duration). The new anchor is then emitted by `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L167-L193]` with millisecond precision in ISO-8601 format including a timezone offset.
- The `HLS_APPEND_LIST` mode (see §8) automatically sets `vs->discontinuity = 1` after reading the existing playlist, which propagates to the next segment's `discont` flag and produces a `#EXT-X-DISCONTINUITY` line at the resume point. This is the muxer's default behavior on restart and ensures clients flush their decoder state at the boundary.


## Component: HLS Demuxer Probe

When an application opens an input file or URL through `avformat_open_input` and does not specify which demuxer to use, FFmpeg samples the first several kilobytes of the input and asks each registered demuxer "does this look like your format?" through a probe callback. The HLS demuxer's probe inspects the sampled bytes for the M3U8 magic line plus one or more HLS-specific tags that distinguish HLS playlists from generic plain-text M3U files used by other audio players.

A successful probe returns a confidence score; the demuxer with the highest score is selected. The HLS probe is deliberately strict about which tags it accepts as evidence — a file beginning with `#EXTM3U` but containing nothing else is not HLS (it could be a Winamp-style playlist), but a file containing `#EXT-X-STREAM-INF`, `#EXT-X-TARGETDURATION`, `#EXT-X-MEDIA-SEQUENCE`, or `#EXTINF` along with the magic line is unambiguously HLS.

### Technical detail

- The probe function is `hls_probe` at `[libavformat/hls.c:L2814]`. It inspects `AVProbeData::buf` (the sampled buffer) for the `#EXTM3U` magic line plus discriminating HLS-specific tags.
- The demuxer is registered at `[libavformat/hls.c:L2900-L2912]` as `const FFInputFormat ff_hls_demuxer` with `.p.name = "hls"`, `.p.long_name = "Apple HTTP Live Streaming"`, `.read_probe = hls_probe` slot at `[libavformat/hls.c:L2907]`, plus a `.priv_data_size = sizeof(HLSContext)` allocation hint and a `.flags_internal = FF_INFMT_FLAG_INIT_CLEANUP` hint that calls `read_close` even on a `read_header` failure.
- Registration flags `.p.flags = AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`:
  - `AVFMT_NOGENSEARCH` — disables the generic byte-offset search; HLS does not support byte-offset seeking into the abstract media timeline.
  - `AVFMT_TS_DISCONT` — declares that timestamp discontinuities across segment boundaries are normal and expected (downstream timestamp consumers should not interpret them as errors).
  - `AVFMT_NO_BYTE_SEEK` — declares that seeking is by segment index, not by byte offset.
  - `AVFMT_SHOW_IDS` — declares that stream IDs assigned by the demuxer are meaningful and should be exposed to the application.
- The probe is purely text-based and never reads the network. Once it confirms the input looks like HLS, the rest of the demuxer takes over with `hls_read_header` (see §11).

## Component: HLS Demuxer Playlist Parser

After the probe confirms the input is HLS, the demuxer's main job is to translate the M3U8 manifest from text into in-memory data structures the rest of the demuxer reads from. The playlist parser scans the manifest line by line, recognizes every `#EXT-X-*` tag defined by the HLS specification (RFC 8216), and builds three linked structures: a list of *variants* (the alternative bitrate ladders advertised in the master playlist), a list of *renditions* (the alternative audio and subtitle tracks), and a list of *playlists* (each containing a list of segments with their URLs, durations, and encryption metadata).

The parser is reentrant: it can be called repeatedly to refresh a live playlist, picking up new segments at the bottom while preserving any state already learned from prior parses. This is essential for live mode where the demuxer must poll the playlist URL at the cadence implied by `#EXT-X-TARGETDURATION` and pick up newly added segments without restarting.

### Technical detail

- Lifecycle entry points (all in `[libavformat/hls.c]`):
  - `hls_read_header` at `[libavformat/hls.c:L2144]` — invoked once by `avformat_open_input` after a successful probe. Calls the parser on the master playlist, opens sub-demuxers for each selected variant, and registers each elementary stream with the application.
  - `hls_read_packet` at `[libavformat/hls.c:L2546]` — invoked by `av_read_frame`. Picks the next playlist whose accumulated PTS is smallest, reads the next packet from that playlist's sub-demuxer (re-fetching the next segment from the network if needed), and returns the packet to the application.
  - `hls_read_seek` at `[libavformat/hls.c:L2709]` — invoked by `av_seek_frame`. Maps a target timestamp to a segment index, restarts the affected sub-demuxers at the matching segment, and clears the queued packet state.
  - `hls_close` at `[libavformat/hls.c:L2127]` — invoked on close. Frees all parsed playlist data, closes sub-demuxers, and releases network state.
- The main parser body (the line-by-line tag scanner) lives around `[libavformat/hls.c:L863-L989]` inside the `parse_playlist` function. Each line is matched against a series of tag prefixes and dispatched to a per-tag handler that updates the in-progress `struct playlist` record.
- Key data structures:
  - `struct segment` at `[libavformat/hls.c:L77-L87]` — per-segment record with `duration`, `url_offset`, `size`, `url`, `key`, `key_type`, `iv[16]`, `init_section`.
  - `struct playlist` at `[libavformat/hls.c:L102-L178]` — per-media-playlist state including `url`, `pb` (the playlist's own AVIOContext for fetching), `read_buffer`, `input`/`input_next` (segment fetchers), `parent` (back-pointer to the demuxer's `AVFormatContext`), `ctx` (the sub-demuxer's `AVFormatContext`), `n_segments`/`segments`, `cur_seq_no`/`last_seq_no` (current and last-known sequence numbers), `m3u8_hold_counters` (retry budget for unchanged live playlists), `key[16]`/`key_url`, ID3 state for ID3-timestamped audio streams, `audio_setup_info` (Sample-AES audio metadata), `n_renditions`/`renditions`, `n_init_sections`/`init_sections`.
  - `struct rendition` at `[libavformat/hls.c:L185-L192]` — alternative audio/subtitle/closed-captions track with `type`, back-pointer `playlist`, `group_id`, `language`, `name`, `disposition` (matched against the master playlist's `#EXT-X-MEDIA` lines).
  - `struct variant` at `[libavformat/hls.c:L194-L204]` — top-level master-playlist variant with `bandwidth`, list of `playlists`, plus `audio_group`/`video_group`/`subtitles_group` matching strings.
  - `HLSContext` (demuxer top level) at `[libavformat/hls.c:L206+]` — owns the `AVClass`, the parent `AVFormatContext`, the variants array, the playlists array, the renditions array.
- Encryption-method enumeration at `[libavformat/hls.c:L71-L75]`: `KEY_NONE = 0`, `KEY_AES_128 = 1`, `KEY_SAMPLE_AES = 2`. Set on each `struct segment::key_type` field as the parser sees `#EXT-X-KEY:METHOD=AES-128` or `#EXT-X-KEY:METHOD=SAMPLE-AES` lines.
- Playlist-type enumeration (demuxer-side) at `[libavformat/hls.c:L91-L95]`: `PLS_TYPE_UNSPECIFIED = 0`, `PLS_TYPE_EVENT = 1`, `PLS_TYPE_VOD = 2`. Set on each `struct playlist::type` field from `#EXT-X-PLAYLIST-TYPE` lines.
- Live-mode refresh loop is bounded by `m3u8_hold_counters` (AVOption at `[libavformat/hls.c:L2878-L2879]`, default 1000). When the parser re-reads a live playlist and observes no new segments, it decrements the counter and retries after a delay; reaching zero terminates the stream with `AVERROR_EOF` (the server is no longer publishing).
- The RFC 8216 specification URL is referenced in the file-level Doxygen comment at `[libavformat/hls.c:L26]` (the surrounding `@file` block documents the demuxer as "Apple HTTP Live Streaming demuxer" with the RFC URL on the adjacent line).
- The full inventory of tags accepted by the parser is enumerated in `./inputs-outputs.md` §10 (Demuxer Input Table).

## Component: HLS Sample Encryption

HLS also supports a second encryption scheme called *Sample Encryption* or *Sample-AES*. Where ordinary HLS encryption (see §5) encrypts an entire segment file with AES-128 in CBC mode (so the segment is opaque without the key), Sample-AES encrypts only the *samples* — the audio frames or the encoded video samples inside a segment — leaving the MPEG transport-stream packet headers and the segment's structural framing in clear text. The result is that a Sample-AES-aware demuxer can parse the segment structure without the key (it can read the timing and stream layout) but the application's decoders cannot actually decode anything without the key.

The HLS demuxer supports Sample-AES decryption for H.264 video and for AAC, AC-3, and E-AC-3 audio. Sample-AES is signaled at multiple levels: the playlist tags the affected stream with `#EXT-X-KEY:METHOD=SAMPLE-AES`, the transport-stream PMT advertises a Sample-AES-specific stream type (a value in the range reserved by the HLS Sample Encryption specification), and for AAC streams an ID3 PRIV tag carrying audio setup information is prepended to each segment so the demuxer knows codec parameters before decoder negotiation.

### Technical detail

- Implementation file: `[libavformat/hls_sample_encryption.c]` (396 lines). It implements three exported functions plus internal helpers for ADTS parsing, AC-3 parsing, and H.264 NAL unit walking.
- Exported function prototypes are at `[libavformat/hls_sample_encryption.h:L59-L63]`:
  - `ff_hls_senc_read_audio_setup_info` at `[libavformat/hls_sample_encryption.c:L61]` — parses the audio setup blob delivered as ID3 metadata.
  - `ff_hls_senc_parse_audio_setup_info` at `[libavformat/hls_sample_encryption.c:L94]` — translates the parsed blob into AVStream codec parameters.
  - `ff_hls_senc_decrypt_frame` at `[libavformat/hls_sample_encryption.c:L388]` — performs the actual per-frame Sample-AES decryption: walks NAL units for H.264 or audio frame headers for AAC/AC-3/E-AC-3 and decrypts the body region in-place.
- Crypto state struct `HLSCryptoContext` at `[libavformat/hls_sample_encryption.h:L43-L47]` carries the AES context pointer (`AVAES *aes_ctx`) plus the 16-byte key and 16-byte IV buffers.
- Audio setup struct `HLSAudioSetupInfo` at `[libavformat/hls_sample_encryption.h:L49-L56]` carries the codec identifier, codec tag, priming sample count, version, setup-data length, and a 10-byte (plus padding) setup-data buffer populated from the ID3 PRIV tag prepended to each AAC segment.
- Constants at `[libavformat/hls_sample_encryption.h:L40-L41]`: `HLS_MAX_ID3_TAGS_DATA_LEN = 138`, `HLS_MAX_AUDIO_SETUP_DATA_LEN = 10`.
- The MPEG transport-stream stream-type values that signal HLS Sample-AES in a PMT live at `[libavformat/mpegts.h:L177-L180]`:
  - `STREAM_TYPE_HLS_SE_VIDEO_H264 = 0xdb` — H.264 video, Sample-AES encrypted.
  - `STREAM_TYPE_HLS_SE_AUDIO_AAC  = 0xcf` — AAC audio, Sample-AES encrypted.
  - `STREAM_TYPE_HLS_SE_AUDIO_AC3  = 0xc1` — AC-3 audio, Sample-AES encrypted.
  - `STREAM_TYPE_HLS_SE_AUDIO_EAC3 = 0xc2` — Enhanced AC-3 audio, Sample-AES encrypted.
  These values are read by the MPEG-TS sub-demuxer when parsing the PMT and dispatched to `ff_hls_senc_decrypt_frame` for each affected packet.
- The demuxer-side `struct playlist` (at `[libavformat/hls.c:L102-L178]`) carries the per-playlist `audio_setup_info` field which is populated by `ff_hls_senc_read_audio_setup_info` when an ID3 PRIV tag is encountered at segment boundaries.
- **Sample Encryption is a demuxer-only feature in this codebase.** The build system at `[libavformat/Makefile:L276-L277]` links `hls_sample_encryption.o` only into the HLS demuxer object set (`OBJS-$(CONFIG_HLS_DEMUXER) += hls.o hls_sample_encryption.o`), not into the HLS muxer (`OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`). The muxer does not natively *produce* Sample-AES-encrypted segments; for that workflow the publisher must use a separate Sample-AES encryptor and feed the encrypted segments through a generic file-publishing path.


## Component: Playlist Tag Writers (Shared with DASH Muxer)

The actual `#EXT-X-*` lines of an M3U8 playlist file are not emitted by the muxer's main translation unit directly. Instead, the muxer calls into a small set of helper functions living in a separate translation unit, each of which emits one specific category of HLS tag. This factoring exists because the FFmpeg DASH muxer also publishes a side-channel M3U8 playlist alongside its primary DASH manifest (for clients that prefer HLS over DASH), and both muxers must produce byte-identical playlist output for the tags they share. Centralizing the writers guarantees both formats stay in lockstep.

This component is therefore a *shared library*, not a private muxer subroutine. The same code path executes whether the user is publishing through the HLS muxer or through the DASH muxer's HLS side-channel, which means: any change to the wire-format output of these helpers affects both formats simultaneously. The eight writer functions cover the full set of standard HLS tags the FFmpeg muxers produce.

### Technical detail

- Translation unit: `[libavformat/hlsplaylist.c]` (206 lines) with prototypes in `[libavformat/hlsplaylist.h]` (65 lines).
- The eight exported writer functions are declared at `[libavformat/hlsplaylist.h:L38-L63]`:
  1. `ff_hls_write_playlist_version` at `[libavformat/hlsplaylist.c:L32-L38]` — emits `#EXTM3U` on line 1 and `#EXT-X-VERSION:%d` on line 2. Always the first two lines of every playlist.
  2. `ff_hls_write_audio_rendition` at `[libavformat/hlsplaylist.c:L40-L56]` — emits one `#EXT-X-MEDIA:TYPE=AUDIO,...` line per audio-only rendition. Attributes: `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE`, `CHANNELS`, `URI`.
  3. `ff_hls_write_subtitle_rendition` at `[libavformat/hlsplaylist.c:L58-L76]` — emits one `#EXT-X-MEDIA:TYPE=SUBTITLES,...` line per subtitle rendition. Attributes: `GROUP-ID`, `NAME`, `DEFAULT`, `LANGUAGE`, `URI`.
  4. `ff_hls_write_stream_info` at `[libavformat/hlsplaylist.c:L78-L108]` — emits one `#EXT-X-STREAM-INF:...` line per variant in the master playlist. Attributes: `BANDWIDTH`, `AVERAGE-BANDWIDTH`, `RESOLUTION=WxH`, `FRAME-RATE`, `CODECS`, optional `AUDIO`/`SUBTITLES`/`CLOSED-CAPTIONS` group references. The variant's playlist URL follows on the next line.
  5. `ff_hls_write_playlist_header` at `[libavformat/hlsplaylist.c:L110-L132]` — emits (after calling `ff_hls_write_playlist_version` internally for the version line) the fixed top-of-playlist tags: `#EXT-X-ALLOW-CACHE`, `#EXT-X-TARGETDURATION:%d`, `#EXT-X-MEDIA-SEQUENCE:%lld`, optional `#EXT-X-PLAYLIST-TYPE`, optional `#EXT-X-I-FRAMES-ONLY`.
  6. `ff_hls_write_init_file` at `[libavformat/hlsplaylist.c:L134-L142]` — emits the `#EXT-X-MAP:URI="<file>",BYTERANGE="<size>@<offset>"` line that references the fragmented-MP4 initialization segment. Called once per playlist when `segment_type=fmp4`.
  7. `ff_hls_write_file_entry` at `[libavformat/hlsplaylist.c:L144-L199]` — emits the per-segment block: optional `#EXT-X-DISCONTINUITY`, the `#EXTINF:%f,` duration line, optional `#EXT-X-BYTERANGE:%lld@%lld`, optional `#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s` with timezone-offset suffix, and the segment URL on its own line.
  8. `ff_hls_write_end_list` at `[libavformat/hlsplaylist.c:L201-L206]` — emits the `#EXT-X-ENDLIST` line. Called once at the end of a finished playlist (VOD mode or trailer publish with `HLS_OMIT_ENDLIST` not set).
- **DASH coupling** is established in the build system at `[libavformat/Makefile:L189]`:

  ```text
  OBJS-$(CONFIG_DASH_MUXER) += dash.o dashenc.o hlsplaylist.o
  ```

  `hlsplaylist.o` is linked into the DASH muxer's object set so when DASH is built with HLS-side-channel support enabled, it can call these writers directly. The HLS muxer's own linkage at `[libavformat/Makefile:L277]` is `OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`. Both muxers therefore depend on the same compiled artifact.
- The shared status of this translation unit has integrator-facing implications, enumerated in `./consumer-dependencies.md` §6.
- The `PlaylistType` enumeration used by `ff_hls_write_playlist_header` is defined in the writer's own header at `[libavformat/hlsplaylist.h:L31-L36]`. It is distinct from the demuxer-side playlist type at `[libavformat/hls.c:L91-L95]` despite carrying similar semantics — the demuxer and muxer use separate type spaces.

## Lifecycle Roll-Up

The muxer's externally visible lifecycle is the standard FFmpeg muxer pattern: an init phase that validates options and allocates per-variant state, a write-header phase that opens the first segment per variant, a write-packet loop that handles one packet at a time and cuts segments at keyframe boundaries, a write-trailer phase that flushes any pending segment and publishes the final playlist (with `#EXT-X-ENDLIST` if applicable), and a deinit phase that frees all state. The five phases are bound into FFmpeg's muxer-dispatch table by the `FFOutputFormat ff_hls_muxer` registration record at the bottom of the muxer source file. Engineering detail on each phase and on the internal callback chain lives in `../technical/pipeline-orchestration.md`; this section is the quick locator map.

| Phase | Function | Source |
|---|---|---|
| Init | `hls_init` | `[libavformat/hlsenc.c:L2866]` |
| Write Header | `hls_write_header` | `[libavformat/hlsenc.c:L2301]` |
| Write Packet | `hls_write_packet` | `[libavformat/hlsenc.c:L2410]` |
| Write Trailer | `hls_write_trailer` | `[libavformat/hlsenc.c:L2727]` |
| Deinit | `hls_deinit` | `[libavformat/hlsenc.c:L2693]` |
| FFOutputFormat registration | `ff_hls_muxer` | `[libavformat/hlsenc.c:L3191-L3207]` |

The demuxer side has its own four-phase lifecycle bound by the `FFInputFormat ff_hls_demuxer` registration record:

| Phase | Function | Source |
|---|---|---|
| Probe | `hls_probe` | `[libavformat/hls.c:L2814]` |
| Read Header | `hls_read_header` | `[libavformat/hls.c:L2144]` |
| Read Packet | `hls_read_packet` | `[libavformat/hls.c:L2546]` |
| Read Seek | `hls_read_seek` | `[libavformat/hls.c:L2709]` |
| Read Close | `hls_close` | `[libavformat/hls.c:L2127]` |
| FFInputFormat registration | `ff_hls_demuxer` | `[libavformat/hls.c:L2900-L2912]` |

Lifecycle coverage is duplicated here for navigational convenience; the authoritative narrative for each phase, the internal callback chain, and the segment-finalization sequencing live in `../technical/pipeline-orchestration.md`.

