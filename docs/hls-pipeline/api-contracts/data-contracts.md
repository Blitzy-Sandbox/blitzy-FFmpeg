# Data Contracts — HLS Pipeline API Contracts

> All source references in this document are anchored to commit `566ad786` (full: `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). See [`../README.md`](../README.md) for the citation convention and glossary.

## Overview

These are the bytes-on-the-wire and bits-in-memory data contracts the FFmpeg HLS pipeline relies on. Every struct field that participates in wire emission, every option a caller can tune, every timestamp convention, every binary layout, every enum value that crosses an API or wire boundary is contracted in this document. A port that changes any value documented here is a breaking change for some downstream consumer — a media player parsing an M3U8 playlist, a CDN ingest endpoint receiving HTTP PUT requests, a decrypter consuming a Sample-AES stream, or an application linking against `libavformat` and constructing an `AVFormatContext` configured for the `hls` output format.

The audience is engineers verifying that a port preserves every data shape, type, and value range the existing pipeline produces or consumes.

### How to read the tables

- Struct field tables enumerate every field of a struct in source order, with C type, defining lines in source, the cross-boundary effect of the field (what depends on its exact value or layout), and an inline citation. The "no summarizing" rule (`README.md` rule 0.10.9) applies: every field has its own row.
- `AVOption` tables enumerate every entry of the `options[]` array, including `AV_OPT_TYPE_CONST` aliases that expose named flag and enum values to the option parser. The columns capture the C field bound by `OFFSET()`, the option type, default value, minimum, maximum, encoder/decoder access flags, and the option unit (used to group `AV_OPT_TYPE_CONST` aliases under their parent option).
- Enum and constant tables list every symbol with its value, the meaning of the value, and the citation.
- Binary layout tables list offset, field, C type, and on-the-wire size.
- Wire-format tables list each format string verbatim, the C identifier of the printf-style emission site, and the producing function.

Inline source citations use the form `[<repo-relative-path>:L<n>]` for single-line references or `[<repo-relative-path>:L<start>-L<end>]` for ranges, anchored to commit `566ad786`. Inferred claims (where source does not directly state a behavior but it can be reasonably deduced) carry the tag `[inferred — no direct source]`.

## Contract: HLSContext Field Formats (Muxer)

The muxer-side `HLSContext` is the top-level state struct allocated once per output session. Its address is stored at `AVFormatContext::priv_data`, and its size is exposed through `FFOutputFormat::priv_data_size = sizeof(HLSContext)` `[libavformat/hlsenc.c:L3201]`. Every public-effect field — those that drive wire emission, configure the encryption path, control the lifecycle of child sub-muxers, or are bound by an `AVOption` — appears in the table below. The struct definition itself spans `[libavformat/hlsenc.c:L202-L267]`.

A port that changes the type, default value, or semantics of any field below changes either the muxer's user-facing CLI/API contract (for `AVOption`-bound fields) or its internal wire-emission behavior (for runtime-computed fields).

| Field | C Type | Lines | Wire/Boundary Effect | Citation |
|-------|--------|-------|----------------------|----------|
| `class` | `const AVClass *` | L203 | Required first field for `AVOption` access via `av_opt_*` APIs; points to `hls_class`. | `[libavformat/hlsenc.c:L203]` |
| `start_sequence` | `int64_t` | L204 | First `EXT-X-MEDIA-SEQUENCE` value or seed value depending on `start_sequence_source_type`; bound by the `start_number` `AVOption`. | `[libavformat/hlsenc.c:L204]` |
| `start_sequence_source_type` | `uint32_t` | L205 | Selects mode among the four `StartSequenceSourceType` enum values (start_number / epoch / epoch_us / datetime); bound by `hls_start_number_source`. | `[libavformat/hlsenc.c:L205]` |
| `time` | `int64_t` | L207 | Target segment duration in microseconds; bound by `hls_time` `AVOption`; default 2,000,000 (2 seconds). | `[libavformat/hlsenc.c:L207]` |
| `init_time` | `int64_t` | L208 | Initial segment duration in microseconds used until `start_sequence + max_nb_segments` segments have been published; bound by `hls_init_time`. | `[libavformat/hlsenc.c:L208]` |
| `max_nb_segments` | `int` | L209 | Maximum number of segments retained in the playlist (sliding window); bound by `hls_list_size`; default 5. | `[libavformat/hlsenc.c:L209]` |
| `hls_delete_threshold` | `int` | L210 | Number of unreferenced segments to retain on disk before deleting; bound by `hls_delete_threshold`; default 1. | `[libavformat/hlsenc.c:L210]` |
| `flags` | `uint32_t` | L211 | Bitfield of `HLSFlags` values that gate optional emission paths; bound by `hls_flags`. | `[libavformat/hlsenc.c:L211]` |
| `pl_type` | `uint32_t` | L212 | Selects `PlaylistType` (NONE/EVENT/VOD); bound by `hls_playlist_type`; controls `EXT-X-PLAYLIST-TYPE` emission. | `[libavformat/hlsenc.c:L212]` |
| `segment_filename` | `char *` | L213 | Filename template (e.g., `out%d.ts`) for segment output; bound by `hls_segment_filename`. | `[libavformat/hlsenc.c:L213]` |
| `fmp4_init_filename` | `char *` | L214 | Filename of fMP4 initialization segment; bound by `hls_fmp4_init_filename`; default `"init.mp4"`. | `[libavformat/hlsenc.c:L214]` |
| `segment_type` | `int` | L215 | Selects `SegmentType` (MPEGTS=0 / FMP4=1); bound by `hls_segment_type`. | `[libavformat/hlsenc.c:L215]` |
| `resend_init_file` | `int` | L216 | Bool: whether to rewrite the fMP4 init file each segment cycle; bound by `hls_fmp4_init_resend`; default 0. | `[libavformat/hlsenc.c:L216]` |
| `use_localtime` | `int` | L218 | Bool: enables `strftime` expansion of `%`-tokens in `hls_segment_filename`; bound by `strftime`; default 0. | `[libavformat/hlsenc.c:L218]` |
| `use_localtime_mkdir` | `int` | L219 | Bool: auto-creates parent directories when `use_localtime` produces nested paths; bound by `strftime_mkdir`. | `[libavformat/hlsenc.c:L219]` |
| `allowcache` | `int` | L220 | Tri-state (-1 unset, 0 NO, 1 YES); controls `EXT-X-ALLOW-CACHE:YES/NO` emission; bound by `hls_allow_cache`; default -1. | `[libavformat/hlsenc.c:L220]` |
| `recording_time` | `int64_t` | L221 | Internal: cached value derived from `time * max_nb_segments`; used as the cross-segment scheduling deadline. | `[libavformat/hlsenc.c:L221]` |
| `max_seg_size` | `int64_t` | L222 | Maximum bytes per segment when single-file byterange mode is engaged; bound by `hls_segment_size`. | `[libavformat/hlsenc.c:L222]` |
| `baseurl` | `char *` | L224 | Prefix string prepended to every segment URI in the playlist; bound by `hls_base_url`. | `[libavformat/hlsenc.c:L224]` |
| `vtt_format_options_str` | `char *` | L225 | Format-options string passed to the WebVTT child muxer; bound by `hls_vtt_options`. | `[libavformat/hlsenc.c:L225]` |
| `subtitle_filename` | `char *` | L226 | Output filename for subtitle segments; bound by `hls_subtitle_path`. | `[libavformat/hlsenc.c:L226]` |
| `format_options` | `AVDictionary *` | L227 | Dictionary of key=value pairs passed to the segment sub-muxer; bound by `hls_segment_options`. | `[libavformat/hlsenc.c:L227]` |
| `encrypt` | `int` | L229 | Bool: enables AES-128 encryption via auto-generated key; bound by `hls_enc`; default 0. | `[libavformat/hlsenc.c:L229]` |
| `key` | `char *` | L230 | Raw key bytes (hex-encoded) supplied via `hls_enc_key` `AVOption`. | `[libavformat/hlsenc.c:L230]` |
| `key_url` | `char *` | L231 | URI emitted in `EXT-X-KEY` line when `encrypt` is enabled; bound by `hls_enc_key_url`. | `[libavformat/hlsenc.c:L231]` |
| `iv` | `char *` | L232 | User-supplied initialization vector (32 hex chars → 16 bytes); bound by `hls_enc_iv`. | `[libavformat/hlsenc.c:L232]` |
| `key_basename` | `char *` | L233 | Internal: computed key filename derived from master playlist URL or `s->url` plus `.key` suffix. | `[libavformat/hlsenc.c:L233]` |
| `encrypt_started` | `int` | L234 | Internal flag: 1 once `do_encrypt` has been invoked successfully for this session. | `[libavformat/hlsenc.c:L234]` |
| `key_info_file` | `char *` | L236 | Path to 3-line text file (URI, key path, optional IV) for periodic-rekey workflow; bound by `hls_key_info_file`. | `[libavformat/hlsenc.c:L236]` |
| `key_file` | `char[LINE_BUFFER_SIZE + 1]` | L237 | Cached path to current key file (read from line 2 of `key_info_file`). | `[libavformat/hlsenc.c:L237]` |
| `key_uri` | `char[LINE_BUFFER_SIZE + 1]` | L238 | Cached URI emitted in `EXT-X-KEY` (read from line 1 of `key_info_file`). | `[libavformat/hlsenc.c:L238]` |
| `key_string` | `char[KEYSIZE*2 + 1]` | L239 | Hex-encoded current key (32 chars + NUL); produced by `ff_data_to_hex`. | `[libavformat/hlsenc.c:L239]` |
| `iv_string` | `char[KEYSIZE*2 + 1]` | L240 | Hex-encoded current IV (32 chars + NUL); emitted as `IV=0x<hex>` in `EXT-X-KEY` when non-empty. | `[libavformat/hlsenc.c:L240]` |
| `vtt_format_options` | `AVDictionary *` | L241 | Parsed dictionary form of `vtt_format_options_str`. | `[libavformat/hlsenc.c:L241]` |
| `method` | `char *` | L243 | HTTP method for segment uploads; default is implicitly `PUT`; bound by `method` `AVOption`. | `[libavformat/hlsenc.c:L243]` |
| `user_agent` | `char *` | L244 | HTTP `User-Agent` header forwarded to sub-muxer's HTTP child; bound by `http_user_agent`. | `[libavformat/hlsenc.c:L244]` |
| `var_streams` | `VariantStream *` | L246 | Dynamically allocated array of per-variant state structs (one per bitrate ladder entry). | `[libavformat/hlsenc.c:L246]` |
| `nb_varstreams` | `unsigned int` | L247 | Count of entries in `var_streams`; defaults to 1 if `var_stream_map` is empty. | `[libavformat/hlsenc.c:L247]` |
| `cc_streams` | `ClosedCaptionsStream *` | L248 | Dynamically allocated array of closed-captions stream descriptors. | `[libavformat/hlsenc.c:L248]` |
| `nb_ccstreams` | `unsigned int` | L249 | Count of entries in `cc_streams`. | `[libavformat/hlsenc.c:L249]` |
| `master_m3u8_created` | `int` | L251 | Internal flag: 1 once the master playlist has been emitted for the first time. | `[libavformat/hlsenc.c:L251]` |
| `master_m3u8_url` | `char *` | L252 | Cached final URL of the master playlist; used as base for relative key URIs. | `[libavformat/hlsenc.c:L252]` |
| `version` | `int` | L253 | HLS protocol version (`EXT-X-VERSION`); computed at write time from feature usage; values in {2, 3, 4, 6, 7}. | `[libavformat/hlsenc.c:L253]` |
| `var_stream_map` | `char *` | L254 | Variant-stream mapping specification (e.g., `"v:0,a:0 v:1,a:1"`); bound by `var_stream_map`. | `[libavformat/hlsenc.c:L254]` |
| `cc_stream_map` | `char *` | L255 | Closed-captions stream mapping specification; bound by `cc_stream_map`. | `[libavformat/hlsenc.c:L255]` |
| `master_pl_name` | `char *` | L256 | Filename of master playlist (if any); bound by `master_pl_name`. | `[libavformat/hlsenc.c:L256]` |
| `master_publish_rate` | `unsigned int` | L257 | Republish master playlist every N child-segment cycles; bound by `master_pl_publish_rate`. | `[libavformat/hlsenc.c:L257]` |
| `http_persistent` | `int` | L258 | Bool: reuse HTTP TCP connection across segment uploads; bound by `http_persistent`; default 0. | `[libavformat/hlsenc.c:L258]` |
| `m3u8_out` | `AVIOContext *` | L259 | Output context for the current playlist write (master or media depending on call site). | `[libavformat/hlsenc.c:L259]` |
| `sub_m3u8_out` | `AVIOContext *` | L260 | Output context for the subtitle (`.vtt.m3u8`) playlist. | `[libavformat/hlsenc.c:L260]` |
| `http_delete` | `AVIOContext *` | L261 | Persistent HTTP DELETE channel for segment cleanup when `HLS_DELETE_SEGMENTS` is set. | `[libavformat/hlsenc.c:L261]` |
| `timeout` | `int64_t` | L262 | I/O timeout in microseconds for HTTP operations; bound by `timeout`; default -1 (no timeout). | `[libavformat/hlsenc.c:L262]` |
| `ignore_io_errors` | `int` | L263 | Bool: swallow segment-write errors instead of propagating; bound by `ignore_io_errors`; default 0. | `[libavformat/hlsenc.c:L263]` |
| `headers` | `char *` | L264 | Additional HTTP headers (CRLF-separated) forwarded to every HTTP I/O; bound by `headers`. | `[libavformat/hlsenc.c:L264]` |
| `has_default_key` | `int` | L265 | Internal: 1 if at least one variant has the default-key flag in `var_stream_map`. | `[libavformat/hlsenc.c:L265]` |
| `has_video_m3u8` | `int` | L266 | Internal: 1 if at least one variant carries a video stream — controls whether the master playlist emits `EXT-X-STREAM-INF` or only `EXT-X-MEDIA`. | `[libavformat/hlsenc.c:L266]` |

## Contract: VariantStream Field Formats

A `VariantStream` represents one entry in the bitrate ladder. The HLS muxer can emit a single playlist (one `VariantStream`) or a master playlist referencing many media playlists (one `VariantStream` per ladder rung). Each `VariantStream` wraps its own child `AVFormatContext *avf` (the MPEG-TS or fMP4 sub-muxer) and tracks an independent segment linked-list, sequence counter, and encryption state. The struct definition spans `[libavformat/hlsenc.c:L120-L194]`.

A port that changes any field type or semantic below changes the per-variant boundary between the HLS meta-muxer and its underlying sub-muxer.

| Field | C Type | Lines | Wire/Boundary Effect | Citation |
|-------|--------|-------|----------------------|----------|
| `var_stream_idx` | `unsigned` | L121 | Index into the parent `HLSContext::var_streams` array; embedded into `HLSSegment::var_stream_idx` for ownership tracking. | `[libavformat/hlsenc.c:L121]` |
| `number` | `unsigned` | L122 | User-facing variant number used in default master playlist `EXT-X-STREAM-INF` group naming. | `[libavformat/hlsenc.c:L122]` |
| `sequence` | `int64_t` | L123 | Current `EXT-X-MEDIA-SEQUENCE` value for this variant; monotonically incremented as segments are appended. | `[libavformat/hlsenc.c:L123]` |
| `oformat` | `const AVOutputFormat *` | L124 | Pointer to the child output format (`mpegts` or `mp4`/`fmp4`) selected per `HLSContext::segment_type`. | `[libavformat/hlsenc.c:L124]` |
| `vtt_oformat` | `const AVOutputFormat *` | L125 | Pointer to the WebVTT child output format used for subtitle tracks. | `[libavformat/hlsenc.c:L125]` |
| `out` | `AVIOContext *` | L126 | Currently active output context for the segment file (per-segment or single-file). | `[libavformat/hlsenc.c:L126]` |
| `out_single_file` | `AVIOContext *` | L127 | Output context for the long-running single-file mode (`HLS_SINGLE_FILE`), kept open across segments. | `[libavformat/hlsenc.c:L127]` |
| `packets_written` | `int` | L128 | Number of packets written to the current segment; gates the `can_split` decision in `hls_write_packet`. | `[libavformat/hlsenc.c:L128]` |
| `init_range_length` | `int` | L129 | Byte length of the fMP4 initialization box (`moov` etc.); emitted in the `BYTERANGE` portion of `EXT-X-MAP` when in single-file fMP4 mode. | `[libavformat/hlsenc.c:L129]` |
| `temp_buffer` | `uint8_t *` | L130 | Internal buffer used by the dynamic-buffer AVIO path when flushing fMP4 fragments. | `[libavformat/hlsenc.c:L130]` |
| `init_buffer` | `uint8_t *` | L131 | Buffered fMP4 init bytes captured from the first sub-muxer dynamic write; resent on `resend_init_file`. | `[libavformat/hlsenc.c:L131]` |
| `avf` | `AVFormatContext *` | L133 | Child sub-muxer context (MPEG-TS or fMP4) that owns codec parameters, packet timestamping, and segment-file emission. | `[libavformat/hlsenc.c:L133]` |
| `vtt_avf` | `AVFormatContext *` | L134 | Child WebVTT sub-muxer context for the subtitle stream of this variant. | `[libavformat/hlsenc.c:L134]` |
| `has_video` | `int` | L136 | Bool: 1 if this variant carries at least one video stream — controls `EXT-X-INDEPENDENT-SEGMENTS` emission and the segment-cut keyframe rule. | `[libavformat/hlsenc.c:L136]` |
| `has_subtitle` | `int` | L137 | Bool: 1 if this variant carries at least one subtitle stream. | `[libavformat/hlsenc.c:L137]` |
| `new_start` | `int` | L138 | Internal flag: 1 at the start of each segment cycle; triggers `start_pts`/`end_pts` reseed on first reference packet. | `[libavformat/hlsenc.c:L138]` |
| `start_pts_from_audio` | `int` | L139 | Bool: 1 if the variant has no video — the audio stream becomes the reference for `start_pts`/segment-cut decisions. | `[libavformat/hlsenc.c:L139]` |
| `dpp` | `double` | L140 | Duration per packet (seconds); cached from reference stream `time_base` × `pkt->duration`. | `[libavformat/hlsenc.c:L140]` |
| `start_pts` | `int64_t` | L141 | PTS of the first packet of the current segment (in reference-stream `time_base`). | `[libavformat/hlsenc.c:L141]` |
| `end_pts` | `int64_t` | L142 | PTS at which segment-cut comparison is evaluated; advances each reference packet. | `[libavformat/hlsenc.c:L142]` |
| `video_lastpos` | `int64_t` | L143 | Byte position in `out` of the last written video packet — used to compute `video_keyframe_size`. | `[libavformat/hlsenc.c:L143]` |
| `video_keyframe_pos` | `int64_t` | L144 | Byte position of the last keyframe boundary; emitted as `BYTERANGE` for I-frames-only playlists. | `[libavformat/hlsenc.c:L144]` |
| `video_keyframe_size` | `int64_t` | L145 | Byte size of the last keyframe block; emitted as `BYTERANGE` size for I-frames-only playlists. | `[libavformat/hlsenc.c:L145]` |
| `duration` | `double` | L146 | Accumulated duration of the current segment in seconds; rounded to integer and emitted as `EXTINF`. | `[libavformat/hlsenc.c:L146]` |
| `start_pos` | `int64_t` | L147 | Starting byte offset in `out` of the current segment (significant for byterange/single-file modes). | `[libavformat/hlsenc.c:L147]` |
| `size` | `int64_t` | L148 | Byte size of the last completed segment; emitted in `EXT-X-BYTERANGE`. | `[libavformat/hlsenc.c:L148]` |
| `nb_entries` | `int` | L149 | Count of segments currently in the `segments` linked list (matches playlist size). | `[libavformat/hlsenc.c:L149]` |
| `discontinuity_set` | `int` | L150 | Internal flag: 1 if `EXT-X-DISCONTINUITY-SEQUENCE` has been emitted at least once (prevents re-emission). | `[libavformat/hlsenc.c:L150]` |
| `discontinuity` | `int` | L151 | Flag set when an upstream gap is detected; forces `EXT-X-DISCONTINUITY` on the next segment entry. | `[libavformat/hlsenc.c:L151]` |
| `reference_stream_index` | `int` | L152 | Index into `streams` array of the stream whose `time_base` and PTS are authoritative for segment-cut. | `[libavformat/hlsenc.c:L152]` |
| `total_size` | `int64_t` | L154 | Sum of `size` across all segments — used to compute `avg_bitrate` for `EXT-X-STREAM-INF`. | `[libavformat/hlsenc.c:L154]` |
| `total_duration` | `double` | L155 | Sum of `duration` across all segments — used for `avg_bitrate`. | `[libavformat/hlsenc.c:L155]` |
| `avg_bitrate` | `int64_t` | L156 | Computed average bitrate (`total_size * 8 / total_duration`); emitted as `AVERAGE-BANDWIDTH` in `EXT-X-STREAM-INF`. | `[libavformat/hlsenc.c:L156]` |
| `max_bitrate` | `int64_t` | L157 | Maximum observed per-segment bitrate; emitted as `BANDWIDTH` in `EXT-X-STREAM-INF`. | `[libavformat/hlsenc.c:L157]` |
| `segments` | `HLSSegment *` | L159 | Head of the linked list of segments currently in the playlist. | `[libavformat/hlsenc.c:L159]` |
| `last_segment` | `HLSSegment *` | L160 | Tail pointer for O(1) append into `segments`. | `[libavformat/hlsenc.c:L160]` |
| `old_segments` | `HLSSegment *` | L161 | Head of the linked list of segments evicted from `segments` but pending file deletion (when `HLS_DELETE_SEGMENTS` is set). | `[libavformat/hlsenc.c:L161]` |
| `basename_tmp` | `char *` | L163 | Temporary filename used when `HLS_TEMP_FILE` flag is set (segment is written to `<name>.tmp` then renamed). | `[libavformat/hlsenc.c:L163]` |
| `basename` | `char *` | L164 | Resolved per-variant segment filename template (after `var_stream_map` substitution). | `[libavformat/hlsenc.c:L164]` |
| `vtt_basename` | `char *` | L165 | Resolved per-variant subtitle segment filename template. | `[libavformat/hlsenc.c:L165]` |
| `vtt_m3u8_name` | `char *` | L166 | Resolved per-variant subtitle playlist filename. | `[libavformat/hlsenc.c:L166]` |
| `m3u8_name` | `char *` | L167 | Resolved per-variant media playlist filename. | `[libavformat/hlsenc.c:L167]` |
| `initial_prog_date_time` | `double` | L169 | Unix-time-with-fractional-seconds anchor for the first segment when `HLS_PROGRAM_DATE_TIME` is set; subsequent PDT values advance by accumulated duration. | `[libavformat/hlsenc.c:L169]` |
| `current_segment_final_filename_fmt` | `char[MAX_URL_SIZE]` | L170 | Cached resolved filename for the segment currently being written (after `strftime`/`%d` expansion). | `[libavformat/hlsenc.c:L170]` |
| `fmp4_init_filename` | `char *` | L172 | Resolved per-variant fMP4 init filename (after `var_stream_map` substitution). | `[libavformat/hlsenc.c:L172]` |
| `base_output_dirname` | `char *` | L173 | Cached directory portion of the variant's output filename (used for `use_localtime_mkdir`). | `[libavformat/hlsenc.c:L173]` |
| `encrypt_started` | `int` | L175 | Internal flag: 1 once encryption setup has succeeded for this variant. | `[libavformat/hlsenc.c:L175]` |
| `key_file` | `char[LINE_BUFFER_SIZE + 1]` | L177 | Per-variant cached key file path (per `hls_key_info_file`). | `[libavformat/hlsenc.c:L177]` |
| `key_uri` | `char[LINE_BUFFER_SIZE + 1]` | L178 | Per-variant cached key URI emitted in `EXT-X-KEY`. | `[libavformat/hlsenc.c:L178]` |
| `key_string` | `char[KEYSIZE*2 + 1]` | L179 | Per-variant hex-encoded current key (32 chars + NUL). | `[libavformat/hlsenc.c:L179]` |
| `iv_string` | `char[KEYSIZE*2 + 1]` | L180 | Per-variant hex-encoded current IV (32 chars + NUL). | `[libavformat/hlsenc.c:L180]` |
| `streams` | `AVStream **` | L182 | Array of pointers to the parent `AVFormatContext::streams` entries that belong to this variant. | `[libavformat/hlsenc.c:L182]` |
| `codec_attr` | `char[128]` | L183 | Computed `CODECS` attribute value (RFC 6381) for `EXT-X-STREAM-INF`. | `[libavformat/hlsenc.c:L183]` |
| `attr_status` | `CodecAttributeStatus` | L184 | State machine: WRITTEN/WILL_NOT_BE_WRITTEN — prevents re-recomputation of `codec_attr`. | `[libavformat/hlsenc.c:L184]` |
| `nb_streams` | `unsigned int` | L185 | Count of entries in `streams`. | `[libavformat/hlsenc.c:L185]` |
| `m3u8_created` | `int` | L186 | Internal flag: 1 once this variant's media playlist has been written for the first time. | `[libavformat/hlsenc.c:L186]` |
| `is_default` | `int` | L187 | Bool: 1 if this variant has `DEFAULT=YES` in its `EXT-X-MEDIA` line. | `[libavformat/hlsenc.c:L187]` |
| `language` | `const char *` | L188 | RFC 5646 language tag emitted in `EXT-X-MEDIA:LANGUAGE`. | `[libavformat/hlsenc.c:L188]` |
| `agroup` | `const char *` | L189 | `EXT-X-MEDIA:GROUP-ID` for audio rendition. | `[libavformat/hlsenc.c:L189]` |
| `sgroup` | `const char *` | L190 | `EXT-X-MEDIA:GROUP-ID` for subtitle rendition. | `[libavformat/hlsenc.c:L190]` |
| `ccgroup` | `const char *` | L191 | `EXT-X-MEDIA:GROUP-ID` for closed-captions rendition. | `[libavformat/hlsenc.c:L191]` |
| `varname` | `const char *` | L192 | Variant name (used as substitution token in `master_pl_name`). | `[libavformat/hlsenc.c:L192]` |
| `subtitle_varname` | `const char *` | L193 | Subtitle variant name for substitution. | `[libavformat/hlsenc.c:L193]` |

## Contract: HLSSegment Field Formats

The `HLSSegment` struct is the per-segment metadata record held in a singly-linked list inside each `VariantStream`. One instance is allocated per segment that has been (or is being) emitted, plus one for any pending file deletion. The struct definition spans `[libavformat/hlsenc.c:L76-L94]`.

The struct uses a flexible-array tail (`char buf[]`) to colocate the `filename`, `sub_filename`, and `key_uri` C strings in a single allocation; the pointer fields above point into that tail. A port must preserve this allocation pattern to retain the public `next`-pointer ABI between FFmpeg builds at the same major version.

| Field | C Type | Lines | Wire/Boundary Effect | Citation |
|-------|--------|-------|----------------------|----------|
| `filename` | `const char *` | L77 | Segment URI as written to `EXTINF`'s following URL line (with optional `baseurl` prefix). | `[libavformat/hlsenc.c:L77]` |
| `sub_filename` | `const char *` | L78 | Subtitle segment URI for the paired `.vtt` segment, if any. | `[libavformat/hlsenc.c:L78]` |
| `duration` | `double` | L79 | Segment duration in seconds; emitted as `EXTINF:%f,\n` or `EXTINF:%ld,\n` (rounded) depending on `HLS_ROUND_DURATIONS`. | `[libavformat/hlsenc.c:L79]` |
| `discont` | `int` | L80 | 1 if this segment should be preceded by `EXT-X-DISCONTINUITY` in the playlist; consumed by `ff_hls_write_file_entry`. | `[libavformat/hlsenc.c:L80]` |
| `pos` | `int64_t` | L81 | Byte offset of the segment payload within the single output file (single-file/byterange modes); emitted after `@` in `EXT-X-BYTERANGE`. | `[libavformat/hlsenc.c:L81]` |
| `size` | `int64_t` | L82 | Byte length of the segment payload; emitted before `@` in `EXT-X-BYTERANGE`. | `[libavformat/hlsenc.c:L82]` |
| `keyframe_pos` | `int64_t` | L83 | Byte offset of the segment's keyframe block; used instead of `pos` when `HLS_I_FRAMES_ONLY` is set. | `[libavformat/hlsenc.c:L83]` |
| `keyframe_size` | `int64_t` | L84 | Byte length of the segment's keyframe block; used instead of `size` for `HLS_I_FRAMES_ONLY`. | `[libavformat/hlsenc.c:L84]` |
| `var_stream_idx` | `unsigned` | L85 | Index of the owning `VariantStream` within `HLSContext::var_streams`. | `[libavformat/hlsenc.c:L85]` |
| `key_uri` | `const char *` | L87 | Pointer into `buf[]` holding the key URI in effect for this segment (used to detect `EXT-X-KEY` boundary in the playlist writer). | `[libavformat/hlsenc.c:L87]` |
| `iv_string` | `char[KEYSIZE*2 + 1]` | L88 | Hex-encoded IV in effect for this segment (32 chars + NUL); emitted as `IV=0x<hex>` when non-empty. | `[libavformat/hlsenc.c:L88]` |
| `next` | `struct HLSSegment *` | L90 | Forward pointer in the singly-linked list of segments. | `[libavformat/hlsenc.c:L90]` |
| `discont_program_date_time` | `double` | L91 | Explicit `EXT-X-PROGRAM-DATE-TIME` (Unix seconds + fractional) for this segment when it carries a discontinuity. | `[libavformat/hlsenc.c:L91]` |
| `buf` | `char[]` | L93 | Flexible-array tail used to store `filename`, `sub_filename`, and `key_uri` contiguously after the struct header. | `[libavformat/hlsenc.c:L93]` |


## Contract: AVOption Table (Muxer)

Every option the HLS muxer exposes is registered in the `options[]` array at `[libavformat/hlsenc.c:L3121-L3180]`, with the `{ NULL }` array terminator on line 3180. The array holds **58 entries**: 35 distinct user-tunable `AVOption` entries plus 23 `AV_OPT_TYPE_CONST` aliases that expose named values for `hls_segment_type` (2 aliases), `hls_flags` (15 aliases), `hls_playlist_type` (2 aliases), and `hls_start_number_source` (4 aliases).

A port that drops, renames, retypes, or repurposes any option below breaks command-line compatibility (`-hls_<name>`) and binary API compatibility (`av_opt_set*`/`av_opt_get*`).

### Column conventions

- **Option Name**: the literal `name` string passed to the FFmpeg option parser.
- **C Field**: the `HLSContext` member that `OFFSET(field)` resolves to (`#define OFFSET(x) offsetof(HLSContext, x)` at `[libavformat/hlsenc.c:L3119]`); empty for `AV_OPT_TYPE_CONST` aliases.
- **AV_OPT_TYPE**: the option's type as declared in the entry.
- **Default**: the `.i64` / `.dbl` / `.str` default value as declared.
- **Min** / **Max**: the numeric bounds (`INT64_MIN..INT64_MAX`, `INT_MIN..INT_MAX`, `UINT_MAX`, or `0..0` for strings/dicts).
- **Flags**: `E` denotes `AV_OPT_FLAG_ENCODING_PARAM` (defined at `[libavformat/hlsenc.c:L3120]`); all muxer options carry this flag.
- **Unit**: the option-parser grouping identifier; used to associate `AV_OPT_TYPE_CONST` aliases with their parent option.

### Table

| Option Name | C Field | AV_OPT_TYPE | Default | Min | Max | Flags | Unit | Citation |
|-------------|---------|-------------|---------|-----|-----|-------|------|----------|
| `start_number` | `start_sequence` | `INT64` | `0` | `0` | `INT64_MAX` | `E` | — | `[libavformat/hlsenc.c:L3122]` |
| `hls_time` | `time` | `DURATION` | `2000000` (µs) | `0` | `INT64_MAX` | `E` | — | `[libavformat/hlsenc.c:L3123]` |
| `hls_init_time` | `init_time` | `DURATION` | `0` | `0` | `INT64_MAX` | `E` | — | `[libavformat/hlsenc.c:L3124]` |
| `hls_list_size` | `max_nb_segments` | `INT` | `5` | `0` | `INT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3125]` |
| `hls_delete_threshold` | `hls_delete_threshold` | `INT` | `1` | `1` | `INT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3126]` |
| `hls_vtt_options` | `vtt_format_options_str` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3127]` |
| `hls_allow_cache` | `allowcache` | `INT` | `-1` | `INT_MIN` | `INT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3128]` |
| `hls_base_url` | `baseurl` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3129]` |
| `hls_segment_filename` | `segment_filename` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3130]` |
| `hls_segment_options` | `format_options` | `DICT` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3131]` |
| `hls_segment_size` | `max_seg_size` | `INT` | `0` | `0` | `INT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3132]` |
| `hls_key_info_file` | `key_info_file` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3133]` |
| `hls_enc` | `encrypt` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3134]` |
| `hls_enc_key` | `key` | `STRING` | (unset) | (default) | (default) | `E` | — | `[libavformat/hlsenc.c:L3135]` |
| `hls_enc_key_url` | `key_url` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3136]` |
| `hls_enc_iv` | `iv` | `STRING` | (unset) | (default) | (default) | `E` | — | `[libavformat/hlsenc.c:L3137]` |
| `hls_subtitle_path` | `subtitle_filename` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3138]` |
| `hls_segment_type` | `segment_type` | `INT` | `SEGMENT_TYPE_MPEGTS` (0) | `0` | `SEGMENT_TYPE_FMP4` (1) | `E` | `segment_type` | `[libavformat/hlsenc.c:L3139]` |
| `mpegts` | — (const alias) | `CONST` | `SEGMENT_TYPE_MPEGTS` (0) | `0` | `UINT_MAX` | `E` | `segment_type` | `[libavformat/hlsenc.c:L3140]` |
| `fmp4` | — (const alias) | `CONST` | `SEGMENT_TYPE_FMP4` (1) | `0` | `UINT_MAX` | `E` | `segment_type` | `[libavformat/hlsenc.c:L3141]` |
| `hls_fmp4_init_filename` | `fmp4_init_filename` | `STRING` | `"init.mp4"` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3142]` |
| `hls_fmp4_init_resend` | `resend_init_file` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3143]` |
| `hls_flags` | `flags` | `FLAGS` | `0` | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3144]` |
| `single_file` | — (const alias) | `CONST` | `HLS_SINGLE_FILE` (`1 << 0`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3145]` |
| `temp_file` | — (const alias) | `CONST` | `HLS_TEMP_FILE` (`1 << 11`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3146]` |
| `delete_segments` | — (const alias) | `CONST` | `HLS_DELETE_SEGMENTS` (`1 << 1`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3147]` |
| `round_durations` | — (const alias) | `CONST` | `HLS_ROUND_DURATIONS` (`1 << 2`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3148]` |
| `discont_start` | — (const alias) | `CONST` | `HLS_DISCONT_START` (`1 << 3`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3149]` |
| `omit_endlist` | — (const alias) | `CONST` | `HLS_OMIT_ENDLIST` (`1 << 4`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3150]` |
| `split_by_time` | — (const alias) | `CONST` | `HLS_SPLIT_BY_TIME` (`1 << 5`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3151]` |
| `append_list` | — (const alias) | `CONST` | `HLS_APPEND_LIST` (`1 << 6`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3152]` |
| `program_date_time` | — (const alias) | `CONST` | `HLS_PROGRAM_DATE_TIME` (`1 << 7`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3153]` |
| `second_level_segment_index` | — (const alias) | `CONST` | `HLS_SECOND_LEVEL_SEGMENT_INDEX` (`1 << 8`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3154]` |
| `second_level_segment_duration` | — (const alias) | `CONST` | `HLS_SECOND_LEVEL_SEGMENT_DURATION` (`1 << 9`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3155]` |
| `second_level_segment_size` | — (const alias) | `CONST` | `HLS_SECOND_LEVEL_SEGMENT_SIZE` (`1 << 10`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3156]` |
| `periodic_rekey` | — (const alias) | `CONST` | `HLS_PERIODIC_REKEY` (`1 << 12`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3157]` |
| `independent_segments` | — (const alias) | `CONST` | `HLS_INDEPENDENT_SEGMENTS` (`1 << 13`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3158]` |
| `iframes_only` | — (const alias) | `CONST` | `HLS_I_FRAMES_ONLY` (`1 << 14`) | `0` | `UINT_MAX` | `E` | `flags` | `[libavformat/hlsenc.c:L3159]` |
| `strftime` | `use_localtime` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3160]` |
| `strftime_mkdir` | `use_localtime_mkdir` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3161]` |
| `hls_playlist_type` | `pl_type` | `INT` | `PLAYLIST_TYPE_NONE` (0) | `0` | `PLAYLIST_TYPE_NB - 1` (2) | `E` | `pl_type` | `[libavformat/hlsenc.c:L3162]` |
| `event` | — (const alias) | `CONST` | `PLAYLIST_TYPE_EVENT` (1) | `INT_MIN` | `INT_MAX` | `E` | `pl_type` | `[libavformat/hlsenc.c:L3163]` |
| `vod` | — (const alias) | `CONST` | `PLAYLIST_TYPE_VOD` (2) | `INT_MIN` | `INT_MAX` | `E` | `pl_type` | `[libavformat/hlsenc.c:L3164]` |
| `method` | `method` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3165]` |
| `hls_start_number_source` | `start_sequence_source_type` | `INT` | `HLS_START_SEQUENCE_AS_START_NUMBER` (0) | `0` | `HLS_START_SEQUENCE_LAST - 1` (3) | `E` | `start_sequence_source_type` | `[libavformat/hlsenc.c:L3166]` |
| `generic` | — (const alias) | `CONST` | `HLS_START_SEQUENCE_AS_START_NUMBER` (0) | `INT_MIN` | `INT_MAX` | `E` | `start_sequence_source_type` | `[libavformat/hlsenc.c:L3167]` |
| `epoch` | — (const alias) | `CONST` | `HLS_START_SEQUENCE_AS_SECONDS_SINCE_EPOCH` (1) | `INT_MIN` | `INT_MAX` | `E` | `start_sequence_source_type` | `[libavformat/hlsenc.c:L3168]` |
| `epoch_us` | — (const alias) | `CONST` | `HLS_START_SEQUENCE_AS_MICROSECONDS_SINCE_EPOCH` (3) | `INT_MIN` | `INT_MAX` | `E` | `start_sequence_source_type` | `[libavformat/hlsenc.c:L3169]` |
| `datetime` | — (const alias) | `CONST` | `HLS_START_SEQUENCE_AS_FORMATTED_DATETIME` (2) | `INT_MIN` | `INT_MAX` | `E` | `start_sequence_source_type` | `[libavformat/hlsenc.c:L3170]` |
| `http_user_agent` | `user_agent` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3171]` |
| `var_stream_map` | `var_stream_map` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3172]` |
| `cc_stream_map` | `cc_stream_map` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3173]` |
| `master_pl_name` | `master_pl_name` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3174]` |
| `master_pl_publish_rate` | `master_publish_rate` | `INT` | `0` | `0` | `UINT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3175]` |
| `http_persistent` | `http_persistent` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3176]` |
| `timeout` | `timeout` | `DURATION` | `-1` | `-1` | `INT_MAX` | `E` | — | `[libavformat/hlsenc.c:L3177]` |
| `ignore_io_errors` | `ignore_io_errors` | `BOOL` | `0` | `0` | `1` | `E` | — | `[libavformat/hlsenc.c:L3178]` |
| `headers` | `headers` | `STRING` | `NULL` | `0` | `0` | `E` | — | `[libavformat/hlsenc.c:L3179]` |

The 58-row enumeration above matches the source line-for-line: `[libavformat/hlsenc.c:L3122]` through `[libavformat/hlsenc.c:L3179]`. Line `[libavformat/hlsenc.c:L3180]` carries the `{ NULL }` sentinel.


## Contract: Timestamp Unit Conventions

HLS uses several different time bases concurrently. The muxer accumulates per-packet durations in `double` seconds, then rounds to integer seconds when emitting `EXT-X-TARGETDURATION`. The demuxer (when parsing ID3-timestamped streams) interprets timestamps as 1/90000 ticks. Sub-muxer time bases (MPEG-TS or fMP4) are passed through unchanged. Mixing or rebasing these units in a port produces drift — clients have observed playback discontinuities when sub-segment durations are recomputed in a different time base than the original.

| Context | Unit | Constant / Base | Where Used | Citation |
|---------|------|-----------------|------------|----------|
| Segment duration accumulation (muxer) | `double` seconds | accumulated via `dpp` (duration per packet) | `vs->duration += dpp` per reference packet | `[libavformat/hlsenc.c:L2486-L2497]` |
| `EXT-X-TARGETDURATION` emission | integer seconds via `lrint` | — | `target_duration = lrint(en->duration)` over playlist | `[libavformat/hlsenc.c:L1585-L1586]` |
| `EXTINF` rounded form | integer seconds via `lrint(duration)` | — | when `HLS_ROUND_DURATIONS` flag is set | `[libavformat/hlsplaylist.c:L160]` |
| `EXTINF` float form | `double` seconds via `%f` printf | — | default emission path | `[libavformat/hlsplaylist.c:L162]` |
| Segment-cut comparison (muxer) | `AV_TIME_BASE_Q` (1/1,000,000) | `AV_TIME_BASE_Q` | `av_compare_ts(pkt->pts - vs->start_pts, st->time_base, end_pts, AV_TIME_BASE_Q)` | `[libavformat/hlsenc.c:L2501-L2502]` |
| Demuxer ID3-timestamped time base | 1/90000 (with 33-bit wrap) | `MPEG_TIME_BASE 90000` | `MPEG_TIME_BASE_Q = {1, 90000}` | `[libavformat/hls.c:L56-L57]` |
| Sub-muxer packet time base (passthrough) | inherited from each source stream's `AVStream::time_base` | per-stream | sub-muxer (MPEG-TS or fMP4) writes timestamps in this base | sub-muxer-defined; no rebasing in HLS [inferred — no direct source] |
| `hls_time` AVOption default | microseconds (`AV_OPT_TYPE_DURATION`) | `2,000,000` (2 seconds) | parsed by option layer; stored in `HLSContext::time` | `[libavformat/hlsenc.c:L3123]` |
| `hls_init_time` AVOption default | microseconds (`AV_OPT_TYPE_DURATION`) | `0` (disabled) | parsed by option layer; stored in `HLSContext::init_time` | `[libavformat/hlsenc.c:L3124]` |
| `timeout` AVOption default | microseconds (`AV_OPT_TYPE_DURATION`) | `-1` (no timeout) | parsed by option layer; stored in `HLSContext::timeout` | `[libavformat/hlsenc.c:L3177]` |
| `EXT-X-PROGRAM-DATE-TIME` emission | Unix seconds + 3-digit milliseconds | `%s.%03d%s\n` (ISO-8601 + tz) | `ff_hls_write_file_entry` when `prog_date_time` is non-NULL | `[libavformat/hlsplaylist.c:L191]` |
| Closed-captions instream ID | text — `CC1`..`CC4` / `SERVICE1`..`SERVICE63` | — | `EXT-X-MEDIA:INSTREAM-ID` value in master playlist | passed-through user input [inferred — no direct source] |

Critical observation: the segment-cut decision at `[libavformat/hlsenc.c:L2501-L2502]` mixes two time bases — `pkt->pts` is in the reference stream's `time_base`, while `end_pts` is in `AV_TIME_BASE_Q`. The mismatch is resolved by `av_compare_ts`, which performs the rebasing internally. A port that drops the `av_compare_ts` indirection and compares the two integers directly will produce non-monotonic segment durations.

## Contract: Codec Extradata Format

HLS requires that every encoded stream's codec configuration bytes (e.g., H.264 SPS/PPS, HEVC VPS/SPS/PPS, AAC `AudioSpecificConfig`) be present in `AVCodecParameters::extradata` and `extradata_size` rather than inlined into the elementary packet stream.

| Aspect | Contract | Citation |
|--------|----------|----------|
| Flag asserted by muxer | `AVFMT_GLOBALHEADER` on the parent format (`ff_hls_muxer.p.flags`) | `[libavformat/hlsenc.c:L3198]` |
| Behavioral effect | The flag is propagated to the lavf core, which sets `AV_CODEC_FLAG_GLOBAL_HEADER` on encoder contexts and demands `AVCodecParameters::extradata` to be populated before `avformat_write_header`. | `AVFMT_GLOBALHEADER` doc in `[libavformat/avformat.h]` [inferred — no direct source] |
| Sub-muxer consumption (TS) | The MPEG-TS muxer reads `extradata` when emitting PMT entries and patches the elementary stream as needed. | sub-muxer-defined; HLS does not modify extradata [inferred — no direct source] |
| Sub-muxer consumption (fMP4) | The fMP4 muxer reads `extradata` to build `avcC` (H.264), `hvcC` (HEVC), `esds`/`mp4a` (AAC), and other codec-specific descriptors inside `moov`/`stsd`/`stbl`. Missing extradata produces structurally invalid fMP4. | sub-muxer-defined [inferred — no direct source] |
| Failure mode | An encoder that emits inline headers but leaves `extradata` empty produces playable MPEG-TS segments (decoders parse SPS/PPS from the bitstream) but invalid fMP4 segments (missing `avcC`). | observed behavior [inferred — no direct source] |

A port that changes the `AVFMT_GLOBALHEADER` declaration silently breaks fMP4 output for callers that rely on the encoder being instructed to produce extradata.

## Contract: Byterange Offset/Size Wire Format

When the HLS muxer operates in single-file mode (`HLS_SINGLE_FILE`) or with a non-zero `hls_segment_size` (`max_seg_size > 0`), each segment is identified by a byte offset and length within a single shared file. The playlist emits one `EXT-X-BYTERANGE` line per file entry, immediately after the corresponding `EXTINF` line.

| Aspect | Contract | Citation |
|--------|----------|----------|
| In-memory representation | `HLSSegment::pos` and `HLSSegment::size`, both `int64_t` | `[libavformat/hlsenc.c:L81-L82]` |
| Wire-format format string | `"#EXT-X-BYTERANGE:%"PRId64"@%"PRId64"\n"` (size before `@`, offset after) | `[libavformat/hlsplaylist.c:L164-L165]` |
| Emission gate | `byterange_mode` is true when `HLS_SINGLE_FILE` OR `max_seg_size > 0` | `[libavformat/hlsenc.c:L1549]` (`hls_window`); `[libavformat/hlsenc.c:L2504]` (`hls_write_packet`) |
| I-frames-only variant | When `iframe_mode` is set, `keyframe_size` and `keyframe_pos` replace `size` and `pos` respectively | `[libavformat/hlsplaylist.c:L164-L165]` |
| Forces HLS version | `byterange_mode` raises `EXT-X-VERSION` to at least 4 | `[libavformat/hlsenc.c:L1556-L1559]` |
| Sequence reset | When `byterange_mode` engages, the first emission resets `EXT-X-MEDIA-SEQUENCE` to `0` | `[libavformat/hlsenc.c:L1556-L1559]` |
| Terminator | Single LF (`\n`); no trailing whitespace | `[libavformat/hlsplaylist.c:L164-L165]` |
| `EXT-X-MAP` companion (fMP4 only) | `EXT-X-MAP` may carry its own `BYTERANGE` attribute in the form `,BYTERANGE="<size>@<offset>"` when fMP4 init shares the segment file | `[libavformat/hlsplaylist.c:L137-L141]` |

A port that swaps the `size` and `offset` order, or that drops the `int64_t` width, breaks any HLS-compliant client (`PRId64` ensures portable 64-bit width regardless of host `int`/`long` size).

## Contract: Encryption IV Derivation

AES-128-CBC requires a 16-byte initialization vector. The HLS muxer derives the IV via one of four paths, ordered by precedence. The result is hex-encoded into 32 ASCII characters (plus NUL) and emitted as `IV=0x<hex>` on the `EXT-X-KEY` line when non-empty.

| Path | Trigger | Derivation | Citation |
|------|---------|------------|----------|
| 1. `hls_key_info_file` line 3 | `key_info_file` is set AND the file's third line is non-empty | The hex string on line 3 of the 3-line key-info file is copied verbatim into `vs->iv_string`. | `[libavformat/hlsenc.c:L737-L738]` |
| 2. `hls_enc_iv` AVOption | `hls_enc_iv` is set AND `iv_string` is currently empty | The supplied bytes (16 bytes) are copied into the IV buffer, then hex-encoded. | `[libavformat/hlsenc.c:L666-L676]` (via `hls->iv` at L673) |
| 3. Sequence-number derived | No user-supplied IV present, `iv_string` is empty | `iv[0..7] = 0x00`, then `iv[8..15] = big-endian vs->sequence via AV_WB64`. The 16-byte IV is then hex-encoded to `iv_string`. | `[libavformat/hlsenc.c:L666-L676]` |
| 4. Inherited from previous segment | `iv_string` is already populated from a prior derivation in this session | The cached `iv_string` is reused. | `[libavformat/hlsenc.c:L666]` (gate on `if (!*hls->iv_string)`) |

### KEYSIZE and IV size

| Symbol | Value | Citation |
|--------|-------|----------|
| `KEYSIZE` | `16` (bytes) | `[libavformat/hlsenc.c:L70]` |
| Hex-encoded length | `KEYSIZE * 2` chars + NUL = 33 chars | `[libavformat/hlsenc.c:L88]` (`HLSSegment::iv_string`) |
| `IV=0x` wire prefix | `,IV=0x%s` (printf format) | `[libavformat/hlsenc.c:L1604-L1605]` |
| Wire emission gate | Emitted only when `iv_string` is non-empty (`*en->iv_string`) | `[libavformat/hlsenc.c:L1604]` |

### Key derivation (companion)

| Path | Trigger | Derivation | Citation |
|------|---------|------------|----------|
| User-supplied via `hls_enc_key` | `hls->key` is set | `memcpy(key, hls->key, KEYSIZE)` | `[libavformat/hlsenc.c:L696-L698]` |
| User-supplied via `hls_key_info_file` | `key_info_file` is set | The file's second line names a key file; 16 bytes are read via `avio_read`. | `[libavformat/hlsenc.c:L734-L735, L760-L761]` |
| Auto-generated | Neither `hls->key` nor `key_info_file` set | `av_random_bytes(key, KEYSIZE)` produces 16 cryptographically strong random bytes; the key is written to `<output_url>.key` on disk. | `[libavformat/hlsenc.c:L691-L709]` |

A port that changes the IV-derivation algorithm (e.g., uses little-endian sequence-number encoding, or hashes the sequence rather than embedding it raw) breaks decryption of any existing recordings produced by this muxer.


## Contract: HLSCryptoContext Binary Layout

`HLSCryptoContext` is the demuxer-side in-memory carrier for AES-128 / Sample-AES key material. One instance is embedded in the demuxer `HLSContext` at `[libavformat/hls.c:L235]`; per-playlist context (for variant-scoped keys) is stored elsewhere as required. The struct definition spans `[libavformat/hls_sample_encryption.h:L43-L47]`.

A port must preserve the byte ordering and exact 16-byte width of `key` and `iv` to ensure interoperability with `libavutil/aes.h`'s AES-128 implementation, which is the consumer of these buffers.

| Offset | Field | C Type | Size | Notes | Citation |
|--------|-------|--------|------|-------|----------|
| `0` | `aes_ctx` | `struct AVAES *` | `sizeof(void *)` (8 bytes on 64-bit, 4 on 32-bit) | Pointer to the AES context allocated by `av_aes_alloc` (defined at `[libavutil/aes.h:L41]`); initialized via `av_aes_init` (at `[libavutil/aes.h:L51]`). | `[libavformat/hls_sample_encryption.h:L44]` |
| `sizeof(void *)` | `key[16]` | `uint8_t[16]` | 16 bytes | Raw 128-bit symmetric key. `KEYSIZE 16` defined at `[libavformat/hlsenc.c:L70]`. Decoded from the network-fetched key file (full-segment AES-128) or the per-stream Sample-AES negotiation. | `[libavformat/hls_sample_encryption.h:L45]` |
| `sizeof(void *) + 16` | `iv[16]` | `uint8_t[16]` | 16 bytes | Raw 128-bit initialization vector. For full-segment AES-128, derived from the `IV` attribute on `EXT-X-KEY` or from the sequence number when absent. | `[libavformat/hls_sample_encryption.h:L46]` |

Total fixed size: `sizeof(void *) + 32` bytes. Struct alignment may add trailing padding to the `sizeof(void *)` boundary; the layout is treated as opaque by callers — they access fields only by name, never by offset.

## Contract: HLSAudioSetupInfo Binary Layout

`HLSAudioSetupInfo` is the companion struct for HLS Sample Encryption audio streams. It carries codec-specific decoder-config bytes parsed from a TS payload's setup-info section. One instance is embedded per `playlist` at `[libavformat/hls.c:L159]`. The struct definition spans `[libavformat/hls_sample_encryption.h:L49-L56]`.

| Offset | Field | C Type | Size | Notes | Citation |
|--------|-------|--------|------|-------|----------|
| `0` | `codec_id` | `enum AVCodecID` | `sizeof(int)` (typically 4 bytes) | Resolved codec identifier (AAC, AC3, EAC3) for the audio stream associated with this setup info. | `[libavformat/hls_sample_encryption.h:L50]` |
| `sizeof(enum) + padding` | `codec_tag` | `uint32_t` | 4 bytes | FourCC-style codec tag used for stream-type negotiation. | `[libavformat/hls_sample_encryption.h:L51]` |
| (per alignment) | `priming` | `uint16_t` | 2 bytes | AAC priming sample count (number of samples to discard from decoder output). | `[libavformat/hls_sample_encryption.h:L52]` |
| (per alignment) | `version` | `uint8_t` | 1 byte | Version field from the wire setup-info record. | `[libavformat/hls_sample_encryption.h:L53]` |
| (per alignment) | `setup_data_length` | `uint8_t` | 1 byte | Number of valid bytes in `setup_data[]` (0..`HLS_MAX_AUDIO_SETUP_DATA_LEN`). | `[libavformat/hls_sample_encryption.h:L54]` |
| (per alignment) | `setup_data[HLS_MAX_AUDIO_SETUP_DATA_LEN + AV_INPUT_BUFFER_PADDING_SIZE]` | `uint8_t[]` | `10 + AV_INPUT_BUFFER_PADDING_SIZE` bytes | Codec-specific decoder configuration (AAC `AudioSpecificConfig`-equivalent or AC3 `bsi` bytes) plus FFmpeg input-buffer padding. | `[libavformat/hls_sample_encryption.h:L55]` |

### Companion constants

| Symbol | Value | Citation |
|--------|-------|----------|
| `HLS_MAX_ID3_TAGS_DATA_LEN` | `138` | `[libavformat/hls_sample_encryption.h:L40]` |
| `HLS_MAX_AUDIO_SETUP_DATA_LEN` | `10` | `[libavformat/hls_sample_encryption.h:L41]` |

A port that changes `HLS_MAX_AUDIO_SETUP_DATA_LEN` breaks any caller that has serialized the struct to persistent storage (e.g., for caching purposes) — the offset of subsequent struct members shifts.

## Contract: STREAM_TYPE_HLS_SE_* Constants

HLS Sample Encryption introduces four MPEG-TS stream-type values that carry the same payload as their non-encrypted counterparts but with frame-level AES-128 encryption applied. A Sample-Encryption-capable demuxer must recognize these values; a port that omits them silently fails on encrypted streams. The defining header comments cite the Apple Developer specification at `[libavformat/mpegts.h:L174-L176]` ("MPEG-2 Stream Encryption Format for HTTP Live Streaming").

| Constant | Value | Underlying Codec | Citation |
|----------|-------|------------------|----------|
| `STREAM_TYPE_HLS_SE_VIDEO_H264` | `0xdb` | H.264 (with Sample-AES) | `[libavformat/mpegts.h:L177]` |
| `STREAM_TYPE_HLS_SE_AUDIO_AAC` | `0xcf` | AAC (with Sample-AES) | `[libavformat/mpegts.h:L178]` |
| `STREAM_TYPE_HLS_SE_AUDIO_AC3` | `0xc1` | AC-3 (with Sample-AES) | `[libavformat/mpegts.h:L179]` |
| `STREAM_TYPE_HLS_SE_AUDIO_EAC3` | `0xc2` | E-AC-3 (with Sample-AES) | `[libavformat/mpegts.h:L180]` |

The MPEG-TS PMT loop in the parent demuxer matches these `stream_type` values, sets `AVStream::codecpar->codec_id` to the corresponding non-encrypted codec, and routes frames through `ff_hls_senc_decrypt_frame` (declared at `[libavformat/hls_sample_encryption.h:L63]`) before forwarding.

## Contract: FFOutputFormat Field Assignments (Muxer Registration)

The `ff_hls_muxer` symbol is the registration record consumed by `libavformat`'s format-lookup tables. It encodes the muxer's name, supported codecs, flags, private-data size, and the five lifecycle callbacks. The complete record spans `[libavformat/hlsenc.c:L3191-L3207]`. Every field is part of the public API contract — changing any value alters the muxer's discovery, defaults, behavior, or memory layout in a way that downstream callers observe.

| Field | Value | Effect | Citation |
|-------|-------|--------|----------|
| `.p.name` | `"hls"` | Format name used by `av_guess_format("hls", ...)`, `-f hls` CLI, and `avformat_alloc_output_context2`. | `[libavformat/hlsenc.c:L3192]` |
| `.p.long_name` | `NULL_IF_CONFIG_SMALL("Apple HTTP Live Streaming")` | Human-readable description shown in `-formats`. | `[libavformat/hlsenc.c:L3193]` |
| `.p.extensions` | `"m3u8"` | Auto-detection extension for output filename probing. | `[libavformat/hlsenc.c:L3194]` |
| `.p.audio_codec` | `AV_CODEC_ID_AAC` | Default audio codec selected when callers don't specify one. | `[libavformat/hlsenc.c:L3195]` |
| `.p.video_codec` | `AV_CODEC_ID_H264` | Default video codec selected when callers don't specify one. | `[libavformat/hlsenc.c:L3196]` |
| `.p.subtitle_codec` | `AV_CODEC_ID_WEBVTT` | Default subtitle codec. | `[libavformat/hlsenc.c:L3197]` |
| `.p.flags` | `AVFMT_NOFILE \| AVFMT_GLOBALHEADER \| AVFMT_NODIMENSIONS` | `NOFILE`: caller does not open a top-level file; muxer manages segment files itself. `GLOBALHEADER`: encoder must produce extradata. `NODIMENSIONS`: streams without explicit dimensions are accepted. | `[libavformat/hlsenc.c:L3198]` |
| `.p.priv_class` | `&hls_class` | `AVClass` for `AVOption` introspection (defined at `[libavformat/hlsenc.c:L3183-L3188]`). | `[libavformat/hlsenc.c:L3199]` |
| `.flags_internal` | `FF_OFMT_FLAG_ALLOW_FLUSH` | Internal flag permitting `av_write_frame(..., NULL)` flush operations. | `[libavformat/hlsenc.c:L3200]` |
| `.priv_data_size` | `sizeof(HLSContext)` | Bytes the caller (`avformat_alloc_output_context2`) allocates for `priv_data`. | `[libavformat/hlsenc.c:L3201]` |
| `.init` | `hls_init` | Phase-1 callback: variant allocation, segment-filename validation, options parsing. | `[libavformat/hlsenc.c:L3202]` |
| `.write_header` | `hls_write_header` | Phase-2 callback: per-variant `avformat_write_header` on the child sub-muxer, codec-attribute computation. | `[libavformat/hlsenc.c:L3203]` |
| `.write_packet` | `hls_write_packet` | Phase-3 callback: segment-cut decision, segment finalize, playlist publish. | `[libavformat/hlsenc.c:L3204]` |
| `.write_trailer` | `hls_write_trailer` | Phase-4 callback: final flush per variant, final playlist publish. | `[libavformat/hlsenc.c:L3205]` |
| `.deinit` | `hls_deinit` | Phase-5 callback: free all per-session allocations. | `[libavformat/hlsenc.c:L3206]` |

## Contract: FFInputFormat Field Assignments (Demuxer Registration)

The `ff_hls_demuxer` symbol mirrors the muxer registration on the demuxer side. The complete record spans `[libavformat/hls.c:L2900-L2912]`. Like the muxer registration, every field is part of the public API contract.

| Field | Value | Effect | Citation |
|-------|-------|--------|----------|
| `.p.name` | `"hls"` | Format name used by `av_find_input_format("hls")`, `-f hls` CLI, and probing infrastructure. | `[libavformat/hls.c:L2901]` |
| `.p.long_name` | `NULL_IF_CONFIG_SMALL("Apple HTTP Live Streaming")` | Human-readable description shown in `-formats`. | `[libavformat/hls.c:L2902]` |
| `.p.priv_class` | `&hls_class` | `AVClass` for demuxer-side `AVOption` introspection. | `[libavformat/hls.c:L2903]` |
| `.p.flags` | `AVFMT_NOGENSEARCH \| AVFMT_TS_DISCONT \| AVFMT_NO_BYTE_SEEK \| AVFMT_SHOW_IDS` | `NOGENSEARCH`: no generic search (URL-driven seek only). `TS_DISCONT`: timestamps may have discontinuities. `NO_BYTE_SEEK`: byte-position seek unsupported. `SHOW_IDS`: stream IDs are user-visible. | `[libavformat/hls.c:L2904]` |
| `.priv_data_size` | `sizeof(HLSContext)` | Bytes the caller (`avformat_open_input`) allocates for `priv_data` (note: this is the demuxer-side `HLSContext`, not the muxer-side). | `[libavformat/hls.c:L2905]` |
| `.flags_internal` | `FF_INFMT_FLAG_INIT_CLEANUP` | Internal flag instructing lavf to call `read_close` even when `read_header` fails. | `[libavformat/hls.c:L2906]` |
| `.read_probe` | `hls_probe` | Probe callback: scans the first few KiB of the input for `#EXTM3U` signature and HLS-specific tags. | `[libavformat/hls.c:L2907]` |
| `.read_header` | `hls_read_header` | Open callback: parses the master/media playlist and allocates the variant/playlist/rendition trees. | `[libavformat/hls.c:L2908]` |
| `.read_packet` | `hls_read_packet` | Packet-fetch callback: drives segment download and sub-demuxer dispatch. | `[libavformat/hls.c:L2909]` |
| `.read_close` | `hls_close` | Close callback: frees all per-session allocations. | `[libavformat/hls.c:L2910]` |
| `.read_seek` | `hls_read_seek` | Seek callback: re-positions the playlist cursor and discards in-flight sub-demuxer state. | `[libavformat/hls.c:L2911]` |

## Contract: HLSFlags Bit Values

The `hls_flags` `AVOption` is a `AV_OPT_TYPE_FLAGS` field that accepts a bitwise OR of named values. Each bit-position is contracted — a port that swaps two flag values (e.g., assigns `HLS_SINGLE_FILE` to bit 1 and `HLS_DELETE_SEGMENTS` to bit 0) would silently corrupt any caller passing flags as raw integers via `av_opt_set_int`. The enum definition spans `[libavformat/hlsenc.c:L96-L113]`.

| Symbol | Bit | Value (decimal) | Effect | Citation |
|--------|-----|-----------------|--------|----------|
| `HLS_SINGLE_FILE` | 0 | `1` | Write all segments into a single file with `EXT-X-BYTERANGE` references. | `[libavformat/hlsenc.c:L98]` |
| `HLS_DELETE_SEGMENTS` | 1 | `2` | Issue HTTP DELETE for segments evicted from the sliding window. | `[libavformat/hlsenc.c:L99]` |
| `HLS_ROUND_DURATIONS` | 2 | `4` | Round `EXTINF` durations to integer seconds (`lrint`). | `[libavformat/hlsenc.c:L100]` |
| `HLS_DISCONT_START` | 3 | `8` | Emit `EXT-X-DISCONTINUITY` immediately after `EXT-X-MEDIA-SEQUENCE` at session start. | `[libavformat/hlsenc.c:L101]` |
| `HLS_OMIT_ENDLIST` | 4 | `16` | Suppress the `EXT-X-ENDLIST` tag on `hls_write_trailer`. | `[libavformat/hlsenc.c:L102]` |
| `HLS_SPLIT_BY_TIME` | 5 | `32` | Permit segment-cut at non-keyframe boundaries when the duration target is reached. | `[libavformat/hlsenc.c:L103]` |
| `HLS_APPEND_LIST` | 6 | `64` | Append to an existing playlist (read prior `EXT-X-MEDIA-SEQUENCE` and continue). | `[libavformat/hlsenc.c:L104]` |
| `HLS_PROGRAM_DATE_TIME` | 7 | `128` | Emit `EXT-X-PROGRAM-DATE-TIME` before each segment entry. | `[libavformat/hlsenc.c:L105]` |
| `HLS_SECOND_LEVEL_SEGMENT_INDEX` | 8 | `256` | Enable `%d` index substitution in `strftime` filename template. | `[libavformat/hlsenc.c:L106]` |
| `HLS_SECOND_LEVEL_SEGMENT_DURATION` | 9 | `512` | Enable `%t` duration substitution in `strftime` filename template. | `[libavformat/hlsenc.c:L107]` |
| `HLS_SECOND_LEVEL_SEGMENT_SIZE` | 10 | `1024` | Enable `%s` size substitution in `strftime` filename template. | `[libavformat/hlsenc.c:L108]` |
| `HLS_TEMP_FILE` | 11 | `2048` | Write segments to `<name>.tmp`, then atomically rename to `<name>` on close. | `[libavformat/hlsenc.c:L109]` |
| `HLS_PERIODIC_REKEY` | 12 | `4096` | Re-read `hls_key_info_file` on every segment cycle (rotating-key workflow). | `[libavformat/hlsenc.c:L110]` |
| `HLS_INDEPENDENT_SEGMENTS` | 13 | `8192` | Emit `EXT-X-INDEPENDENT-SEGMENTS` when the variant has video. | `[libavformat/hlsenc.c:L111]` |
| `HLS_I_FRAMES_ONLY` | 14 | `16384` | Emit `EXT-X-I-FRAMES-ONLY`; playlist file entries use keyframe-only byteranges; forces version 4. | `[libavformat/hlsenc.c:L112]` |

Bit 15 is unused at commit `566ad786`. The set `{0..14}` covers all 15 currently-defined values.


## Contract: SegmentType Enum Values

The `hls_segment_type` `AVOption` selects which container format the muxer uses for segment payloads. The enum definition spans `[libavformat/hlsenc.c:L115-L118]`.

| Symbol | Value | Effect | Citation |
|--------|-------|--------|----------|
| `SEGMENT_TYPE_MPEGTS` | `0` | Segments are MPEG-TS files (`.ts`); compatible with HLS version 3 (default) plus version-bump rules below. | `[libavformat/hlsenc.c:L116]` |
| `SEGMENT_TYPE_FMP4` | `1` | Segments are fragmented MP4 files (`.m4s`); `EXT-X-MAP` references the `init.mp4` initialization box; forces `EXT-X-VERSION:7`. | `[libavformat/hlsenc.c:L117]` |

The forced version bump for fMP4 is implemented at `[libavformat/hlsenc.c:L1569-L1571]` inside `hls_window`: when `segment_type == SEGMENT_TYPE_FMP4`, the computed `version` is set to 7 regardless of other feature flags.

## Contract: StartSequenceSourceType Enum Values

The `hls_start_number_source` `AVOption` selects the policy for seeding the first `EXT-X-MEDIA-SEQUENCE` value. The enum definition spans `[libavformat/hlsenc.c:L57-L63]`.

| Symbol | Value | Behavior | Citation |
|--------|-------|----------|----------|
| `HLS_START_SEQUENCE_AS_START_NUMBER` | `0` | Use the literal `start_number` `AVOption` value. | `[libavformat/hlsenc.c:L58]` |
| `HLS_START_SEQUENCE_AS_SECONDS_SINCE_EPOCH` | `1` | Use Unix time at session start, in whole seconds. | `[libavformat/hlsenc.c:L59]` |
| `HLS_START_SEQUENCE_AS_FORMATTED_DATETIME` | `2` | Use the `YYYYMMDDhhmmss` 14-digit datetime as the sequence (does not overflow `int64_t`). | `[libavformat/hlsenc.c:L60]` |
| `HLS_START_SEQUENCE_AS_MICROSECONDS_SINCE_EPOCH` | `3` | Use Unix time at session start, in microseconds. | `[libavformat/hlsenc.c:L61]` |
| `HLS_START_SEQUENCE_LAST` | `4` | Sentinel value used to bound the option's max (`HLS_START_SEQUENCE_LAST - 1`); not user-settable. | `[libavformat/hlsenc.c:L62]` |

The `hls_start_number_source` `AVOption` exposes the first four values via `AV_OPT_TYPE_CONST` aliases (`generic`, `epoch`, `epoch_us`, `datetime`) at `[libavformat/hlsenc.c:L3167-L3170]`. Note that the alias names do not match the enum tail tokens exactly (e.g., `generic` ↔ `HLS_START_SEQUENCE_AS_START_NUMBER`) — a port must preserve both sets of names.

## Contract: PlaylistType Enum Values (Muxer-side)

The muxer-side `PlaylistType` enum drives `EXT-X-PLAYLIST-TYPE` emission and gates several downstream behaviors (e.g., `EVENT` permits later additions but no rewrites; `VOD` mandates a final `EXT-X-ENDLIST`). The enum definition spans `[libavformat/hlsplaylist.h:L31-L36]`.

| Symbol | Value | Effect | Citation |
|--------|-------|--------|----------|
| `PLAYLIST_TYPE_NONE` | `0` | Live mode (default); no `EXT-X-PLAYLIST-TYPE` tag emitted. | `[libavformat/hlsplaylist.h:L32]` |
| `PLAYLIST_TYPE_EVENT` | `1` | Event mode; emits `#EXT-X-PLAYLIST-TYPE:EVENT`; the playlist may grow but existing entries must not be removed or rewritten. | `[libavformat/hlsplaylist.h:L33]` |
| `PLAYLIST_TYPE_VOD` | `2` | VOD mode; emits `#EXT-X-PLAYLIST-TYPE:VOD`; the playlist is finalized on `hls_write_trailer` with `EXT-X-ENDLIST` (unless `HLS_OMIT_ENDLIST` is set). | `[libavformat/hlsplaylist.h:L34]` |
| `PLAYLIST_TYPE_NB` | `3` | Sentinel used as the option's upper bound (`PLAYLIST_TYPE_NB - 1`); not user-settable. | `[libavformat/hlsplaylist.h:L35]` |

The `hls_playlist_type` `AVOption` exposes the EVENT and VOD values via `AV_OPT_TYPE_CONST` aliases at `[libavformat/hlsenc.c:L3163-L3164]`; the NONE value is the default and has no alias.

Note: The demuxer-side has a separately-defined `PlaylistType` enum with **different symbol names** (`PLS_TYPE_UNSPECIFIED`, `PLS_TYPE_EVENT`, `PLS_TYPE_VOD`) at `[libavformat/hls.c:L91-L95]`. The two enums are conceptually parallel but the type names are not interchangeable — a port that conflates the two breaks the muxer/demuxer code-sharing boundary.

## Contract: KeyType Enum Values (Demuxer-side)

The demuxer-side `KeyType` enum classifies the encryption applied to a segment, populated from the playlist's `EXT-X-KEY:METHOD=` attribute. Each `segment` struct holds an instance at `key_type` `[libavformat/hls.c:L83]`. The enum definition spans `[libavformat/hls.c:L71-L75]`.

| Symbol | Value | Behavior | Citation |
|--------|-------|----------|----------|
| `KEY_NONE` | `0` | No encryption applied; segment payload is consumed directly. Source of value: implicit (first enumerator). | `[libavformat/hls.c:L72]` |
| `KEY_AES_128` | `1` | Full-segment AES-128-CBC encryption; the entire segment payload is decrypted via the `crypto:` protocol layer before being demuxed. | `[libavformat/hls.c:L73]` |
| `KEY_SAMPLE_AES` | `2` | Per-frame Sample-AES encryption (Apple Sample Encryption); decrypted on a per-NAL/per-frame basis after demux but before passing to the codec. | `[libavformat/hls.c:L74]` |

The `EXT-X-KEY:METHOD=AES-128` and `EXT-X-KEY:METHOD=SAMPLE-AES` strings in the playlist parser map directly to `KEY_AES_128` and `KEY_SAMPLE_AES`. A method string not in this set produces a parse error.

## Cross-References

This document is the **contract reference** — it lists what every value MUST be, sized to, and shaped like. Companion documents in the same Layer 3 set:

| Companion Document | Purpose |
|--------------------|---------|
| [`./functional-invariants.md`](./functional-invariants.md) | MUST/MUST NOT statements that constrain when these contracts apply and how they compose. |
| [`./integration-contracts.md`](./integration-contracts.md) | External-system contracts (HTTP, key fetch, init segment delivery) that consume these data shapes. |
| [`./timing-dependencies.md`](./timing-dependencies.md) | Ordering constraints under which these data values are produced and consumed. |

Layer 2 documents that provide narrative context for the same data:

| Companion Document | Purpose |
|--------------------|---------|
| [`../technical/data-model.md`](../technical/data-model.md) | Field-by-field business-meaning dictionary (this document is the contract; data-model.md is the narrative). |
| [`../technical/codec-logic.md`](../technical/codec-logic.md) | Decision tables that resolve which contract value applies under which input conditions. |
| [`../technical/pipeline-orchestration.md`](../technical/pipeline-orchestration.md) | Lifecycle order in which `hls_init` / `hls_write_header` / `hls_write_packet` / `hls_write_trailer` / `hls_deinit` produce and consume these values. |

Reference back to the master index: [`../README.md`](../README.md).

## Validation Checklist

The following checklist is used by reviewers and by the documentation author before declaring this document complete. Every box must be checkable against the source at commit `566ad786`.

### Structural

- [x] Title `# Data Contracts — HLS Pipeline API Contracts` on line 1.
- [x] Commit-anchor banner immediately after the title.
- [x] `## Overview` section present.
- [x] 18 `## Contract:` H2 sections in the order specified in the agent prompt.
- [x] `## Cross-References` section present.
- [x] `## Validation Checklist` (this section) present at the end.

### Content / No-Summarizing

- [x] Every `HLSContext` (muxer) field at `[libavformat/hlsenc.c:L202-L267]` has its own table row.
- [x] Every `VariantStream` field at `[libavformat/hlsenc.c:L120-L194]` has its own table row.
- [x] Every `HLSSegment` field at `[libavformat/hlsenc.c:L76-L94]` has its own table row.
- [x] Every `AVOption` (58 total: 35 distinct + 23 const aliases) at `[libavformat/hlsenc.c:L3122-L3179]` has its own table row.
- [x] All 15 `HLSFlags` bit values at `[libavformat/hlsenc.c:L98-L112]` enumerated with bit positions.
- [x] All 4 `STREAM_TYPE_HLS_SE_*` constants at `[libavformat/mpegts.h:L177-L180]` present with hex values.
- [x] All 5 `StartSequenceSourceType` values at `[libavformat/hlsenc.c:L58-L62]` enumerated (4 user-visible + 1 sentinel).
- [x] All 3 demuxer `KeyType` values at `[libavformat/hls.c:L72-L74]` enumerated.
- [x] All 4 muxer-side `PlaylistType` values at `[libavformat/hlsplaylist.h:L32-L35]` enumerated (3 user-visible + 1 sentinel).
- [x] 2 `SegmentType` values at `[libavformat/hlsenc.c:L116-L117]` enumerated.
- [x] `FFOutputFormat` 15-field table present at `[libavformat/hlsenc.c:L3192-L3206]` with line citations.
- [x] `FFInputFormat` 11-field table present at `[libavformat/hls.c:L2901-L2911]` with line citations.
- [x] `HLSCryptoContext` binary layout table present (3 fields).
- [x] `HLSAudioSetupInfo` binary layout table present (6 fields).
- [x] Encryption IV derivation has 4 documented paths.
- [x] Timestamp unit conventions table covers segment-cut, EXTINF, TARGETDURATION, PROGRAM-DATE-TIME, demuxer ID3 base, and all `DURATION`-typed AVOptions.
- [x] Byterange wire-format table includes `%"PRId64"@%"PRId64"` format string verbatim.
- [x] No "see X", "and similar", or "etc." groupings replace per-field rows.

### Citation Accuracy

- [x] Every citation uses `[<path>:L<n>]` or `[<path>:L<n>-L<m>]`.
- [x] Line numbers match the verified anchors retrieved during discovery.
- [x] Inferred claims that cannot be directly anchored carry the `[inferred — no direct source]` tag.

### Reference Verification (Spot Checks)

The following spot checks have been validated by direct source comparison:

- [x] `hls_time` `AV_OPT_TYPE_DURATION` default `2,000,000` at `[libavformat/hlsenc.c:L3123]` — matches.
- [x] `hls_list_size` `AV_OPT_TYPE_INT` default `5` at `[libavformat/hlsenc.c:L3125]` — matches.
- [x] `hls_segment_type` const aliases `mpegts` / `fmp4` at `[libavformat/hlsenc.c:L3140-L3141]` — matches.
- [x] `hls_flags` const aliases: `single_file` through `iframes_only` (15 aliases) at `[libavformat/hlsenc.c:L3145-L3159]` — matches.
- [x] `hls_start_number_source` const aliases: `generic` / `epoch` / `epoch_us` / `datetime` (4 aliases) at `[libavformat/hlsenc.c:L3167-L3170]` — matches.
- [x] `HLS_I_FRAMES_ONLY = (1 << 14)` at `[libavformat/hlsenc.c:L112]` — matches.
- [x] `HLS_SINGLE_FILE = (1 << 0)` at `[libavformat/hlsenc.c:L98]` — matches.
- [x] `STREAM_TYPE_HLS_SE_VIDEO_H264 = 0xdb` at `[libavformat/mpegts.h:L177]` — matches.
- [x] `KEYSIZE = 16` at `[libavformat/hlsenc.c:L70]` — matches.
- [x] `MPEG_TIME_BASE = 90000` at `[libavformat/hls.c:L56]` — matches.

### Style

- [x] Tables use pipe-delimited markdown with header and separator rows.
- [x] LF line endings, UTF-8 without BOM.
- [x] Third-person impersonal, present tense throughout.
- [x] No marketing adjectives ("robust", "powerful", "seamless") anywhere.
- [x] No Mermaid diagrams (this document is reference-only).

