# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to produce a **complete three-layer reverse-engineering documentation set** for the FFmpeg HLS (Apple HTTP Live Streaming) muxer/demuxer pipeline. The output is thirteen markdown files plus a master index, written to a brand-new directory tree at `/docs/hls-pipeline/`, all source references anchored to commit `566ad786` (verified as the current `HEAD` at `[Repository:HEAD]`). The audience is mixed: library integrators who need plain-language understanding of what the system produces, and engineers planning to refactor, extend, or port the subsystem who need engineering-grade depth.

**Request category:** CREATE NEW DOCUMENTATION. None of the target files exist in the repository today; the existing FFmpeg documentation pipeline (Texinfo + Doxygen, under `[doc/]`) is left untouched. This is not an update of `[doc/muxers.texi:§hls]` (line 1887 of that file), which is a user-facing option reference and addresses a different need.

**Documentation types present in the scope** (all in a single deliverable):

- **Architecture documentation** — functional inventory, process flows, pipeline orchestration (Layers 1 and 2)
- **Technical reference** — codec/format decision logic, struct dictionary, integration surface (Layer 2)
- **API contract documentation** — invariants, data contracts, timing constraints, integration contracts (Layer 3)
- **Operational reference** — inputs/outputs, exception handling, consumer dependencies (Layer 1)

### 0.1.2 User Requirements Restated with Technical Precision

The user requirement maps cleanly to three output sub-trees and thirteen leaf documents. Each leaf has explicit length and content-completeness budgets (see Section 0.7). The restatement below preserves the user's filenames and output paths verbatim.

| Layer | Output Path | Files | Audience Anchor |
|-------|-------------|-------|-----------------|
| 1 — System Functionality ("What does this system do") | `/docs/hls-pipeline/functionality/` | `functional-inventory.md`, `inputs-outputs.md`, `consumer-dependencies.md`, `exception-handling.md` | Library integrators first, engineers second |
| 2 — System Technical Details ("How does it work under the hood") | `/docs/hls-pipeline/technical/` | `process-flows.md`, `codec-logic.md`, `data-model.md`, `pipeline-orchestration.md`, `integration-interfaces.md` | Engineers planning a port or rewrite |
| 3 — API Contracts ("What must be preserved exactly") | `/docs/hls-pipeline/api-contracts/` | `functional-invariants.md`, `data-contracts.md`, `timing-dependencies.md`, `integration-contracts.md` | Zero-deviation porting checklist for engineers |

### 0.1.3 Special Instructions and Constraints

CRITICAL directives extracted from the user prompt and surfaced verbatim where the user used quoted requirements:

- **Commit anchor**: "All source references (file paths, line numbers) MUST be anchored to commit `566ad786` (current HEAD at documentation time). Prepend each code reference block with the commit hash." This applies to every single source citation across all thirteen documents.
- **Mermaid-only diagrams**: "All diagrams MUST use Mermaid syntax (fenced mermaid blocks) for consistent markdown rendering." This rule appears three times in the user input — for `process-flows.md`, for `pipeline-orchestration.md`, and for `timing-dependencies.md`.
- **Decision tables, not pseudocode**: "Use decision tables for codec/format logic — not pseudocode" appears in the output-format mandate and is restated specifically for `codec-logic.md` ("Exhaustive — this is the most critical document. Use decision tables, not pseudocode").
- **Plain-language summary, then technical detail**: "Each section opens with a plain-language executive summary (readable by a library integrator without deep FFmpeg knowledge), followed by technical detail (for engineers), with code references traceable to source — file paths and line numbers, not just function names."
- **Zero production impact**: "All existing production logic and behavior — document as-is. No refactoring, optimization, or interface changes. Comments and documentation only." Translated to a project rule: no code edits anywhere in the repository.
- **Length and completeness targets**: Per-document length budgets are given verbatim by the user (see Section 0.7).

**User Examples**: None of the user input contains literal code/output examples to be preserved verbatim. The output filenames (e.g., `functional-inventory.md`, `inputs-outputs.md`) are themselves examples and ARE preserved verbatim.

**User Provided Templates**: None provided. The user supplies output requirements and length budgets, not a literal template.

**Style preferences**: Mixed plain-language + technical-depth, executive-summary-first per section, decision tables for logic branches, table format for I/O and contracts, narrative paragraphs for context, source citations as `path:line` (not function names).

**Web search requirements**: None. The HLS specification (RFC 8216) is referenced in the source itself (`[libavformat/hls.c:L26-L27]` cites `https://www.rfc-editor.org/rfc/rfc8216.txt`) and the user does not request external research. The documentation set is grounded entirely in repository source.

### 0.1.4 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- **To document each functional component**, we will CREATE one component section per major component within `functional-inventory.md`, where each component corresponds to a concrete struct or state machine in `[libavformat/hlsenc.c]` or `[libavformat/hls.c]` (`HLSContext`, `VariantStream`, `HLSSegment`, `ClosedCaptionsStream`, demuxer `playlist`, `variant`, `rendition`, `HLSCryptoContext`).
- **To document every input and output**, we will CREATE three tables in `inputs-outputs.md` — input table (AVPacket fields, AVFormatContext fields, AVOption values), output table (TS/fMP4 segment files, M3U8 playlist writes, downstream callbacks), per-component cross-reference — sourced from `hls_write_packet` (`[libavformat/hlsenc.c:L2410]`), the `options[]` array (`[libavformat/hlsenc.c:L3121-L3181]`), and the `ff_hls_write_*` family in `[libavformat/hlsplaylist.c]`.
- **To document downstream consumers**, we will CREATE `consumer-dependencies.md` and enumerate what depends on the M3U8 manifest format (HLS spec section), TS/fMP4 segment files, EXT-X-* tag emission, AVOutputFormat callback shape, and `libavformat/Makefile` Linkage (e.g., `[libavformat/Makefile:L189]` showing `dashenc` reuses `hlsplaylist.o`).
- **To document exception handling**, we will CREATE `exception-handling.md` and map every `AVERROR(*)` return in the in-scope files (`[libavformat/hlsenc.c]` 129 return statements, `[libavformat/hls.c]` 68 return statements) to recovery paths and `ignore_io_errors` behavior (`[libavformat/hlsenc.c:L3175]`).
- **To document process flows**, we will CREATE `process-flows.md` with one Mermaid diagram per major process: segment generation, playlist update, live sliding window, encryption key rotation. Each diagram traces from `avformat_write_header` (`[libavformat/avformat.h:L186-L188]`) through `av_write_frame`/`av_interleaved_write_frame` through `av_write_trailer`.
- **To document codec/format decision logic exhaustively**, we will CREATE `codec-logic.md` using decision tables for every if/switch branch: HLS_VERSION negotiation (`[libavformat/hlsenc.c:L1551-L1571]`), segment-cut keyframe rule (`[libavformat/hlsenc.c:L2470-L2475]`), TS vs fMP4 segment format (`SEGMENT_TYPE_MPEGTS` vs `SEGMENT_TYPE_FMP4` at `[libavformat/hlsenc.c:L116-L119]`), byterange mode, discontinuity insertion, EXT-X-TARGETDURATION (`[libavformat/hlsplaylist.c:L120]`), program-date-time injection, second-level segment filename expansion.
- **To document the data model**, we will CREATE `data-model.md` with a full field-level dictionary of `HLSContext` (`[libavformat/hlsenc.c:L202-L268]`), `VariantStream` (`[libavformat/hlsenc.c:L120-L194]`), `HLSSegment` (`[libavformat/hlsenc.c:L80-L96]`), `ClosedCaptionsStream` (`[libavformat/hlsenc.c:L196-L200]`), demuxer `HLSContext` (`[libavformat/hls.c:L204+]`), `playlist` (`[libavformat/hls.c:L101-L172]`), `segment` (`[libavformat/hls.c:L77-L87]`), `variant`/`rendition` (`[libavformat/hls.c:L184-L204]`), `HLSCryptoContext`/`HLSAudioSetupInfo` (`[libavformat/hls_sample_encryption.h:L42-L55]`), plus every referenced `AVFormatContext` / `AVStream` / `AVOutputFormat` / `AVOption` field used by the HLS code.
- **To document pipeline orchestration**, we will CREATE `pipeline-orchestration.md` covering `hls_init` → `hls_write_header` → `hls_write_packet` loop → `hls_write_trailer` → `hls_deinit` (entry points at `[libavformat/hlsenc.c:L2866, L2301, L2410, L2727, L2693]`) plus internal callbacks `hls_start`/`hls_window`/`hls_append_segment`/`hls_mux_init` (`[libavformat/hlsenc.c:L1675, L1531, L1042, L773]`).
- **To document every external interface**, we will CREATE `integration-interfaces.md` covering AVIOContext file/HTTP writes, protocol handlers (`file://`, `http://`, `https://`, `crypto:`), `hlsenc_io_open`/`hlsenc_io_close` HTTP persistent-connection logic (`[libavformat/hlsenc.c:L292-L331]`), AES-128 key fetch (`hls_encryption_start` `[libavformat/hlsenc.c:L714]`), `use_localtime`/`hls_segment_filename` filename templating, and HTTP method override (default PUT).
- **To document functional invariants**, we will CREATE `functional-invariants.md` as a zero-deviation checklist: M3U8 tag emission order, EXT-X-VERSION minimum negotiation (`[libavformat/hlsenc.c:L1551-L1571]`), segment naming conventions (`POSTFIX_PATTERN "_%d"` at `[libavformat/hlsenc.c:L75]`), EXT-X-DISCONTINUITY placement, PTS/DTS passthrough, extradata injection.
- **To document data contracts**, we will CREATE `data-contracts.md` with struct field formats, AVOption type and bounds (sourced from `[libavformat/hlsenc.c:L3121-L3181]`), timestamp unit conventions (`MPEG_TIME_BASE = 90000` at `[libavformat/hls.c:L56]`), AES-128 key/IV format (KEYSIZE 16 at `[libavformat/hlsenc.c:L72]`), byterange offsets.
- **To document timing dependencies**, we will CREATE `timing-dependencies.md` with a Mermaid sequencing diagram showing ordering constraints: keyframe detection precedes segment cut, segment write completes before playlist update, EXT-X-TARGETDURATION is computed before first segment.
- **To document integration contracts**, we will CREATE `integration-contracts.md` with one page per external contract: AES-128 key URI fetch protocol, fMP4 initialization segment delivery, HTTP chunked transfer, variant-stream bandwidth annotation (`[libavformat/hlsplaylist.c:L78-L108]`).

### 0.1.5 Inferred Documentation Needs

The user's three-layer specification implies several supporting deliverables that are not named explicitly but are required for the set to function as a deliverable:

- **Master index README at `/docs/hls-pipeline/README.md`** — required because the user specifies three sub-directories with thirteen files but no entry point. The README documents the reading order, the commit anchor convention, the citation format, and the audience separation between the three layers. Without this index, a first-time reader has no map.
- **Commit-anchor preamble in every file** — implicit from the user's "prepend each code reference block with the commit hash" rule. The README hosts the canonical commit anchor banner; every leaf document re-states the convention in a short preamble so files are still readable standalone.
- **Cross-layer references** — `codec-logic.md` decision tables and `data-model.md` dictionary entries cite each other. The pipeline orchestration document depends on data-model definitions. These cross-references are inferred from the user's "organize by functional component first, then technical detail" rule.
- **Decision-table format spec implicit on `codec-logic.md`** — the user states "use decision tables, not pseudocode" but does not pin a column structure. The implementation will use Condition / Branch / Resulting Behavior / Source Citation columns for every decision table to keep the format uniform.
- **No source code edits anywhere** — the user mandates documentation-only. Files that exist purely for code (e.g., `[libavformat/hlsenc.c]` itself) MUST NOT be modified to add doxygen comments or annotations.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals an established but **distinctly different** documentation infrastructure that already publishes user-facing reference material for FFmpeg, including a short HLS option reference. The new `/docs/hls-pipeline/` tree lives alongside this infrastructure without modifying it.

**Current documentation framework** (Source: `[doc/]` directory contents):

| Framework | Version | Configuration | Output |
|-----------|---------|---------------|--------|
| Doxygen | 1.8.8 | `[doc/Doxyfile:L1]` | `doc/doxy/` (source-level HTML, EXTRACT_ALL=YES at `[doc/Doxyfile]`); wrapper script `[doc/doxy-wrapper.sh]` |
| Texinfo | — | `[doc/Makefile]` driving `[doc/t2h.pm]`, `[doc/t2h.init]`, `[doc/texidep.pl]`, `[doc/texi2pod.pl]` | man pages (`MANPAGES1`/`MANPAGES3` at `[doc/Makefile:L17-L19]`), HTML pages, POD pages, TXT pages |
| Mermaid CLI (mmdc) | 10.6.1 | Locally installed (`/usr/bin/mmdc`) | Diagram validation only — not required by the rendering pipeline of the new docs |

**Documentation generator configuration locations**:

- `[doc/Doxyfile]` — Doxygen project descriptor (PROJECT_NAME = FFmpeg, OUTPUT_DIRECTORY = doc/doxy, GENERATE_HTML = YES, HAVE_DOT = NO, UML_LOOK = NO)
- `[doc/Makefile:L17-L42]` — defines `MANPAGES`, `PODPAGES`, `HTMLPAGES`, `TXTPAGES`, `DOCS`
- `[doc/t2h.init]` — deprecated texi2html wrapper; superseded by `[doc/t2h.pm]`
- `[doc/print_options.c]` — auto-generates per-format/codec option tables for Texinfo inclusion

**API documentation tools in use**: Doxygen comments are written inline in C headers (e.g., the `/** @file ... */` block at `[libavformat/hls.c:L24-L28]`).

**Diagram tools detected**: None used in the current FFmpeg docs. `mmdc` is available locally but unused in the existing pipeline. The new docs introduce Mermaid as fenced blocks rendered by the standard markdown viewer (GitHub/GitLab/IDE preview).

**Documentation hosting/deployment setup**: FFmpeg publishes generated HTML at `https://ffmpeg.org/ffmpeg.html`, `https://ffmpeg.org/ffplay.html`, `https://ffmpeg.org/ffprobe.html` (sourced from Texinfo); the new `/docs/hls-pipeline/` tree is repository-internal markdown — no deployment step is added.

**Existing HLS documentation in repository** (the new doc set augments but does not replace):

- `[doc/muxers.texi:L1887]` — `@section hls` (user-facing option reference for the HLS muxer, including `hls_init_time`, `hls_time`, `hls_segment_filename`, etc.)
- `[doc/demuxers.texi]` — contains HLS demuxer reference
- `[libavformat/hls.c:L24-L28]` — file-level Doxygen block stating "Apple HTTP Live Streaming demuxer; https://www.rfc-editor.org/rfc/rfc8216.txt"
- `[libavformat/hlsenc.c:L3-L4]` — copyright headers identifying authorship (Luca Barbato 2012, Akamai Technologies 2017)
- `[libavformat/hls_sample_encryption.h:L23-L27]` — file-level Doxygen identifying the Sample Encryption specification

**Repository markdown files** (root-level): `README.md`, `INSTALL.md`, `CONTRIBUTING.md`, `LICENSE.md`, `COPYING.GPLv2`, `COPYING.GPLv3`, `COPYING.LGPLv2.1`, `COPYING.LGPLv3`. The only markdown file under `doc/` is `[doc/transforms.md]`. No precedent for nested markdown documentation trees exists, so the new `/docs/hls-pipeline/` tree creates a new convention for in-repo engineering documentation.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns used to enumerate the code to document:

| Pattern | Purpose | Files Found |
|---------|---------|-------------|
| `libavformat/hls*` | All HLS muxer/demuxer code | `hls.c`, `hlsenc.c`, `hlsplaylist.c`, `hlsplaylist.h`, `hls_sample_encryption.c`, `hls_sample_encryption.h` |
| `libavformat/segment.c`, `libavformat/mpegtsenc.c`, `libavformat/mpegts.c`, `libavformat/mpegts.h` | Supporting muxers and demuxers directly called by HLS | confirmed by user scope |
| `libavformat/avformat.h`, `libavutil/opt.h`, `libavutil/aes.h`, `libavutil/dict.h` | Public headers defining types used by HLS | confirmed by user scope |
| `FFOutputFormat ff_hls_muxer` | Muxer registration | `[libavformat/hlsenc.c:L3191]` |
| `FFInputFormat ff_hls_demuxer` | Demuxer registration | `[libavformat/hls.c:L2900]` |
| `ff_hls_write_*` exports | Playlist tag writers | `[libavformat/hlsplaylist.c:L32, L40, L58, L78, L110, L134, L144, L201]` |
| `ff_hls_senc_*` exports | Sample encryption helpers | `[libavformat/hls_sample_encryption.h:L57-L62]` |

**Modules requiring documentation:**

- **HLS muxer module** (`[libavformat/hlsenc.c]` 3,207 LOC + `[libavformat/hlsplaylist.c]` 206 LOC + `[libavformat/hlsplaylist.h]` 65 LOC)
  - Public API surface: `FFOutputFormat ff_hls_muxer` (`[libavformat/hlsenc.c:L3191-L3207]`) — name "hls", extensions "m3u8", flags `AVFMT_NOFILE | AVFMT_GLOBALHEADER | AVFMT_NODIMENSIONS`, callbacks `init`/`write_header`/`write_packet`/`write_trailer`/`deinit`
  - Current documentation: user-facing option reference exists at `[doc/muxers.texi:L1887]`; no engineering-level architecture documentation
  - Documentation needed: complete architecture coverage (Layers 1, 2, 3) per user spec
- **HLS demuxer module** (`[libavformat/hls.c]` 2,912 LOC + `[libavformat/hls_sample_encryption.c]` 396 LOC + `[libavformat/hls_sample_encryption.h]` 65 LOC)
  - Public API surface: `FFInputFormat ff_hls_demuxer` (`[libavformat/hls.c:L2900-L2912]`) — name "hls", flags `AVFMT_NOGENSEARCH | AVFMT_TS_DISCONT | AVFMT_NO_BYTE_SEEK | AVFMT_SHOW_IDS`, callbacks `read_probe`/`read_header`/`read_packet`/`read_close`/`read_seek`
  - Current documentation: file-level Doxygen comment only at `[libavformat/hls.c:L24-L28]`
  - Documentation needed: identical three-layer treatment as muxer
- **Supporting segmenter** (`[libavformat/segment.c]` 1,136 LOC) — generic alternative segmenter, registered as `ff_segment_muxer` (`[libavformat/segment.c:L1107]`) and `ff_stream_segment_muxer` (`[libavformat/segment.c:L1123]`); referenced by `[doc/muxers.texi:L1911]` as the "see also" for HLS. Documentation needed: cross-reference inside `consumer-dependencies.md` and `integration-interfaces.md`.
- **MPEG-TS muxer** (`[libavformat/mpegtsenc.c]` 2,424 LOC + `[libavformat/mpegts.h]` 306 LOC) — the underlying mux used when `segment_type=mpegts` (default); registered as `ff_mpegts_muxer` (`[libavformat/mpegtsenc.c:L2408]`). Documentation needed: interface description in `integration-interfaces.md` (HLS calls `avformat_write_header` on the child MPEG-TS muxer context).
- **MPEG-TS demuxer** (`[libavformat/mpegts.c]` 3,735 LOC) — used by `[libavformat/hls.c]` to parse demuxed segments; `STREAM_TYPE_HLS_SE_VIDEO_H264 = 0xdb`, `STREAM_TYPE_HLS_SE_AUDIO_AAC = 0xcf`, `STREAM_TYPE_HLS_SE_AUDIO_AC3 = 0xc1`, `STREAM_TYPE_HLS_SE_AUDIO_EAC3 = 0xc2` (HLS sample encryption stream types) at `[libavformat/mpegts.h:L242-L247]`.

**Configuration options requiring documentation** — sourced from the `options[]` arrays:

- Muxer: 47 user-tunable options at `[libavformat/hlsenc.c:L3121-L3181]` (e.g., `hls_time` default 2,000,000 µs, `hls_list_size` default 5, `hls_segment_type` mpegts|fmp4, 16 `hls_flags` constants HLS_SINGLE_FILE through HLS_I_FRAMES_ONLY at `[libavformat/hlsenc.c:L99-L114]`, 4 `hls_start_number_source` modes at `[libavformat/hlsenc.c:L58-L63]`, 3 `hls_playlist_type` values at `[libavformat/hlsplaylist.h:L32-L36]`)
- Demuxer: options at `[libavformat/hls.c:L2852+]`

**Related documentation found** (existing docs that provide context, not replaced):

- `[doc/muxers.texi:L1887-L1956+]` — HLS muxer @section (user-facing CLI option reference)
- `[doc/demuxers.texi:§hls]` — HLS demuxer reference
- `[libavformat/Makefile:L189]` shows `dashenc.o` consumes `hlsplaylist.o` — the playlist tag writers are shared between HLS and DASH muxers (cross-format reuse documented in `consumer-dependencies.md`)
- `[libavformat/Makefile:L276-L277]` — `OBJS-$(CONFIG_HLS_DEMUXER) += hls.o hls_sample_encryption.o`; `OBJS-$(CONFIG_HLS_MUXER) += hlsenc.o hlsplaylist.o`
- `[libavformat/allformats.c:L216-L217]` — `extern const FFInputFormat ff_hls_demuxer; extern const FFOutputFormat ff_hls_muxer;`

### 0.2.3 Web Search Research Conducted

**No web search is required** for this documentation set. Justification:

- The user explicitly anchors all references to the local commit `566ad786`
- The HLS specification (RFC 8216) URL is already cited in `[libavformat/hls.c:L26]`
- The HLS Sample Encryption specification URL is already cited in `[libavformat/hls_sample_encryption.h:L25-L26]` (`https://developer.apple.com/library/ios/documentation/AudioVideo/Conceptual/HLS_Sample_Encryption`)
- The HLS Sample Encryption Format URL is cited in `[libavformat/mpegts.h:L233-L235]` (`https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/HLS_Sample_Encryption/`)
- All decision logic, AVOption defaults, struct fields, and tag emission are present in the source code

Should the documentation authoring stage need to reference RFC 8216 by section number (e.g., to assert M3U8 spec compliance), the citation will be to the RFC's section number directly (e.g., RFC 8216 §4.4.2.1 for EXT-X-TARGETDURATION) — no live web lookup is needed because the specification is stable.


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The user defines an explicit in-scope source list and a thirteen-document output set. The mapping below shows, for every source-code component the new docs MUST cover, which of the thirteen output files contain that coverage.

**Modules requiring documentation:**

| Source Module | Public APIs / Key Symbols | Current Documentation | Documentation Needed |
|---------------|---------------------------|------------------------|-----------------------|
| `[libavformat/hlsenc.c]` (3,207 LOC) | `FFOutputFormat ff_hls_muxer` (L3191), `hls_init` (L2866), `hls_write_header` (L2301), `hls_write_packet` (L2410), `hls_write_trailer` (L2727), `hls_deinit` (L2693), `hls_start` (L1675), `hls_window` (L1531), `hls_append_segment` (L1042), `hls_mux_init` (L773), `hls_encryption_start` (L714), `options[]` (L3121) | None at engineer level | All 13 files |
| `[libavformat/hlsplaylist.c]` (206 LOC) + `[libavformat/hlsplaylist.h]` (65 LOC) | `ff_hls_write_playlist_version` (L32), `ff_hls_write_audio_rendition` (L40), `ff_hls_write_subtitle_rendition` (L58), `ff_hls_write_stream_info` (L78), `ff_hls_write_playlist_header` (L110), `ff_hls_write_init_file` (L134), `ff_hls_write_file_entry` (L144), `ff_hls_write_end_list` (L201), `PlaylistType` enum (`.h:L32-L36`) | None | `functional-inventory.md`, `inputs-outputs.md`, `consumer-dependencies.md` (DASH reuses these), `data-model.md`, `data-contracts.md`, `integration-interfaces.md`, `integration-contracts.md` |
| `[libavformat/hls.c]` (2,912 LOC) | `FFInputFormat ff_hls_demuxer` (L2900), `hls_probe` (L2814), `hls_read_header` (L2144), `hls_read_packet` (L2546), `hls_read_seek` (L2709), `hls_close` (L2127), demuxer `HLSContext`/`playlist`/`segment`/`variant`/`rendition` structs (L77-L204) | File-level Doxygen at L24-L28 only | All 13 files |
| `[libavformat/hls_sample_encryption.c]` (396 LOC) + `[libavformat/hls_sample_encryption.h]` (65 LOC) | `HLSCryptoContext` (`.h:L42-L46`), `HLSAudioSetupInfo` (`.h:L48-L55`), `ff_hls_senc_read_audio_setup_info` (`.h:L57`), `ff_hls_senc_parse_audio_setup_info` (`.h:L59`), `ff_hls_senc_decrypt_frame` (`.h:L61`) | File-level Doxygen `.h:L23-L27` only | `functional-inventory.md`, `data-model.md`, `data-contracts.md`, `integration-interfaces.md`, `integration-contracts.md`, `functional-invariants.md` |
| `[libavformat/segment.c]` (1,136 LOC) | `FFOutputFormat ff_segment_muxer` (L1107), `FFOutputFormat ff_stream_segment_muxer` (L1123), `seg_init` (L690), `seg_write_header` (L838), `seg_write_packet` (L867), `seg_write_trailer` (L1011) | None | `consumer-dependencies.md` (alternative segmenter), `integration-interfaces.md` |
| `[libavformat/mpegtsenc.c]` (2,424 LOC) | `FFOutputFormat ff_mpegts_muxer` (L2408) — used as the underlying mux when `segment_type=mpegts` | Existing texinfo reference | `integration-interfaces.md`, `integration-contracts.md` (TS segment format contract), `pipeline-orchestration.md` (sub-muxer lifecycle) |
| `[libavformat/mpegts.c]` (3,735 LOC) + `[libavformat/mpegts.h]` (306 LOC) | `FFInputFormat ff_mpegts_demuxer` (L3710), `STREAM_TYPE_HLS_SE_*` constants (`.h:L242-L246`), `avpriv_mpegts_parse_open`/`_parse_packet`/`_parse_close` (`.h:L257-L260`) | Existing texinfo reference | `data-model.md` (TS constants), `integration-interfaces.md`, `integration-contracts.md` (HLS SE stream type contract) |
| `[libavformat/avformat.h]` (3,163 LOC) | `AVFormatContext`, `AVStream`, `AVOutputFormat`, `AVInputFormat`, `avformat_write_header`, `av_write_frame`, `av_interleaved_write_frame`, `av_write_trailer` (described L186-L237) | Public Doxygen | `data-model.md` (referenced fields), `data-contracts.md`, `pipeline-orchestration.md` |
| `[libavutil/opt.h]` (1,194 LOC) | `AVOption` infrastructure consumed by HLS option table | Public Doxygen | `data-model.md` (AVOption referenced fields), `data-contracts.md` (AV_OPT_TYPE_* semantics) |
| `[libavutil/aes.h]` (69 LOC) | `av_aes_alloc` (L40), `av_aes_init` (L50), `av_aes_crypt` (L61) | Public Doxygen | `integration-interfaces.md`, `integration-contracts.md` (AES-128 key/IV format contract) |
| `[libavutil/dict.h]` (242 LOC) | `AVDictionary`, `av_dict_set`, `av_dict_get`, `av_dict_iterate`, `av_dict_free` | Public Doxygen | `data-model.md` (AVDictionary referenced), `data-contracts.md` |

**Configuration options requiring documentation:**

- **HLS muxer options** at `[libavformat/hlsenc.c:L3121-L3181]`: 47 entries — every one documented in `inputs-outputs.md` (table) and `data-contracts.md` (type/bounds contract).
- **HLS muxer flags** at `[libavformat/hlsenc.c:L99-L114]`: 16 flags — `HLS_SINGLE_FILE`, `HLS_DELETE_SEGMENTS`, `HLS_ROUND_DURATIONS`, `HLS_DISCONT_START`, `HLS_OMIT_ENDLIST`, `HLS_SPLIT_BY_TIME`, `HLS_APPEND_LIST`, `HLS_PROGRAM_DATE_TIME`, `HLS_SECOND_LEVEL_SEGMENT_INDEX`, `HLS_SECOND_LEVEL_SEGMENT_DURATION`, `HLS_SECOND_LEVEL_SEGMENT_SIZE`, `HLS_TEMP_FILE`, `HLS_PERIODIC_REKEY`, `HLS_INDEPENDENT_SEGMENTS`, `HLS_I_FRAMES_ONLY` — every one documented as a decision branch in `codec-logic.md`.
- **Start sequence source types** at `[libavformat/hlsenc.c:L58-L63]`: 4 modes — `HLS_START_SEQUENCE_AS_START_NUMBER`, `_AS_SECONDS_SINCE_EPOCH`, `_AS_FORMATTED_DATETIME`, `_AS_MICROSECONDS_SINCE_EPOCH`.
- **Segment types** at `[libavformat/hlsenc.c:L116-L119]`: 2 modes — `SEGMENT_TYPE_MPEGTS`, `SEGMENT_TYPE_FMP4`.
- **Playlist types** at `[libavformat/hlsplaylist.h:L32-L36]`: `PLAYLIST_TYPE_NONE`, `_EVENT`, `_VOD`, `_NB`.
- **Demuxer key types** at `[libavformat/hls.c:L71-L74]`: `KEY_NONE`, `KEY_AES_128`, `KEY_SAMPLE_AES`.
- **HLS muxer options** are also exposed verbatim in the user-facing `[doc/muxers.texi:L1887+]` reference — the new docs MUST be consistent with the texinfo descriptions while expanding into engineering depth.

**Features requiring user guides** (these are the "components" listed by the user under the functional-inventory entry):

- **Segment generation** — covered in `functional-inventory.md` + `process-flows.md` (Mermaid) + `codec-logic.md` (segment-cut decision table) + `pipeline-orchestration.md` (`hls_write_packet` loop). Current coverage: none at engineer level. Gaps: keyframe-boundary detection, `hls_time` vs `hls_init_time` logic, audio-only `start_pts_from_audio` semantics.
- **Playlist construction** — covered in `functional-inventory.md` + `process-flows.md` (playlist-update flow) + `codec-logic.md` (HLS_VERSION negotiation; EXT-X-* tag conditions) + `pipeline-orchestration.md` (`hls_window` call sites). Current coverage: none at engineer level. Gaps: byterange mode playlist layout, temp-file rename atomicity, `master_publish_rate` cadence.
- **Encryption handling** — covered in `functional-inventory.md` + `process-flows.md` (encryption key rotation flow) + `codec-logic.md` (key reload conditions, `periodic_rekey` semantics, IV derivation) + `integration-interfaces.md` (key URI fetch) + `integration-contracts.md` (AES-128 + Sample Encryption contracts). Current coverage: file-level Doxygen only. Gaps: key info file format, IV computation when `iv_string` absent, Sample Encryption AAC priming.
- **Variant stream management** — covered in `functional-inventory.md` (VariantStream struct) + `codec-logic.md` (`var_stream_map` parsing) + `data-model.md` (full VariantStream dictionary). Current coverage: option descriptions in texinfo only.
- **Live vs VOD modes** — covered in `functional-inventory.md` + `codec-logic.md` (`pl_type` decisions; `omit_endlist` and `delete_segments` interactions) + `timing-dependencies.md` (sliding-window ordering). Current coverage: option descriptions only.
- **Discontinuity handling** — covered in `functional-inventory.md` + `codec-logic.md` (`discont_start`, mid-stream `EXT-X-DISCONTINUITY` emission at `[libavformat/hlsenc.c:L1594]`, `discont_program_date_time` recalculation at `[libavformat/hlsenc.c:L1273-L1275]`) + `functional-invariants.md` (placement rules). Current coverage: none.

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Engineer-facing architecture documentation**: 0% coverage today. The user spec creates this from zero.
- **Undocumented public APIs** in scope: `hls_init`, `hls_write_header`, `hls_write_packet`, `hls_write_trailer`, `hls_deinit`, `hls_read_header`, `hls_read_packet`, `hls_read_seek`, `hls_close`, `hls_probe`, every `ff_hls_write_*`, every `ff_hls_senc_*`. None has a usage-level description outside the source comments. The new docs close this gap completely.
- **Missing component-relationship narrative**: No document today explains how `HLSContext` relates to `VariantStream`, how `VariantStream` relates to `HLSSegment`, or how the muxer's child `AVFormatContext *avf` field (`[libavformat/hlsenc.c:L134]`) routes packets to either the MPEG-TS muxer or the fMP4 muxer. The user's `data-model.md` and `pipeline-orchestration.md` deliverables close this gap.
- **Incomplete decision-logic reference**: HLS_VERSION negotiation (`[libavformat/hlsenc.c:L1551-L1571]`) cascades through six conditions producing versions 2, 3, 4, 6, or 7; no single reference document captures this matrix. `codec-logic.md` resolves this.
- **Outdated documentation**: None applicable — there is no prior engineer-level HLS documentation to update or supersede.
- **Missing cross-format dependency note**: `[libavformat/Makefile:L189]` shows `dashenc.o` consumes `hlsplaylist.o` — DASH muxer reuses HLS playlist writers. This dependency is not documented anywhere; `consumer-dependencies.md` will surface it.
- **Missing protocol contract documentation**: `hlsenc_io_open` (`[libavformat/hlsenc.c:L292-L311]`) implements HTTP-persistent-connection reuse via `ff_http_do_new_request`; default HTTP method is PUT (`[libavformat/hlsenc.c:L334]`); these contracts are not documented today. `integration-contracts.md` resolves this.
- **AVERROR mapping**: 129 return statements in `[libavformat/hlsenc.c]` and 68 in `[libavformat/hls.c]` produce `AVERROR(ENOMEM)`, `AVERROR(EINVAL)`, `AVERROR(EIO)`, `AVERROR_MUXER_NOT_FOUND`, `AVERROR_PATCHWELCOME`, `AVERROR_EOF`. No consolidated mapping exists. `exception-handling.md` resolves this with a per-scenario table organized by failure mode.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The new documentation tree is rooted at `/docs/hls-pipeline/` with the three sub-directories named by the user verbatim. A `README.md` index file is added at the root to provide entry-point navigation, commit-anchor banner, and reading-order guidance. Trailing comments in the layout show the user-specified intent for each leaf.

```text
docs/
└── hls-pipeline/
    ├── README.md                          # Index: 3-layer map, commit anchor 566ad786, reading order, citation format
    ├── functionality/                     # Layer 1: "What does this system do" — plain-language first
    │   ├── functional-inventory.md        # One section per HLS component (1–2 pages each)
    │   ├── inputs-outputs.md              # Table format: every I/O captured per component
    │   ├── consumer-dependencies.md       # 3–5 pages: downstream consumers of HLS outputs
    │   └── exception-handling.md          # 2–4 pages: failure scenarios → AVERROR codes → recovery
    ├── technical/                         # Layer 2: "How does it work under the hood" — engineer depth
    │   ├── process-flows.md               # One Mermaid diagram per major process + 1 page narrative each
    │   ├── codec-logic.md                 # Exhaustive decision tables (no length cap, no pseudocode)
    │   ├── data-model.md                  # Full field-level dictionary of every struct
    │   ├── pipeline-orchestration.md      # 3–5 pages + Mermaid dependency diagram
    │   └── integration-interfaces.md      # One page per external interface
    └── api-contracts/                     # Layer 3: "What must be preserved exactly"
        ├── functional-invariants.md       # Zero-deviation checklist, no limit
        ├── data-contracts.md              # Full reference, table format
        ├── timing-dependencies.md         # 2–3 pages + Mermaid sequencing diagram
        └── integration-contracts.md       # One page per external contract
```

### 0.4.2 Content Generation Strategy

#### 0.4.2.1 Information Extraction Approach

The new documentation is generated by extracting facts from the in-scope source. The extraction map is:

| Information Category | Primary Source | Target Document |
|----------------------|----------------|-----------------|
| Component inventory | Struct definitions in `[libavformat/hlsenc.c:L80-L268]` and `[libavformat/hls.c:L77-L204]` | `functional-inventory.md` |
| AVOption table | `[libavformat/hlsenc.c:L3121-L3181]` and demuxer options at `[libavformat/hls.c:L2852+]` | `inputs-outputs.md`, `data-contracts.md` |
| EXT-X-* tag emission | `[libavformat/hlsplaylist.c:L32-L205]` plus muxer-specific emissions at `[libavformat/hlsenc.c:L1594, L1598, L1603]` | `inputs-outputs.md`, `functional-invariants.md` |
| HLS_VERSION negotiation | `[libavformat/hlsenc.c:L1551-L1571]` | `codec-logic.md`, `functional-invariants.md` |
| Segment-cut decision | `[libavformat/hlsenc.c:L2470-L2503]` | `codec-logic.md`, `process-flows.md` |
| HLS_FLAGS enum | `[libavformat/hlsenc.c:L99-L114]` | `codec-logic.md`, `data-model.md` |
| Segment lifecycle | `hls_append_segment` `[libavformat/hlsenc.c:L1042-L1288]`, `hls_window` `[libavformat/hlsenc.c:L1531-L1660]`, `hls_start` `[libavformat/hlsenc.c:L1675+]` | `pipeline-orchestration.md`, `process-flows.md` |
| Lifecycle entry points | `hls_init` (L2866), `hls_write_header` (L2301), `hls_write_packet` (L2410), `hls_write_trailer` (L2727), `hls_deinit` (L2693) | `pipeline-orchestration.md`, `timing-dependencies.md` |
| Demuxer probe and read | `hls_probe` (L2814), `hls_read_header` (L2144), `hls_read_packet` (L2546), `hls_read_seek` (L2709) | `pipeline-orchestration.md`, `process-flows.md` |
| AES-128 encryption | `hls_encryption_start` `[libavformat/hlsenc.c:L714]` plus `[libavutil/aes.h:L40-L62]` | `integration-interfaces.md`, `integration-contracts.md` |
| HLS Sample Encryption | `[libavformat/hls_sample_encryption.c]`, `[libavformat/hls_sample_encryption.h:L42-L62]`, `[libavformat/mpegts.h:L242-L246]` | `integration-interfaces.md`, `integration-contracts.md` |
| HTTP persistence | `hlsenc_io_open` `[libavformat/hlsenc.c:L292-L311]`, `hlsenc_io_close` `[libavformat/hlsenc.c:L313-L331]`, `set_http_options` `[libavformat/hlsenc.c:L333-L350]` | `integration-interfaces.md`, `integration-contracts.md` |
| Error returns / AVERROR mapping | `grep -n "return AVERROR" libavformat/hlsenc.c libavformat/hls.c` — 129 + 68 sites | `exception-handling.md` |

#### 0.4.2.2 Template Application

No user-provided template exists. Implementation will apply a uniform per-document template (also itself a project rule — see Section 0.10):

- Top-of-file commit-anchor banner: "All source references in this document are anchored to commit `566ad786`."
- Plain-language executive summary (1–3 paragraphs, integrator audience)
- Technical detail body (decision tables, narrative, field tables, code-reference blocks; engineer audience)
- Per-claim source citations of the form `[path:Lstart-Lend]` immediately after the claim

#### 0.4.2.3 Documentation Standards

Standards applied uniformly across all 14 output files:

- **Markdown formatting** — standard CommonMark headers (`#`, `##`, `###`, `####`); no HTML except for hard line breaks
- **Mermaid diagram integration** — fenced code blocks tagged `mermaid` (rule mandated three times by user). Diagram types: `flowchart`, `sequenceDiagram`, `stateDiagram-v2`, `classDiagram`, depending on what the diagram conveys
- **Code examples** — fenced code blocks tagged with the language (`c`, `m3u8`, `bash`); short, ≤2–3 lines per snippet (per the project's professional documentation standard)
- **Source citations** — inline `[path:Lstart-Lend]` brackets immediately after the claim, with each code reference block prepended by the commit hash `566ad786` per user mandate
- **Tables** — used for parameter descriptions, return values, decision branches, contracts, and inventories
- **Consistent terminology** — `muxer`/`demuxer`/`mux`/`demux` matches FFmpeg convention; `playlist` is used for M3U8 manifest, `segment` for a single TS/fMP4 file, `variant` for an EXT-X-STREAM-INF entry

### 0.4.3 Diagram and Visual Strategy

The user mandates Mermaid for all diagrams. Diagram types per document:

| Document | Diagram Type | Mermaid Kind | Source Code Anchor |
|----------|--------------|--------------|---------------------|
| `process-flows.md` (Layer 2) | Segment generation flow | `flowchart LR` | `hls_write_packet` `[libavformat/hlsenc.c:L2410-L2691]` |
| `process-flows.md` | Playlist update flow | `flowchart LR` | `hls_window` `[libavformat/hlsenc.c:L1531-L1660]` |
| `process-flows.md` | Live sliding window flow | `flowchart LR` | `hls_delete_old_segments` `[libavformat/hlsenc.c:L531]` + `hls_window` |
| `process-flows.md` | Encryption key rotation flow | `flowchart LR` | `hls_encryption_start` `[libavformat/hlsenc.c:L714]` + `HLS_PERIODIC_REKEY` flag at `[libavformat/hlsenc.c:L112]` |
| `pipeline-orchestration.md` (Layer 2) | Lifecycle DAG | `flowchart TB` showing init → write_header → write_packet loop → write_trailer → deinit | `[libavformat/hlsenc.c:L3191-L3207]` (FFOutputFormat callback assignment) |
| `pipeline-orchestration.md` | Child muxer dependency | `classDiagram` showing `HLSContext` -> `VariantStream` -> `avf: AVFormatContext` -> ts/fmp4 muxer | `[libavformat/hlsenc.c:L134]` |
| `timing-dependencies.md` (Layer 3) | Ordering constraints | `sequenceDiagram` showing decoder → keyframe detect → segment-cut → segment-write → playlist-update | `[libavformat/hlsenc.c:L2470-L2691]` |
| `data-model.md` (Layer 2) | Struct relationships (optional) | `classDiagram` | All structs at `[libavformat/hlsenc.c:L80-L268]` |

Screenshot/image requirements: none — the documentation set is pure markdown + Mermaid.

Architecture diagram specifications: Mermaid `flowchart` orientation will be `LR` (left-to-right) for process flows and `TB` (top-to-bottom) for lifecycle. Node labels MUST use `[Component]` for synchronous boxes and `((State))` for state markers when state machines are diagrammed (live mode, VOD mode, paused).


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

CRITICAL: Every documentation file to be created, updated, or referenced is enumerated below. Target file is listed FIRST in each row. All thirteen user-named files are CREATE; one supporting README.md index is also CREATE.

Documentation Transformation Modes:

- CREATE — Create a new documentation file
- UPDATE — Update an existing documentation file
- DELETE — Remove an obsolete documentation file
- REFERENCE — Use as an example for documentation style and structure

| Target Documentation File | Transformation | Source Code/Docs | Content / Changes |
|---------------------------|----------------|------------------|-------------------|
| `docs/hls-pipeline/README.md` | CREATE | (none — net new) | Master index for the doc set: 3-layer map; commit anchor banner stating "All references anchored to commit 566ad786"; recommended reading order (integrator path: functionality → contracts; engineer path: functional-inventory → process-flows → codec-logic → data-model → pipeline-orchestration → contracts); citation format spec; cross-doc dependency note |
| `docs/hls-pipeline/functionality/functional-inventory.md` | CREATE | `libavformat/hlsenc.c`, `libavformat/hls.c`, `libavformat/hlsplaylist.c`, `libavformat/hls_sample_encryption.c` | One section per component (target 1–2 pages each): Segment generation (cites `hls_write_packet:L2410`, `hls_start:L1675`); Playlist construction (cites `hls_window:L1531`, `hlsplaylist.c:L32-L205`); Encryption handling (cites `hls_encryption_start:L714`, `hls_sample_encryption.c`); Variant stream management (cites `VariantStream:L120-L194`); Live vs VOD modes (cites `pl_type` option:L3169-L3171); Discontinuity handling (cites L1594, L1273-L1275); Demuxer probe + playlist parsing (cites `hls_probe:L2814`, `hls_read_header:L2144`); each section opens with plain-language summary, then technical detail with `path:line` citations |
| `docs/hls-pipeline/functionality/inputs-outputs.md` | CREATE | `libavformat/hlsenc.c:L3121-L3181` (options array), `libavformat/hlsplaylist.c` (tag writers), `libavformat/hls_sample_encryption.h:L42-L55` | Table-format I/O capture, every entry captured (no summarizing). Per component: Inputs table (AVPacket fields used at L2410-L2502, AVFormatContext fields, all 47 AVOption entries, key info file format); Outputs table (TS segment files, fMP4 init.mp4 segment files, M3U8 playlist writes with each EXT-X-* tag emission, child muxer callbacks, HTTP PUT/DELETE requests); Demuxer side: input M3U8 lines parsed (L863-L979) and output AVPacket fields |
| `docs/hls-pipeline/functionality/consumer-dependencies.md` | CREATE | `libavformat/Makefile:L189` (DASH reuses hlsplaylist.o), HLS spec consumers | 3–5 pages covering: media players (any HLS-compliant player consuming M3U8 + TS/fMP4 segments — what they expect from EXT-X-VERSION-negotiated playlists), CDN ingest (HTTP PUT default at L334, `hls_base_url`, persistent connection behavior), test suites (where in `[tests/fate/]` HLS-related references exist — survey only, not modified per scope), downstream libavformat consumers (DASH muxer at `[libavformat/Makefile:L189]` reusing hlsplaylist writers — concrete cross-format dependency), what would break if hlsenc stopped (no fallback segmenter for the HLS exact M3U8 emission shape) |
| `docs/hls-pipeline/functionality/exception-handling.md` | CREATE | `libavformat/hlsenc.c` (129 return statements), `libavformat/hls.c` (68 return statements) | 2–4 pages organized by failure scenario, each mapped to AVERROR. Scenarios: bad input packet (AVERROR(EINVAL) at L583, L587, L755), missing keyframe (`can_split = 0` path at L2477-L2479), I/O failures (ENOMEM at L278, EIO via `ignore_io_errors` flag at L3175), mid-stream discontinuities (`vs->discontinuity` set on time gap at L1273), clock discontinuities (`av_compare_ts` boundary at L2502, `discont_program_date_time` recalculation at L1273), segment write failures (`hls_rename_temp_file` at L1300, `flush_dynbuf` at L2611), demuxer broken-playlist recovery (`m3u8_hold_counters` retries) |
| `docs/hls-pipeline/technical/process-flows.md` | CREATE | `hls_write_packet:L2410`, `hls_window:L1531`, `hls_delete_old_segments:L531`, `hls_encryption_start:L714` | One Mermaid `flowchart LR` per major process + 1 page of supporting narrative each. Diagrams: (1) Segment generation (avformat_open_input/write_header → packet loop → keyframe boundary → flush → close segment file → append segment metadata); (2) Playlist update (sequence increment → target_duration max → ff_hls_write_playlist_header → per-segment file entry → optional EXT-X-ENDLIST); (3) Live sliding window (`hls_list_size` triggers `hls_delete_old_segments` → unlink old segment files → adjust sequence); (4) Encryption key rotation (key_info_file reload on `HLS_PERIODIC_REKEY` → av_aes_init with new key → EXT-X-KEY line emitted before next segment). All diagrams use fenced `mermaid` code blocks. AVFormatContext/AVIOContext/segment-file-writer interaction shown explicitly in each diagram. |
| `docs/hls-pipeline/technical/codec-logic.md` | CREATE | All decision branches in `libavformat/hlsenc.c`, `libavformat/hls.c`, `libavformat/hlsplaylist.c` | Exhaustive decision tables (no length cap, NO pseudocode per user mandate). Tables for: HLS_VERSION negotiation (6 conditions, 5 outcomes — citing L1551-L1571); segment-cut decision (keyframe flag + `HLS_SPLIT_BY_TIME` + `pts - end_pts > 0` + `pts - start_pts ≥ end_pts` at L2470-L2502); TS vs fMP4 selection (`SEGMENT_TYPE_*` enum at L116-L119; affects styp header emission, EXT-X-MAP, EXT-X-BYTERANGE eligibility, version bump to 7); EXT-X-TARGETDURATION computation (max of all segment durations rounded — `lrint(en->duration)` at L1585-L1586); EXT-X-DISCONTINUITY placement (DISCONT_START at sequence start L1594; mid-stream when `vs->discontinuity` set); EXT-X-PROGRAM-DATE-TIME injection (HLS_PROGRAM_DATE_TIME flag at L106; rolling recompute via `discont_program_date_time` at L1273-L1275, L1622-L1623); byterange mode logic (`HLS_SINGLE_FILE` OR `max_seg_size > 0` at L2503; forces version 4, sequence reset to 0 at L1557-L1559); start sequence source (4 modes at L58-L63: number, epoch_seconds, epoch_microseconds, formatted_datetime); second-level filename templating (%d/%t/%s expansion when `use_localtime` is set, flags HLS_SECOND_LEVEL_SEGMENT_* at L107-L109); HTTP method selection (`method` option overrides default PUT at L334-L336); periodic_rekey reload cadence; iframes_only EXT-X-I-FRAMES-ONLY emission rules; independent_segments emission when `vs->has_video` (L1597-L1599); appended-list mode disables `init_time` (L3104-L3105) |
| `docs/hls-pipeline/technical/data-model.md` | CREATE | All in-scope struct definitions + referenced fields | Full field-level dictionary (no summarizing per user mandate). Sections: HLSContext (muxer, L202-L268 — every one of 60+ fields including `start_sequence`, `time`, `init_time`, `max_nb_segments`, `hls_delete_threshold`, `flags`, `pl_type`, `segment_type`, `use_localtime`, all key/IV strings, `var_streams` pointer, `nb_varstreams`, `cc_streams`, `master_m3u8_created`, etc.); VariantStream (L120-L194 — `sequence`, `oformat`, `vtt_oformat`, `out`, `out_single_file`, `packets_written`, `init_range_length`, `temp_buffer`, `init_buffer`, `avf`/`vtt_avf` child muxer contexts, `has_video`/`has_subtitle`/`new_start`/`start_pts_from_audio`, `dpp` duration-per-packet, `start_pts`/`end_pts`/`video_lastpos`/`video_keyframe_pos`/`video_keyframe_size`, `duration`, `start_pos`, `size`, `nb_entries`, `discontinuity_set`, `discontinuity`, `reference_stream_index`, `total_size`, `total_duration`, `avg_bitrate`, `max_bitrate`, `segments`/`last_segment`/`old_segments` linked-list heads, all `basename`/`m3u8_name`/`vtt_basename`/`vtt_m3u8_name`/`fmp4_init_filename`/`base_output_dirname` strings, `initial_prog_date_time`, `current_segment_final_filename_fmt`, `encrypt_started`, all key/IV buffers, `streams`/`nb_streams`/`codec_attr`/`attr_status`, `m3u8_created`, `is_default`, `language`/`agroup`/`sgroup`/`ccgroup`/`varname`/`subtitle_varname`); HLSSegment (L80-L96 — `filename`, `sub_filename`, `duration`, `discont`, `pos`, `size`, `keyframe_pos`, `keyframe_size`, `var_stream_idx`, `key_uri`, `iv_string`, `next` linked-list pointer, `discont_program_date_time`, `buf[]` flexible-array tail); ClosedCaptionsStream (L196-L200); Demuxer HLSContext (`libavformat/hls.c:L204+`); demuxer playlist (L101-L172 — `url`, `pb`, `read_buffer`, `input`/`input_next`/`input_read_done`, `parent`, `index`, `ctx`, `pkt`, `has_noheader_flag`, `main_streams`/`n_main_streams`, `finished`, `type`, `target_duration`, `start_seq_no`/`time_offset_flag`/`start_time_offset`, `n_segments`/`segments`, `needed`, `broken`, `cur_seq_no`/`last_seq_no`, `m3u8_hold_counters`, `cur_seg_offset`, `last_load_time`, `cur_init_section`/`init_sec_buf`/`init_sec_buf_size`/`init_sec_data_len`/`init_sec_buf_read_offset`, `key_url`/`key`, all ID3 fields, `audio_setup_info`, `seek_*` fields, `n_renditions`/`renditions`, `n_init_sections`/`init_sections`, `is_subtitle`); demuxer segment (L77-L87); demuxer variant (L194-L204); demuxer rendition (L184-L191); HLSCryptoContext (`hls_sample_encryption.h:L42-L46`); HLSAudioSetupInfo (`hls_sample_encryption.h:L48-L55`); referenced AVFormatContext fields (from `avformat.h`); referenced AVStream fields; referenced AVOutputFormat fields; referenced AVOption fields (`opt.h`); referenced AVDictionary entries (`dict.h`); HLSFlags enum (L99-L114); SegmentType enum (L116-L119); StartSequenceSourceType enum (L58-L63); PlaylistType enum (`hlsplaylist.h:L32-L36`); demuxer KeyType enum (`hls.c:L71-L74`); demuxer PlaylistType enum (`hls.c:L92-L96`); MPEG-TS STREAM_TYPE_HLS_SE_* constants (`mpegts.h:L242-L246`); KEYSIZE 16 (L72), LINE_BUFFER_SIZE (L73), HLS_MICROSECOND_UNIT 1000000 (L74), BUFSIZE 16*1024 (L75), POSTFIX_PATTERN "_%d" (L76); HLS_MAX_ID3_TAGS_DATA_LEN 138 and HLS_MAX_AUDIO_SETUP_DATA_LEN 10 (`hls_sample_encryption.h:L37-L38`) |
| `docs/hls-pipeline/technical/pipeline-orchestration.md` | CREATE | `hls_init:L2866`, `hls_write_header:L2301`, `hls_write_packet:L2410`, `hls_write_trailer:L2727`, `hls_deinit:L2693`, FFOutputFormat callback table at L3191 | 3–5 pages plus a Mermaid `flowchart TB` dependency diagram. Topics: execution order (`avformat_write_header` calls `hls_init` first via `init` slot, then `hls_write_header` proper, then loops `hls_write_packet`, then `hls_write_trailer`, then `hls_deinit` per FFOutputFormat ordering at L3201-L3206); internal callback chain (`hls_write_packet` → `hls_append_segment` → `sls_flags_filename_process` → `hls_window` → `hls_start`); segment finalization (`av_write_frame(oc, NULL)` flush at L2503, `avio_close_dyn_buf` for fMP4 init at L2508, `avio_flush` then close); playlist-flush timing (after segment file fully written — see Section 0.5.2 of timing-dependencies); restart/recovery behavior (`ignore_io_errors` flag — failures swallowed when set); demuxer lifecycle (`hls_probe` → `hls_read_header` → `hls_read_packet` loop → `hls_close`); child muxer relationship via `VariantStream::avf` AVFormatContext pointer (the HLS muxer is a "meta-muxer" that wraps an MPEG-TS or fMP4 sub-muxer). All diagrams use fenced `mermaid` code blocks. |
| `docs/hls-pipeline/technical/integration-interfaces.md` | CREATE | `hlsenc_io_open:L292`, `hlsenc_io_close:L313`, `set_http_options:L333`, `hls_encryption_start:L714`, `libavutil/aes.h:L40-L62`, `libavformat/hls_sample_encryption.h` | One page per interface, structured consistently. Interfaces: AVIOContext file writes (`avio_write`/`avio_flush`/`avio_tell` on segment files); AVIOContext HTTP writes (with `http_persistent` reuse — L302-L304); protocol handlers (`file://`, `http://`, `https://` — schemes detected by `ff_is_http_proto` at L294); crypto callback (AES-128 via `av_aes_alloc`/`av_aes_init`/`av_aes_crypt` chain at L714+, key URI prepended to segment URLs as `crypto:` pseudo-protocol at L2755); external process hooks (`hls_segment_filename` templating, `use_localtime` strftime expansion at L274-L290); fMP4 init file resend (`hls_init_file_resend:L2362`); HTTP DELETE for `hls_delete_segments` (`http_delete` field at L262); MPEG-TS sub-muxer interface (`avformat_write_header` on `vs->avf:L134`); fMP4 sub-muxer interface (`avformat_write_header` on `vs->avf` with movflag config); Sample Encryption decrypt path (`ff_hls_senc_decrypt_frame` at `hls_sample_encryption.h:L61`) |
| `docs/hls-pipeline/api-contracts/functional-invariants.md` | CREATE | `libavformat/hlsplaylist.c`, `libavformat/hlsenc.c`, RFC 8216 reference at `libavformat/hls.c:L26` | Zero-deviation checklist (no length limit). Items: M3U8 spec compliance (EXTM3U on line 1, EXT-X-VERSION on line 2 — `hlsplaylist.c:L36-L37`); EXT-X-VERSION negotiation rules (versions 2/3/4/6/7 per `hlsenc.c:L1551-L1571`); segment naming (`POSTFIX_PATTERN "_%d"` at L75 produces `<basename>_<N>.ts`); EXT-X-DISCONTINUITY placement (sequence start when HLS_DISCONT_START; before next segment when `vs->discontinuity` set); PTS/DTS passthrough (no timestamp rewrite — the HLS muxer is a meta-muxer, the sub-muxer owns timestamps); extradata injection (`AVFMT_GLOBALHEADER` flag at L3199 forces extradata in `codec_par`); EXT-X-TARGETDURATION ≥ max segment duration (L1585-L1586); EXT-X-ENDLIST emitted on VOD ONLY when not `HLS_OMIT_ENDLIST` (`hlsplaylist.c:L201-L205`); EXT-X-INDEPENDENT-SEGMENTS only when `vs->has_video` (L1597-L1599); EXT-X-I-FRAMES-ONLY only when HLS_I_FRAMES_ONLY flag and version 4 enforced (L1561); EXT-X-MAP only when `segment_type=fmp4` (`hlsplaylist.c:L134-L142`); EXT-X-BYTERANGE only when byterange_mode (`hlsplaylist.c:L162-L165`); EXT-X-KEY METHOD=AES-128 line precedes the segment(s) it encrypts (L1603-L1610); fMP4 forces version 7 (L1569-L1571); EXT-X-PLAYLIST-TYPE only when explicitly set (EVENT or VOD at `hlsplaylist.c:L122-L128`); EXT-X-ALLOW-CACHE only when `allowcache != -1` (`hlsplaylist.c:L114-L119`); CRLF normalization (M3U8 lines end with `\n` per all `avio_printf` calls). Cross-references the same line locators used in `data-contracts.md` |
| `docs/hls-pipeline/api-contracts/data-contracts.md` | CREATE | All struct + AVOption + enum definitions | Full reference, table format. Tables for: every AVOption (47 muxer + N demuxer at L3121-L3181 — name, AV_OPT_TYPE, default, min, max, unit, encoder/decoder flag); timestamp unit conventions (`AV_TIME_BASE_Q` used in `av_compare_ts:L2502`, `MPEG_TIME_BASE = 90000` for demuxer at `hls.c:L56`, `MPEG_TIME_BASE_Q` at `hls.c:L57`, segment duration tracked in `double` seconds via `dpp` accumulation, EXT-X-TARGETDURATION emitted as integer seconds); codec extradata format (sub-muxer responsibility, asserted by `AVFMT_GLOBALHEADER`); byterange offsets (`pos`/`size` int64_t in `HLSSegment:L84-L85`, emitted as `<size>@<offset>\n` at `hlsplaylist.c:L164`); encryption IV derivation (when `iv` option absent: random_seed-based derivation in `hls_encryption_start`; else hex-coded user value, 32 hex chars → 16 bytes per KEYSIZE at L72); HLSCryptoContext binary layout (32 bytes key+IV at `hls_sample_encryption.h:L42-L46`); HLSAudioSetupInfo binary layout (codec_id, codec_tag, priming, version, length, payload at `hls_sample_encryption.h:L48-L55`); STREAM_TYPE_HLS_SE_* enumeration values (0xdb, 0xcf, 0xc1, 0xc2 at `mpegts.h:L242-L246`); KEYSIZE 16 constant (L72); 4-mode StartSequenceSourceType integer values 0..3 (L58-L63); 16-bit HLSFlags packed as uint32_t `flags` field (L205); FFOutputFormat field assignments at L3191-L3207 (name, long_name, extensions, codecs, flags, priv_class, flags_internal, priv_data_size, callbacks) |
| `docs/hls-pipeline/api-contracts/timing-dependencies.md` | CREATE | `hls_write_packet:L2410-L2691`, `hls_window:L1531-L1660`, `hls_append_segment:L1042-L1288` | 2–3 pages plus a Mermaid `sequenceDiagram` showing ordering constraints. Constraints: keyframe detection precedes segment cut decision (`can_split` computed at L2473-L2475 before pts comparison at L2502); segment file must be fully written before playlist update (`av_write_frame(oc, NULL)` flush + `avio_flush` + `flush_dynbuf` precede `hls_window` at L2628); EXT-X-TARGETDURATION computed from accumulated segment durations BEFORE first playlist write (L1583-L1586 within hls_window — for the very first publish, target equals first segment's duration); temp-file rename atomicity (HLS_TEMP_FILE writes `<name>.tmp` then renames — must complete before player reads, see `hls_rename_temp_file:L1300`); fMP4 init segment written via `hls_init_file_resend:L2362` BEFORE the first media segment is referenced from playlist; AES-128 key MUST be installed via `hls_encryption_start` BEFORE the first segment of a key-period is written (L714 invocation precedes segment start); EXT-X-DISCONTINUITY line in playlist precedes the file entry for the first post-discontinuity segment (placement enforced by `ff_hls_write_file_entry` at `hlsplaylist.c:L155-L157`); demuxer ID3 timestamp parsing precedes packet emission (`is_id3_timestamped` check in `hls.c`). All diagrams use fenced `mermaid` code blocks. |
| `docs/hls-pipeline/api-contracts/integration-contracts.md` | CREATE | `hls_encryption_start:L714`, `hlsenc_io_open:L292`, `ff_hls_write_init_file:L134`, `ff_hls_write_stream_info:L78` (in hlsplaylist.c) | One page per contract. Contracts: AES-128 key URI fetch (URI format from `hls_enc_key_url` option at L3138; 16-byte raw binary key contract); fMP4 initialization segment delivery (single init.mp4 referenced by EXT-X-MAP — `hls_fmp4_init_filename` default `"init.mp4"` at L3143; resend triggered by `hls_fmp4_init_resend` option at L3144); HTTP chunked transfer behavior (default PUT method at L334-L335 unless `method` option overrides; `http_persistent` reuses TCP connection via `ff_http_do_new_request` at L304); variant-stream BANDWIDTH annotation (per RFC 8216 §4.4.5.2 — emitted at `hlsplaylist.c:L93` as `EXT-X-STREAM-INF:BANDWIDTH=%d` with computed `avg_bandwidth`/`max_bitrate` from `VariantStream` accumulators at L172-L175); HLS Sample Encryption transport (`STREAM_TYPE_HLS_SE_VIDEO_H264 = 0xdb`, `_AUDIO_AAC = 0xcf`, `_AUDIO_AC3 = 0xc1`, `_AUDIO_EAC3 = 0xc2` at `mpegts.h:L242-L246` — these are the contracts a Sample-Encryption-capable demuxer must recognize); EXT-X-KEY METHOD line contract (METHOD=AES-128 with URI and optional IV at L1603-L1610); HTTP DELETE for old segments (`http_delete` AVIOContext at L262); EXT-X-MEDIA AUDIO/SUBTITLE rendition format (`ff_hls_write_audio_rendition` at `hlsplaylist.c:L40`, `ff_hls_write_subtitle_rendition` at L58 — preserved verbatim for downstream player compatibility); program-date-time format (`#EXT-X-PROGRAM-DATE-TIME:%s.%03d%s\n` at `hlsplaylist.c:L191` — ISO 8601 with millisecond precision) |
| `[doc/muxers.texi:L1887+]` | REFERENCE | (existing FFmpeg texinfo HLS reference) | Used by `consumer-dependencies.md` and `inputs-outputs.md` as the canonical user-facing description of each HLS muxer option. The new documents must remain consistent with this reference for option names, default values, and short descriptions. NOT modified by this task. |
| `[libavformat/Makefile]` | REFERENCE | (build dependency) | L189 (`dashenc` reuses `hlsplaylist.o`) and L276-L277 (HLS demuxer/muxer object set) are cited in `consumer-dependencies.md` to surface the cross-format coupling. NOT modified. |
| `[libavformat/allformats.c:L216-L217]` | REFERENCE | (format registration) | The `extern const FFInputFormat ff_hls_demuxer; extern const FFOutputFormat ff_hls_muxer;` registration line is cited in `pipeline-orchestration.md` and `data-contracts.md`. NOT modified. |
| `[libavformat/avformat.h:L186-L188]` | REFERENCE | (public API contract) | The avformat write API summary (`avformat_write_header`/`av_write_frame`/`av_interleaved_write_frame`/`av_write_trailer`) is cited in `pipeline-orchestration.md`. NOT modified. |

All 13 user-named files plus the implicit README.md index are confirmed CREATE. No files are scheduled for DELETE. No source code files are modified (zero production impact per user mandate).

### 0.5.2 New Documentation Files Detail

For each of the fourteen CREATE entries below, the per-file blueprint specifies the executive summary topic, the technical-detail section list, the diagrams (when required), and the key source citations. The "Sections" lists are sequence-locked — these are the sub-headings the documentation file MUST contain.

#### 0.5.2.1 docs/hls-pipeline/README.md

- **Type:** Index / Entry Point
- **Source Code:** none — index only
- **Sections:**
    - Title and Commit Anchor banner ("All references in this set are anchored to commit 566ad786")
    - Three-Layer Map (links to functionality/, technical/, api-contracts/)
    - Reading Order — Integrator Path (Layer 1 → Layer 3)
    - Reading Order — Engineer Port-Scoping Path (Layer 1 functional-inventory → Layer 2 in sequence → Layer 3)
    - Citation Format Specification ("Source citations appear inline as `[path:Lstart-Lend]`")
    - Cross-Document Dependency Notes
    - Glossary of HLS-specific terms (AVERROR(*), EXT-X-*, M3U8, fMP4)
- **Diagrams:** none required
- **Key Citations:** set-wide preamble only

#### 0.5.2.2 docs/hls-pipeline/functionality/functional-inventory.md

- **Type:** Functional Component Inventory
- **Source Code:** `libavformat/hlsenc.c`, `libavformat/hls.c`, `libavformat/hlsplaylist.c`, `libavformat/hls_sample_encryption.c`
- **Sections** (one section per component, 1–2 pages each):
    - Overview (purpose, audience, plain-language summary)
    - Component: Segment Generation (plain-language → technical detail; cites `hlsenc.c:L2410`, `L2470-L2502`, `L1675`)
    - Component: Playlist Construction (cites `hlsenc.c:L1531`, `hlsplaylist.c:L110-L132`, `L201-L205`)
    - Component: Encryption Handling (cites `hlsenc.c:L714`, `hls_sample_encryption.h:L42-L62`)
    - Component: Variant Stream Management (cites `hlsenc.c:L120-L194`, options at `L3164-L3166`)
    - Component: Closed Captions Track (cites `hlsenc.c:L196-L200`, `L1400`)
    - Component: Live vs VOD Mode Selection (cites `hlsplaylist.h:L32-L36`, options at `L3169-L3171`)
    - Component: Discontinuity Handling (cites `hlsenc.c:L1594`, `L1273-L1275`, `L1622-L1623`)
    - Component: HLS Demuxer Probe (cites `hls.c:L2814-L2849`)
    - Component: HLS Demuxer Playlist Parser (cites `hls.c:L863-L979`, `L2144-L2545`)
    - Component: HLS Sample Encryption (cites `hls_sample_encryption.c`, `mpegts.h:L242-L246`)
- **Diagrams:** none required (process flows live in `process-flows.md`)
- **Key Citations:** `hlsenc.c`, `hls.c`, `hlsplaylist.c`, `hls_sample_encryption.c`

#### 0.5.2.3 docs/hls-pipeline/functionality/inputs-outputs.md

- **Type:** I/O Tables
- **Source Code:** `hlsenc.c:L3121-L3181` (options), `hlsplaylist.c` (tag writers), `hls.c` playlist parser
- **Sections** (every I/O captured, no summarizing):
    - Overview (table conventions, what counts as an input vs output)
    - Per-Component Input Table (AVPacket fields used; AVFormatContext fields used; every AVOption with default and bounds — 47 entries)
    - Per-Component Output Table (TS segment writes; fMP4 init+segment writes; every EXT-X-* tag emitted; sub-muxer av_write_frame callbacks; HTTP PUT/DELETE)
    - HLS Demuxer Input Table (every M3U8 line type accepted at `hls.c:L863-L979`)
    - HLS Demuxer Output Table (AVPacket fields populated)
    - `hls_key_info_file` Format (3-line text file: key URI, key file path, optional IV — cites `hls_encryption_start:L714`)
- **Diagrams:** none required
- **Key Citations:** `hlsenc.c`, `hlsplaylist.c`, `hls.c`

#### 0.5.2.4 docs/hls-pipeline/functionality/consumer-dependencies.md

- **Type:** Downstream Dependency Map
- **Source Code:** `libavformat/Makefile:L189`, `libavformat/allformats.c:L216-L217`, HLS spec
- **Sections** (3–5 pages):
    - Overview
    - Media Player Dependencies (what HLS-compliant players require: EXT-X-VERSION-negotiated playlist shape, contiguous EXT-X-MEDIA-SEQUENCE, EXT-X-TARGETDURATION accuracy)
    - CDN Ingest Dependencies (HTTP PUT default, persistent connection behavior, `hls_base_url`)
    - Test Suite Dependencies (survey of HLS references in `tests/` tree — read-only, per scope)
    - Downstream libavformat Consumer: DASH Muxer (cites `libavformat/Makefile:L189` showing `dashenc.o` reuses `hlsplaylist.o`)
    - Breakage Analysis: what would fail if `hlsenc.c` stopped producing valid M3U8 (`segment_muxer` at `libavformat/segment.c:L1107` is an alternative but emits different conventions)
- **Diagrams:** none required
- **Key Citations:** `Makefile`, `allformats.c`, `segment.c`

#### 0.5.2.5 docs/hls-pipeline/functionality/exception-handling.md

- **Type:** Failure Scenario Catalog
- **Source Code:** `hlsenc.c` (129 return-AVERROR sites), `hls.c` (68 return-AVERROR sites)
- **Sections** (2–4 pages, organized by failure scenario):
    - Overview
    - Bad Input Packet (`AVERROR(EINVAL)` / `AVERROR(ENOMEM)` at `hlsenc.c:L278`, `L583`, `L587`, `L681`, `L744`, `L749`)
    - Missing Keyframe (silent: `can_split=0` short-circuit at `hlsenc.c:L2477-L2479`; no AVERROR returned)
    - I/O Failures: file open / write / close (AVERROR via `hlsenc_io_open` at `L292`; suppressed by `ignore_io_errors` flag at `L3175`)
    - Mid-Stream Discontinuities (`vs->discontinuity` propagation through `hls_append_segment` at `L1062-L1066`)
    - Clock Discontinuities (`discont_program_date_time` recompute at `L1273-L1275`)
    - Segment Write Failures (`flush_dynbuf` failure at `L2611`; `hls_rename_temp_file` failure at `L1300`)
    - HTTP DELETE Failures (when `HLS_DELETE_SEGMENTS` active)
    - Demuxer Broken Playlist (`m3u8_hold_counters` retry loop)
    - Demuxer Key Fetch Failure (key URI fetch error)
    - AVERROR Code Reference Table (every distinct AVERROR observed → scenario → recovery)
- **Diagrams:** none required
- **Key Citations:** `hlsenc.c`, `hls.c`

#### 0.5.2.6 docs/hls-pipeline/technical/process-flows.md

- **Type:** End-to-End Data Flow Diagrams
- **Source Code:** `hls_write_packet:L2410`, `hls_window:L1531`, `hls_delete_old_segments:L531`, `hls_encryption_start:L714`
- **Sections** (one Mermaid diagram per major process + 1 page narrative each):
    - Overview
    - Process: Segment Generation (Mermaid `flowchart LR`; cites `L2410-L2691`)
    - Process: Playlist Update (Mermaid `flowchart LR`; cites `L1531-L1660`)
    - Process: Live Sliding Window (Mermaid `flowchart LR`; cites `L531-L713` and `L1122` `max_nb_segments` check)
    - Process: Encryption Key Rotation (Mermaid `flowchart LR`; cites `L714`, `L112` `HLS_PERIODIC_REKEY`)
    - Interaction Diagram: AVFormatContext ⇄ AVIOContext ⇄ Segment File Writer (Mermaid `flowchart TB`; cites `hlsenc.c:L292-L331`, `L2503-L2520`)
- **Diagrams:** 5 Mermaid flowcharts, all fenced as `mermaid` code blocks
- **Key Citations:** `hlsenc.c` throughout

#### 0.5.2.7 docs/hls-pipeline/technical/codec-logic.md

- **Type:** Exhaustive Decision Logic Reference
- **Source Code:** All branches across `hlsenc.c`, `hls.c`, `hlsplaylist.c`
- **Sections** (decision tables only, no pseudocode, no length cap):
    - Overview (decision-table column convention)
    - Decision: HLS_VERSION Negotiation (6 conditions × 5 outcomes — `L1551-L1571`)
    - Decision: Segment Cut (keyframe flag + `HLS_SPLIT_BY_TIME` + pts comparison — `L2470-L2502`)
    - Decision: SEGMENT_TYPE Selection (MPEG-TS vs fMP4 — `L116-L119`; downstream effects)
    - Decision: EXT-X-TARGETDURATION Computation (max-with-`lrint` — `L1585-L1586`)
    - Decision: EXT-X-DISCONTINUITY Placement (DISCONT_START — `L1594`; mid-stream — `L1273`)
    - Decision: EXT-X-PROGRAM-DATE-TIME Injection (`HLS_PROGRAM_DATE_TIME` flag — `L106`; rolling recompute — `L1273-L1275`, `L1622-L1623`)
    - Decision: Byterange Mode (`HLS_SINGLE_FILE` OR `max_seg_size > 0` — `L2503`; forces version 4 — `L1557-L1559`)
    - Decision: Start Sequence Source (4 modes — `L58-L63`)
    - Decision: Second-Level Filename Templating (%d/%t/%s — `L107-L109`)
    - Decision: HTTP Method (PUT default — `L334-L336`)
    - Decision: Periodic Rekey Cadence (`HLS_PERIODIC_REKEY` — `L112`)
    - Decision: EXT-X-INDEPENDENT-SEGMENTS Emission (`vs->has_video` AND `HLS_INDEPENDENT_SEGMENTS` — `L1597-L1599`)
    - Decision: EXT-X-I-FRAMES-ONLY Emission (`HLS_I_FRAMES_ONLY` — `L1561`, version pinned to 4)
    - Decision: EXT-X-MAP Emission (`segment_type=fmp4` — `hlsplaylist.c:L134-L142`)
    - Decision: EXT-X-BYTERANGE Emission (byterange_mode — `hlsplaylist.c:L162-L165`)
    - Decision: EXT-X-ENDLIST Emission (last AND NOT `HLS_OMIT_ENDLIST` — `L1626`)
    - Decision: `hls_init_time` Activation (sequence < init_list_dur threshold — `L2455-L2461`)
    - Decision: append_list Mode (disables `init_time` — `L3104-L3105`)
    - Decision: Demuxer Key Type Resolution (`KEY_NONE` / `KEY_AES_128` / `KEY_SAMPLE_AES` — `hls.c:L71-L74`)
- **Diagrams:** none (decision tables only — per user mandate "use decision tables, not pseudocode")
- **Key Citations:** `hlsenc.c`, `hls.c`, `hlsplaylist.c` throughout

#### 0.5.2.8 docs/hls-pipeline/technical/data-model.md

- **Type:** Full Field Dictionary
- **Source Code:** All structs in scope plus referenced `AVFormatContext`/`AVStream`/`AVOutputFormat`/`AVOption` fields
- **Sections** (every field documented, no summarizing):
    - Overview
    - Struct: HLSContext (muxer) — every field at `L202-L268`
    - Struct: VariantStream — every field at `L120-L194`
    - Struct: HLSSegment — every field at `L80-L96`
    - Struct: ClosedCaptionsStream — every field at `L196-L200`
    - Struct: HLSContext (demuxer) — every field at `hls.c:L204+`
    - Struct: playlist (demuxer) — every field at `hls.c:L101-L172`
    - Struct: segment (demuxer) — every field at `hls.c:L77-L87`
    - Struct: variant (demuxer) — every field at `hls.c:L194-L204`
    - Struct: rendition (demuxer) — every field at `hls.c:L184-L191`
    - Struct: HLSCryptoContext — every field at `hls_sample_encryption.h:L42-L46`
    - Struct: HLSAudioSetupInfo — every field at `hls_sample_encryption.h:L48-L55`
    - Enum: HLSFlags — all 16 values at `L99-L114`
    - Enum: SegmentType — both values at `L116-L119`
    - Enum: StartSequenceSourceType — all 4 values at `L58-L63`
    - Enum: PlaylistType (muxer-side) — all values at `hlsplaylist.h:L32-L36`
    - Enum: PlaylistType (demuxer-side) — all values at `hls.c:L92-L96`
    - Enum: KeyType (demuxer) — all values at `hls.c:L71-L74`
    - Constants: `KEYSIZE 16` (`L72`), `LINE_BUFFER_SIZE` (`L73`), `HLS_MICROSECOND_UNIT 1000000` (`L74`), `BUFSIZE 16384` (`L75`), `POSTFIX_PATTERN "_%d"` (`L76`)
    - Constants: `HLS_MAX_ID3_TAGS_DATA_LEN 138`, `HLS_MAX_AUDIO_SETUP_DATA_LEN 10` (`hls_sample_encryption.h:L37-L38`)
    - Constants: `MPEG_TIME_BASE 90000` (`hls.c:L56`), `MPEG_TIME_BASE_Q` (`hls.c:L57`)
    - MPEG-TS Stream Type Constants Used by HLS: `STREAM_TYPE_HLS_SE_VIDEO_H264 0xdb`, `_AUDIO_AAC 0xcf`, `_AUDIO_AC3 0xc1`, `_AUDIO_EAC3 0xc2` (`mpegts.h:L242-L246`)
    - Referenced AVFormatContext fields (from `avformat.h`, only those actually read/written by HLS code)
    - Referenced AVStream fields
    - Referenced AVOutputFormat fields (`.p.name`, `.p.long_name`, `.p.extensions`, etc.)
    - Referenced AVOption fields (from `opt.h`)
    - Referenced AVDictionary entries (from `dict.h`)
- **Diagrams:** optional Mermaid `classDiagram` showing `HLSContext` → `VariantStream` → `HLSSegment` ownership
- **Key Citations:** `hlsenc.c`, `hls.c`, `hls_sample_encryption.h`, `mpegts.h`, `avformat.h`, `opt.h`, `aes.h`, `dict.h`

#### 0.5.2.9 docs/hls-pipeline/technical/pipeline-orchestration.md

- **Type:** Lifecycle and Callback Chain Description
- **Source Code:** `hls_init:L2866`, `hls_write_header:L2301`, `hls_write_packet:L2410`, `hls_write_trailer:L2727`, `hls_deinit:L2693`
- **Sections** (3–5 pages plus Mermaid dependency diagram):
    - Overview
    - Lifecycle Entry Points (FFOutputFormat callback assignment at `L3201-L3206`)
    - Phase: init — `hls_init` at `L2866` (variant-stream allocation, segment-filename validation, options parsing)
    - Phase: write_header — `hls_write_header` at `L2301` (per-variant `avformat_write_header` on child mux, `codec_attr` computation at `L2336`, `AVMEDIA_TYPE_SUBTITLE` branching)
    - Phase: write_packet — `hls_write_packet` at `L2410` (lookup variant + child format context; segment-cut decision; segment finalize; `hls_window` publish)
    - Phase: write_trailer — `hls_write_trailer` at `L2727` (final flush per variant, final `hls_window` publish, master playlist publish)
    - Phase: deinit — `hls_deinit` at `L2693`
    - Internal Callback Chain (`hls_write_packet` → `hls_append_segment` → `sls_flags_filename_process` → `hls_window` → `hls_start`)
    - Segment Finalization Sequence (`av_write_frame(NULL)` at `L2503` → `avio_close_dyn_buf` for fMP4 at `L2508` → `avio_flush` → `hlsenc_io_close` → `hls_append_segment`)
    - Playlist Flush Timing (every segment publish triggers `hls_window` for the affected variant)
    - Restart / Recovery Behavior (`ignore_io_errors` flag at `L3175` — failures swallowed; otherwise propagated)
    - Demuxer Lifecycle (`hls_probe` → `hls_read_header` → `hls_read_packet` loop → `hls_close` at `hls.c`)
    - Child Muxer Relationship (`VariantStream::avf:L134` child `AVFormatContext` — meta-muxer pattern)
- **Diagrams:** Mermaid `flowchart TB` for lifecycle DAG; Mermaid `classDiagram` for child-muxer ownership (both fenced as `mermaid` code blocks)
- **Key Citations:** `hlsenc.c`, `hls.c`, `avformat.h:L186-L188`

#### 0.5.2.10 docs/hls-pipeline/technical/integration-interfaces.md

- **Type:** External Interface Reference
- **Source Code:** `hlsenc_io_open:L292`, `hlsenc_io_close:L313`, `set_http_options:L333`, `hls_encryption_start:L714`, `libavutil/aes.h`, `libavformat/hls_sample_encryption.h`
- **Sections** (one page per interface, structured consistently):
    - Overview
    - Interface: File AVIOContext Writes
    - Interface: HTTP AVIOContext Writes (with persistent-connection reuse via `http_persistent` at `L302-L304`)
    - Interface: Protocol Handlers (`file://`, `http://`, `https://`, `crypto:`)
    - Interface: `ff_is_http_proto` Detection (`L294`)
    - Interface: AES-128 Crypto Pipeline (`av_aes_alloc` / `av_aes_init` / `av_aes_crypt` — `aes.h:L40-L62`)
    - Interface: HLS Sample Encryption Pipeline (`ff_hls_senc_decrypt_frame` — `hls_sample_encryption.h:L61`)
    - Interface: `hls_segment_filename` Templating (`replace_str_data_in_filename` at `hlsenc.c:L376`)
    - Interface: `use_localtime` strftime Expansion (`strftime_expand` at `hlsenc.c:L270`)
    - Interface: MPEG-TS Sub-Muxer (`avformat_write_header` on `VariantStream::avf` when `segment_type=mpegts`)
    - Interface: fMP4 Sub-Muxer (`avformat_write_header` on `VariantStream::avf` when `segment_type=fmp4`)
    - Interface: HTTP DELETE for `hls_delete_segments` (`http_delete` at `L262`)
    - Interface: fMP4 Initialization Segment Resend (`hls_init_file_resend` at `L2362`)
- **Diagrams:** none required (per-interface structured pages)
- **Key Citations:** `hlsenc.c`, `hls_sample_encryption.h`, `mpegts.h`, `aes.h`

#### 0.5.2.11 docs/hls-pipeline/api-contracts/functional-invariants.md

- **Type:** Zero-Deviation Behavior Checklist
- **Source Code:** `hlsplaylist.c` (canonical line-level emissions), `hlsenc.c` (conditional emissions), RFC 8216
- **Sections** (checklist, no length limit):
    - Overview ("These behaviors MUST be identical across any refactor or port")
    - Invariant: M3U8 Header Order (`EXTM3U` on line 1, `EXT-X-VERSION` on line 2 — `hlsplaylist.c:L36-L37`)
    - Invariant: EXT-X-VERSION Negotiation (5 outcomes — `L1551-L1571`)
    - Invariant: Segment Naming Convention (`POSTFIX_PATTERN "_%d"` — `L75`)
    - Invariant: EXT-X-DISCONTINUITY Placement
    - Invariant: PTS/DTS Passthrough Semantics (sub-muxer owns timestamps)
    - Invariant: Extradata Injection (`AVFMT_GLOBALHEADER` — `L3199`)
    - Invariant: EXT-X-TARGETDURATION ≥ max segment duration (`L1585-L1586`)
    - Invariant: EXT-X-ENDLIST emission conditional on NOT `HLS_OMIT_ENDLIST`
    - Invariant: EXT-X-INDEPENDENT-SEGMENTS conditional on `vs->has_video`
    - Invariant: EXT-X-I-FRAMES-ONLY pins version to 4
    - Invariant: EXT-X-MAP emitted only for fMP4
    - Invariant: EXT-X-BYTERANGE emitted only for byterange mode
    - Invariant: EXT-X-KEY METHOD=AES-128 line precedes encrypted segment entries
    - Invariant: fMP4 forces version 7
    - Invariant: EXT-X-PLAYLIST-TYPE only when explicitly set
    - Invariant: EXT-X-ALLOW-CACHE only when `allowcache != -1`
    - Invariant: LF (not CRLF) line terminator on all playlist writes
- **Diagrams:** none required
- **Key Citations:** `hlsplaylist.c`, `hlsenc.c`

#### 0.5.2.12 docs/hls-pipeline/api-contracts/data-contracts.md

- **Type:** Full Data Contract Reference
- **Source Code:** All struct + AVOption + enum definitions used at the API boundary
- **Sections** (full reference, table format, every contract — no summarizing):
    - Overview
    - Contract: HLSContext Field Formats (every public-effect field)
    - Contract: VariantStream Field Formats
    - Contract: HLSSegment Field Formats
    - Contract: AVOption Table (47 muxer options — name, type, default, min, max, unit)
    - Contract: Timestamp Unit Conventions (segment duration in `double` seconds; TARGETDURATION as integer seconds; `MPEG_TIME_BASE 90000` for demuxer)
    - Contract: Codec Extradata Format (sub-muxer responsibility under `AVFMT_GLOBALHEADER`)
    - Contract: Byterange Offset/Size (`int64_t` pair; `<size>@<offset>\n` emission)
    - Contract: Encryption IV Derivation (`KEYSIZE 16` hex bytes; absent IV → random_seed-derived)
    - Contract: HLSCryptoContext Binary Layout
    - Contract: HLSAudioSetupInfo Binary Layout
    - Contract: STREAM_TYPE_HLS_SE_* values (0xdb, 0xcf, 0xc1, 0xc2)
    - Contract: FFOutputFormat field assignments (`L3191-L3207`)
- **Diagrams:** none required
- **Key Citations:** `hlsenc.c`, `hls_sample_encryption.h`, `mpegts.h`

#### 0.5.2.13 docs/hls-pipeline/api-contracts/timing-dependencies.md

- **Type:** Processing-Order Contract Reference
- **Source Code:** `hls_write_packet:L2410-L2691`, `hls_window:L1531-L1660`, `hls_append_segment:L1042-L1288`
- **Sections** (2–3 pages plus Mermaid sequencing diagram):
    - Overview
    - Constraint: Keyframe Detection Precedes Segment-Cut (`L2473-L2475` → `L2502`)
    - Constraint: Segment File Fully Written Before Playlist Update (flush → window publish at `L2628`)
    - Constraint: EXT-X-TARGETDURATION Computed Before First Segment is Emitted
    - Constraint: Temp-File Atomic Rename (`HLS_TEMP_FILE`)
    - Constraint: fMP4 Init Segment Written Before First Media Segment Referenced
    - Constraint: AES-128 Key Installed Before Encrypted Segment Written
    - Constraint: EXT-X-DISCONTINUITY Line Precedes Affected Segment Entry
    - Constraint: ID3 Timestamp Parsed Before Demuxer Emits First Packet
    - Sequence Diagram: End-to-End Ordering (Mermaid `sequenceDiagram`, fenced as `mermaid` code block)
- **Diagrams:** 1 Mermaid `sequenceDiagram`
- **Key Citations:** `hlsenc.c`, `hlsplaylist.c`, `hls.c`

#### 0.5.2.14 docs/hls-pipeline/api-contracts/integration-contracts.md

- **Type:** External-System Interface Contract Reference
- **Source Code:** `hls_encryption_start:L714`, `hlsenc_io_open:L292`, `ff_hls_write_init_file`, `ff_hls_write_stream_info`
- **Sections** (one page per contract, structured consistently):
    - Overview
    - Contract: AES-128 Key URI Fetch (URI format; 16-byte raw binary key contract)
    - Contract: fMP4 Initialization Segment Delivery (default filename `"init.mp4"`; resend on m3u8 refresh)
    - Contract: HTTP Chunked Transfer (PUT default; method override; `http_persistent` connection reuse)
    - Contract: Variant-Stream BANDWIDTH Annotation (`EXT-X-STREAM-INF:BANDWIDTH=%d` format)
    - Contract: HLS Sample Encryption Transport (0xdb/0xcf/0xc1/0xc2 stream-type values)
    - Contract: EXT-X-KEY METHOD Line Layout (METHOD=AES-128, URI, optional IV=0x...)
    - Contract: HTTP DELETE for Old Segment Cleanup
    - Contract: EXT-X-MEDIA Audio/Subtitle Rendition Format
    - Contract: EXT-X-PROGRAM-DATE-TIME ISO-8601 Format (millisecond precision)
- **Diagrams:** none required (per-contract structured pages)
- **Key Citations:** `hlsenc.c`, `hlsplaylist.c`, `hls_sample_encryption.h`, `mpegts.h`, `aes.h`

### 0.5.3 Documentation Files to Update Detail

No existing documentation files are updated. All thirteen user-named files plus the README index are net-new CREATE operations. The existing FFmpeg documentation (Texinfo `[doc/muxers.texi:L1887]`, file-level Doxygen blocks) remains unchanged.

### 0.5.4 Documentation Configuration Updates

No changes to any documentation build configuration:

- `[doc/Doxyfile]` — NOT modified. Doxygen scope unchanged.
- `[doc/Makefile]` — NOT modified. Manpage/POD/HTML build flow unchanged.
- `[doc/t2h.pm]`, `[doc/t2h.init]` — NOT modified.
- No `mkdocs.yml`, `docusaurus.config.js`, `.readthedocs.yml`, or `sphinx/conf.py` exists in this repository; none is added. The new markdown tree is consumed by readers via plain markdown rendering.
- No `package.json` or `requirements.txt` is added — the documentation is dependency-free at the consumption side.

### 0.5.5 Cross-Documentation Dependencies

Logical dependencies between the new documents (used to plan generation order and cross-references):

- `data-model.md` is referenced by every Layer 2 and Layer 3 document for field definitions. It is the canonical struct dictionary.
- `process-flows.md` and `pipeline-orchestration.md` share lifecycle assertions; one diagram-driven, the other narrative-driven. Both are cited by `timing-dependencies.md`.
- `functional-invariants.md` cites the same line locators as `data-contracts.md` and `integration-contracts.md`.
- `exception-handling.md` sources its AVERROR list from grep-able sites in `hlsenc.c` and `hls.c`; the resulting recovery descriptions are reused (without restating) inside `integration-contracts.md` where appropriate.
- The README index links to all thirteen leaves and articulates the commit anchor convention once.

No additional table-of-contents, glossary, or index file is required beyond the README — the glossary fits inside the README index section.


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

This documentation effort is consumption-side dependency-free at the markdown rendering layer. The only authoring-side tool that materially affects deliverable shape is the Mermaid renderer, which is required because the user mandate states that all diagrams MUST use Mermaid syntax in fenced `mermaid` code blocks. Standard GitHub-flavored markdown rendering (or any compatible viewer that natively renders fenced `mermaid` blocks) is sufficient for downstream consumers. The repository already ships with a Mermaid CLI binary on the build host (`mmdc 10.6.1` confirmed via `mmdc --version`).

| Registry | Package Name | Version | Status | Purpose |
|----------|--------------|---------|--------|---------|
| npm | `@mermaid-js/mermaid-cli` | 10.6.1 | Already available on host | Validate fenced `mermaid` blocks parse correctly; render preview PNG/SVG if exporting (not required for shipped markdown) |
| system | `node` | 22.22.2 | Already installed | Runtime for `mmdc` |
| system | `npm` | 11.1.0 | Already installed | Package manager (no new installs needed) |
| system | `python3` | 3.12.3 | Already installed | Optional helper for batch markdown linting / link-check |
| system | `git` | from FFmpeg build host | Already installed | Anchor verification (`git rev-parse HEAD` confirms `566ad786…`) |

No new package installations are required. No new dependency manifests (`package.json`, `requirements.txt`, `pyproject.toml`, etc.) are added by this work. The deliverables remain pure markdown with embedded Mermaid blocks that render natively on GitHub, GitLab, Bitbucket, and any compliant static-site generator.

Existing repository documentation infrastructure that is **NOT** used by this task (and therefore requires no version pinning here, but is documented for completeness so reviewers know what tooling exists):

| Registry | Package Name | Version | Status | Why Not Used |
|----------|--------------|---------|--------|--------------|
| system | `doxygen` | 1.8.8 (per `[doc/Doxyfile]`) | Present in repo config | The repository's Doxygen pipeline targets C source code documentation. Our deliverables are stand-alone markdown narratives outside that pipeline. |
| system | `makeinfo` / `texi2html` | as bundled (Texinfo) | Present | Used by `[doc/muxers.texi]` and other `.texi` files; we do not author Texinfo. |
| system | `perl` | as bundled | Present | Used by `[doc/t2h.pm]`; not used by markdown deliverables. |

### 0.6.2 Source-Code Dependencies for Documentation Extraction

The HLS pipeline source code is the sole source of truth for every claim. The fourteen in-scope source files (no version pin needed — anchored to commit `566ad786`) are reproduced here for traceability:

| Path | Lines | Role in Extraction |
|------|-------|---------------------|
| `libavformat/hlsenc.c` | 3,207 | Primary muxer source — extract HLSContext, VariantStream, HLSSegment, options, every EXT-X-* emission |
| `libavformat/hls.c` | 2,912 | Primary demuxer source — extract demuxer HLSContext, playlist/segment/variant/rendition structs, parser |
| `libavformat/hlsplaylist.c` | 206 | Playlist tag-writer source — extract every `avio_printf` line emission |
| `libavformat/hlsplaylist.h` | 65 | Extract `PlaylistType` enum, writer prototypes |
| `libavformat/hls_sample_encryption.c` | 396 | Sample-encryption transform source — extract `ff_hls_senc_*` functions |
| `libavformat/hls_sample_encryption.h` | 65 | Extract `HLSCryptoContext`, `HLSAudioSetupInfo`, constants, prototypes |
| `libavformat/segment.c` | 1,136 | Generic segment muxer (referenced for breakage analysis) |
| `libavformat/mpegtsenc.c` | 2,424 | MPEG-TS muxer (referenced as sub-muxer for `segment_type=mpegts`) |
| `libavformat/mpegts.c` | 3,735 | MPEG-TS demuxer (referenced as sub-demuxer; provides HLS sample-encryption stream-type constants) |
| `libavformat/mpegts.h` | 306 | Extract `STREAM_TYPE_HLS_SE_*` (`L242-L246`) |
| `libavformat/avformat.h` | 3,163 | Public API contract source for `avformat_write_header` / `av_write_frame` / `av_interleaved_write_frame` / `av_write_trailer` / `AVFormatContext` / `AVStream` / `AVOutputFormat` field references |
| `libavutil/opt.h` | 1,194 | AVOption infrastructure — referenced for option type semantics in `data-contracts.md` |
| `libavutil/aes.h` | 69 | AES interface — `av_aes_alloc:L40`, `av_aes_init:L50`, `av_aes_crypt:L61` |
| `libavutil/dict.h` | 242 | AVDictionary API — referenced where `hls_segment_options` (`AV_OPT_TYPE_DICT`) propagates to sub-muxer |

Auxiliary references that are read but not modified:

| Path | Locator | Role |
|------|---------|------|
| `libavformat/Makefile` | `:L189` (DASH→hlsplaylist reuse), `:L276-L277` (HLS object set) | Cited in `consumer-dependencies.md` for cross-format coupling |
| `libavformat/allformats.c` | `:L216-L217` | Cited in `pipeline-orchestration.md` / `data-contracts.md` for format registration |
| `doc/muxers.texi` | `:L1887` (start of `@section hls`) | Cited as canonical user-facing option description for `consumer-dependencies.md` and `inputs-outputs.md` |
| `doc/Doxyfile` | (existence noted) | Cited in 0.2 Documentation Discovery; not consumed by our markdown |

### 0.6.3 Documentation Reference Updates

No documentation link transformations are required. The new tree is self-contained under `docs/hls-pipeline/` and uses relative links between its own files. No existing markdown, README, or Texinfo file is updated to reference the new tree (per scope: "Comments and documentation only. No refactoring, optimization, or interface changes" — adding an entry to an existing `README.md` would constitute editing an out-of-scope file). The new `[docs/hls-pipeline/README.md]` is the entry point; consumers reach it via the documented path.

Link conventions inside the new tree:

- Inter-document links use relative paths from the document's own location (e.g., from `functionality/exception-handling.md` to `technical/codec-logic.md` the relative path is `../technical/codec-logic.md`).
- Source-code citations use the inline `path:line` form `[<repo-relative-path>:L<start>-L<end>]` immediately after the claim they support (per user's traceability rule and the AAP's "Citation discipline" rule from the section prompt).
- The commit anchor is stated once at the top of `[docs/hls-pipeline/README.md]` and inherited by every leaf document by reference.

### 0.6.4 Summary of Dependency Changes

No dependency additions, updates, or removals are introduced by this work at the project's package-manifest level. The deliverables are pure markdown text. Authoring-side tooling (`mmdc` for optional Mermaid validation, `git` for commit-anchor verification) is already present on the build host. The repository's existing Doxygen and Texinfo pipelines remain untouched.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

Current coverage analysis (anchored to commit `566ad786`):

| Coverage Dimension | Current State | Target | Gap |
|--------------------|---------------|--------|-----|
| Public-API entry points documented at engineer/integrator level (`hls_init`, `hls_write_header`, `hls_write_packet`, `hls_write_trailer`, `hls_deinit`, `hls_probe`, `hls_read_header`, `hls_read_packet`, `hls_close`, `hls_read_seek`) | 0% engineer-facing; sparse `/**` Doxygen blocks on a subset only | 100% — every entry point documented in `functional-inventory.md`, `pipeline-orchestration.md`, `inputs-outputs.md` | 10/10 |
| AVOption coverage in muxer (47 options at `[libavformat/hlsenc.c:L3121-L3181]`) | Texinfo-side coverage exists at `[doc/muxers.texi:L1887+]` but is integrator-facing only | 100% — every option documented field-level in `data-contracts.md` and `inputs-outputs.md` | 47/47 |
| AVOption coverage in demuxer | Texinfo-side coverage exists; engineer-facing none | 100% — every demuxer option documented | every option |
| HLSFlags enum coverage (16 values at `[libavformat/hlsenc.c:L99-L114]`) | 0% engineer-facing | 100% — every flag has semantics + branch citations in `codec-logic.md` and `data-model.md` | 16/16 |
| SegmentType enum coverage (2 values at `L116-L119`) | 0% engineer-facing | 100% — both values documented with branch-on-segment-type behavior in `codec-logic.md` | 2/2 |
| StartSequenceSourceType coverage (4 values at `L58-L63`) | 0% engineer-facing | 100% — all four modes documented in `codec-logic.md` and `data-model.md` | 4/4 |
| Demuxer KeyType coverage (3 values at `[libavformat/hls.c:L71-L74]`) | 0% engineer-facing | 100% — all three values documented | 3/3 |
| Struct field coverage — muxer HLSContext (60+ fields at `L202-L268`) | 0% field-level | 100% — every field in `data-model.md` | every field |
| Struct field coverage — VariantStream (70+ fields at `L120-L194`) | 0% field-level | 100% — every field in `data-model.md` | every field |
| Struct field coverage — HLSSegment (`L80-L96`) | 0% field-level | 100% | every field |
| Struct field coverage — demuxer HLSContext / playlist / segment / variant / rendition (`hls.c:L77-L204`) | 0% field-level | 100% | every field |
| Struct field coverage — HLSCryptoContext / HLSAudioSetupInfo (`hls_sample_encryption.h:L42-L55`) | 0% field-level | 100% | every field |
| EXT-X-* tag emission coverage (≈15 tag types across `hlsplaylist.c` + `hlsenc.c`) | 0% engineer-facing emission-rule documentation | 100% — every tag's emission rule documented in `codec-logic.md` and `functional-invariants.md`, every line locator cited | every tag |
| Decision branch coverage (`codec-logic.md`) | 0% as decision tables | 100% — every documented branch represented as decision table, no pseudocode | every branch listed in 0.5.2.7 |
| Process-flow diagram coverage | 0% | 5 Mermaid flowcharts (segment generation, playlist update, live sliding window, encryption key rotation, AVFormatContext/AVIOContext interaction) | 5/5 |
| Pipeline-orchestration diagram coverage | 0% | 2 Mermaid diagrams (lifecycle DAG `flowchart TB`, ownership `classDiagram`) | 2/2 |
| Timing-dependencies diagram coverage | 0% | 1 Mermaid `sequenceDiagram` end-to-end | 1/1 |
| AVERROR scenario catalog | 0% engineer-facing | 100% — every distinct AVERROR observed in the 129 return-sites of `hlsenc.c` + 68 return-sites of `hls.c` mapped to scenario + recovery path | every distinct code |
| Integration interface coverage | 0% engineer-facing | 13 interfaces enumerated in `integration-interfaces.md`, every one with direction/format/frequency/contract | 13/13 |
| Functional invariant coverage | 0% as zero-deviation checklist | Every spec invariant listed in `functional-invariants.md` (M3U8 header order, version negotiation, naming, discontinuity placement, PTS/DTS passthrough, extradata, target-duration computation, endlist conditions, etc.) | every invariant |

Coverage targets are derived from the user's explicit length-and-completeness specifications restated in the SUCCESS CRITERIA of the user input:

- Functional Inventory: 1–2 pages per component, one section per component, no components omitted
- Inputs & Outputs Map: table format, every I/O captured — no summarizing
- Consumer Dependencies: 3–5 pages
- Exception Handling: 2–4 pages, organized by failure scenario, mapped to AVERROR codes
- Process Flows: one Mermaid diagram per major process + 1 page narrative each
- Codec/Format Logic: exhaustive — every rule, condition, branch, and threshold. No length cap.
- Data Model & Dictionary: full field-level dictionary — every field, no summarizing
- Pipeline Orchestration: 3–5 pages + Mermaid dependency diagram
- Integration Interfaces: one page per interface, structured consistently
- Functional Invariants: checklist format, no limit
- Data Contracts: full reference, table format, every contract
- Timing Dependencies: 2–3 pages + Mermaid sequencing diagram
- Integration Contracts: one page per contract, structured consistently

Per-file coverage focus areas:

- `functional-inventory.md` — focus on plain-language clarity for integrators; every component named at the top must have a 1–2 page section.
- `inputs-outputs.md` — focus on enumeration completeness; the validator for this file is "can a reader, without reading source, list every AVPacket field consumed and every byte emitted?"
- `codec-logic.md` — focus on decision-table exhaustiveness; the validator is "every conditional in the listed branches at `[libavformat/hlsenc.c]` and `[libavformat/hls.c]` appears in at least one decision table."
- `data-model.md` — focus on field-level completeness; the validator is "every field declared in the in-scope struct definitions has a row in a documentation table with name, type, size, and business meaning."
- `functional-invariants.md` — focus on zero-deviation phrasing; every item begins with "MUST" or "MUST NOT" and cites a line locator.

### 0.7.2 Documentation Quality Criteria

#### 0.7.2.1 Completeness Requirements

- Every public-API entry point in the muxer and demuxer must have a description, parameters, return-value semantics, error returns, and at least one usage scenario.
- Every user guide section must include setup (option configuration), usage (representative invocation), and troubleshooting (linked to `exception-handling.md`).
- Every architecture document must include at least one Mermaid diagram and a narrative explaining why the diagram looks the way it does.
- Every codec/format-logic claim must appear in a decision table; pseudocode is forbidden per user mandate.
- Every struct field is enumerated in `data-model.md` with name, C type, in-memory size (where determinable), and business meaning.

#### 0.7.2.2 Accuracy Validation

- Source citations: every `[<path>:L<start>-L<end>]` reference must resolve to existing lines at commit `566ad786`. Inferred claims (i.e., not directly traceable to a specific source range) must be tagged `[inferred — no direct source]` per the section prompt's "Citation discipline" rule.
- API signatures: function names cited in documentation must match the names at the cited line ranges; no renaming or normalization.
- Field names: struct field names cited in `data-model.md` and `data-contracts.md` must match the verbatim C identifier (no camelCase reformatting).
- Option defaults: every default value cited for an AVOption must match the literal default in the options array at `[libavformat/hlsenc.c:L3121-L3181]`.
- Decision branches: every branch in a decision table must correspond to an `if`/`else if`/`switch case`/ternary in the cited line range.

#### 0.7.2.3 Clarity Standards

- Plain-language-first: every section opens with an executive summary readable by a library integrator without deep FFmpeg knowledge, followed by technical detail for engineers (per user mandate).
- Progressive disclosure: simple-to-complex order within each section (overview → mechanism → edge cases → invariants).
- Consistent terminology: HLS spec terminology (segment, playlist, variant stream, rendition, target duration, discontinuity) is used uniformly; the README glossary defines each term once.
- No marketing language: no "robust", "powerful", "seamless", or similar adjectives — only descriptive and precise terms.

#### 0.7.2.4 Maintainability Requirements

- Source citations enable mechanical re-validation: a future maintainer can run `git diff 566ad786..HEAD -- <cited-file>` to identify documentation rows that may have drifted.
- Each document carries an explicit commit-anchor banner inherited from `README.md`.
- Each decision table row carries its own `[path:line]` locator so individual rows can be re-validated independently.
- Template-aligned section ordering across documents within a layer (e.g., every Layer 1 functionality doc uses the same Overview / Component / Citations skeleton).

### 0.7.3 Example and Diagram Requirements

#### 0.7.3.1 Minimum Examples per Document

| Document | Minimum Examples |
|----------|------------------|
| `functional-inventory.md` | Each component section has at least 1 representative scenario (e.g., live mode, VOD mode) |
| `inputs-outputs.md` | None (table-driven; reference, not example-driven) |
| `consumer-dependencies.md` | At least 1 worked breakage scenario per consumer category |
| `exception-handling.md` | At least 1 reproduction sketch per AVERROR scenario (input → expected error → recovery) |
| `process-flows.md` | 5 Mermaid flowcharts (one per major process) |
| `codec-logic.md` | Each decision table includes at least one input-row example illustrating the branch outcome |
| `data-model.md` | None required beyond field dictionary |
| `pipeline-orchestration.md` | Lifecycle DAG plus ownership diagram |
| `integration-interfaces.md` | Each interface page includes 1 representative invocation example (e.g., HTTP PUT URL pattern, AES key URI pattern) |
| `functional-invariants.md` | Each invariant has a "MUST" / "MUST NOT" phrasing with a concrete violation example where helpful |
| `data-contracts.md` | None beyond contract tables |
| `timing-dependencies.md` | Sequencing diagram |
| `integration-contracts.md` | Each contract page includes 1 wire-format example (M3U8 fragment, HTTP request line, etc.) |

#### 0.7.3.2 Required Diagram Types

| Document | Diagram Type | Rendering Format |
|----------|--------------|------------------|
| `process-flows.md` | `flowchart LR` ×4 + `flowchart TB` ×1 | Fenced `mermaid` blocks |
| `pipeline-orchestration.md` | `flowchart TB` + `classDiagram` | Fenced `mermaid` blocks |
| `data-model.md` | `classDiagram` (optional, ownership view) | Fenced `mermaid` blocks |
| `timing-dependencies.md` | `sequenceDiagram` | Fenced `mermaid` blocks |
| All others | none required | n/a |

Mermaid is mandatory for every diagram per user mandate. ASCII art, PlantUML, Graphviz dot, or rendered PNG/SVG embeds are NOT used.

#### 0.7.3.3 Code-Example Testing

- Worked example AVOption strings are validated by manual inspection against the options array at `[libavformat/hlsenc.c:L3121-L3181]`.
- M3U8 fragments cited as wire-format examples are validated against the `avio_printf` format strings at `[libavformat/hlsplaylist.c]` and `[libavformat/hlsenc.c]`.
- No example code is compiled or executed (no source-code modification, no test additions per user scope).

#### 0.7.3.4 Visual Content Freshness

- Diagrams are anchored to commit `566ad786`. Any subsequent code change that adds a new lifecycle phase or process branch necessitates a diagram refresh; this is tracked by citing the diagram source lines explicitly in the narrative immediately following the diagram.
- The README banner makes the anchor explicit: any documentation maintainer reading the docs after a code change is alerted by the commit difference between the banner and `HEAD`.

### 0.7.4 Acceptance Criteria Summary

Documentation is considered complete when:

- All 14 markdown files (13 user-named + README index) exist at the prescribed paths.
- Every coverage table row in 0.7.1 reads 100% (or the documented target count is met).
- Every diagram in 0.7.3.2 is rendered as a fenced `mermaid` code block and parses successfully with `mmdc --input <file> --output /tmp/preview.svg` (validation step performed by the author; not a shipping deliverable).
- Every `[<path>:L<start>-L<end>]` citation in the deliverable resolves to non-empty content at commit `566ad786`.
- Every inferred claim is tagged `[inferred — no direct source]` per the section prompt's citation-discipline rule.
- No source-code file is modified (verified via `git diff 566ad786 -- libavformat/ libavutil/` returning an empty patch for tracked files outside the new docs path).


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

#### 0.8.1.1 New Documentation Files (CREATE)

The following markdown files are created. The user-named list of thirteen plus the implicit README index is the entirety of the deliverable surface:

- `docs/hls-pipeline/README.md` (index + commit-anchor banner + glossary)
- `docs/hls-pipeline/functionality/functional-inventory.md`
- `docs/hls-pipeline/functionality/inputs-outputs.md`
- `docs/hls-pipeline/functionality/consumer-dependencies.md`
- `docs/hls-pipeline/functionality/exception-handling.md`
- `docs/hls-pipeline/technical/process-flows.md`
- `docs/hls-pipeline/technical/codec-logic.md`
- `docs/hls-pipeline/technical/data-model.md`
- `docs/hls-pipeline/technical/pipeline-orchestration.md`
- `docs/hls-pipeline/technical/integration-interfaces.md`
- `docs/hls-pipeline/api-contracts/functional-invariants.md`
- `docs/hls-pipeline/api-contracts/data-contracts.md`
- `docs/hls-pipeline/api-contracts/timing-dependencies.md`
- `docs/hls-pipeline/api-contracts/integration-contracts.md`

Trailing wildcard pattern: `docs/hls-pipeline/**/*.md` is the in-scope CREATE surface, but only the fourteen explicit files above are created — no other files match the pattern under this delivery.

#### 0.8.1.2 Documentation File Updates (UPDATE)

None. No existing markdown, Texinfo, Doxygen, or README file is updated.

#### 0.8.1.3 Documentation Configuration Files

None. No `mkdocs.yml`, `docusaurus.config.js`, `.readthedocs.yml`, `sphinx/conf.py`, or `package.json` is added; the existing `[doc/Doxyfile]` and `[doc/Makefile]` are not modified.

#### 0.8.1.4 Documentation Assets

None. No image, SVG, PNG, or diagram-source asset is added. All diagrams are embedded as fenced `mermaid` code blocks inside the markdown files. No `docs/images/`, `docs/examples/`, or `docs/assets/` directory is created.

#### 0.8.1.5 Documentation Generation

None. No build script, diagram-generation configuration, or API-doc-generation setting is added. The deliverables are consumed by readers via plain markdown rendering (GitHub, GitLab, or any compliant viewer).

#### 0.8.1.6 In-Scope Source-Code Read Surface

The following source files are READ (never modified) to derive every documented claim. Reads are restricted to commit `566ad786`:

- `libavformat/hlsenc.c` (full file — 3,207 lines)
- `libavformat/hls.c` (full file — 2,912 lines)
- `libavformat/hlsplaylist.c` (full file — 206 lines)
- `libavformat/hlsplaylist.h` (full file — 65 lines)
- `libavformat/hls_sample_encryption.c` (full file — 396 lines)
- `libavformat/hls_sample_encryption.h` (full file — 65 lines)
- `libavformat/segment.c` (read for cross-format breakage analysis — 1,136 lines)
- `libavformat/mpegtsenc.c` (read for sub-muxer interface — 2,424 lines)
- `libavformat/mpegts.c` (read for sub-demuxer interface — 3,735 lines)
- `libavformat/mpegts.h` (`STREAM_TYPE_HLS_SE_*` constants — 306 lines)
- `libavformat/avformat.h` (read for `AVFormatContext` / `AVStream` / `AVOutputFormat` / `AVInputFormat` field semantics — 3,163 lines)
- `libavutil/opt.h` (AVOption infrastructure — 1,194 lines)
- `libavutil/aes.h` (AES public interface — 69 lines)
- `libavutil/dict.h` (AVDictionary public interface — 242 lines)

Auxiliary reads (cross-reference only, never modified):

- `libavformat/Makefile` (`L189`, `L276-L277`)
- `libavformat/allformats.c` (`L216-L217`)
- `doc/muxers.texi` (`L1887+` as cross-reference)
- `doc/Doxyfile` (existence inspection)

#### 0.8.1.7 Anchor and Traceability Mandate

Every cited line range MUST be valid at commit `566ad786`. The README banner declares this anchor once; every leaf document inherits it. The trailing wildcard pattern `[**/*]:L<start>-L<end>` is acceptable only when the line range is verifiable at the anchor.

### 0.8.2 Explicitly Out of Scope

#### 0.8.2.1 Source-Code Modifications

- No edits to any C source file (`.c`) — including no addition of `/** Doxygen */` blocks, no inline comments, no refactor, no rename, no reformat. The user mandate states "All existing production logic and behavior — document as-is. No refactoring, optimization, or interface changes. Comments and documentation only."
- No edits to any C header file (`.h`).
- No edits to any inline-assembly or SIMD file.

#### 0.8.2.2 Test Suite Modifications

- No edits to any file under `tests/` — including no new fate tests, no test-reference updates, no test-script modification. The user input lists `tests/ — test harnesses and fate reference data` as out of scope.

#### 0.8.2.3 Feature and Interface Changes

- No new features, no new AVOptions, no new EXT-X-* tag emissions, no new HLS spec compliance levels.
- No code refactoring, no struct field renames, no enum reordering, no function signature changes.
- No deprecation, removal, or visibility change of any public symbol.

#### 0.8.2.4 Deployment and Build Changes

- No changes to `configure`, `Makefile`, `library.mak`, `libavformat/Makefile`, `libavformat/allformats.c`, or any other build configuration.
- No CI configuration changes (no edits to `.github/`, no `.gitlab-ci.yml` changes, no fate runner configuration).
- No `ffbuild/`, `compat/`, or `presets/` modifications.

#### 0.8.2.5 Out-of-Scope Source Subtrees (User-Specified)

The following subtrees are explicitly out of scope per user input and are NOT read for documentation extraction:

- Hardware acceleration backends: `libavcodec/vulkan*`, `libavfilter/vf_*_vulkan.c`, CUDA/OpenCL filter implementations
- Platform-specific SIMD optimizations: `libavcodec/x86/`, `libavcodec/aarch64/`, `libavcodec/arm/`
- All other container formats in `libavformat/` not directly called by the HLS pipeline (e.g., `mov.c`, `matroska.c`, `flacenc.c`, etc.)
- `tests/` — test harnesses and fate reference data
- `compat/` — compatibility shims

#### 0.8.2.6 Documentation Out of Scope

- No documentation outside the `docs/hls-pipeline/` tree.
- No documentation for unrelated FFmpeg subsystems (codec layer, filter graph, device IO, scaling, color conversion) — even if the HLS pipeline transitively depends on them at runtime.
- No translation of the new English-language documentation into other languages.
- No marketing material, release notes, or changelog entries (the user mandate says zero production impact).

### 0.8.3 Untouchable

The following are explicitly Untouchable per user input — read only when strictly necessary to document the HLS pipeline's interface to them, never modified, never used as a basis for any modification:

- External codec libraries linked via `ffbuild/`: `libx264`, `libvpx`, `libaom`, `libfdk-aac`, and all third-party encoder/decoder wrappers (`libavcodec/*_wrapper.c`) — code that is not owned by this project and will not be rewritten under any circumstance.
- All existing production logic and behavior of the HLS pipeline itself — document as-is. The mandate is comments-and-documentation only. No refactoring, optimization, or interface change of any kind, no matter how small or seemingly trivial.

### 0.8.4 Scope Validation Rules

The following machine-checkable rules are used to validate scope adherence at task completion:

| Rule | Validation Command (conceptual) | Expected Result |
|------|---------------------------------|-----------------|
| No source code modified | `git diff 566ad786..HEAD -- libavformat/ libavutil/ libavcodec/ libavfilter/ libavdevice/ libswresample/ libswscale/ libpostproc/ tools/ tests/ compat/ ffbuild/ presets/ configure Makefile` | Empty patch |
| No build config modified | `git diff 566ad786..HEAD -- '*.mak' Makefile configure 'libavformat/allformats.c'` | Empty patch |
| New tree exists | `find docs/hls-pipeline -name '*.md' \| wc -l` | 14 (13 leaves + README) |
| All required files exist | Per-path `test -f` on each of the 14 paths in 0.8.1.1 | All pass |
| No additional files in tree | `find docs/hls-pipeline -type f ! -name '*.md'` | Empty result |
| Every citation has valid anchor | Mechanical grep + line-range verification against commit 566ad786 | All resolve |

Scope adherence is verified before marking the documentation effort complete.


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

This effort produces stand-alone markdown documentation that does not require a build pipeline. The execution parameters below capture the conventions any documentation generator agent must follow.

#### 0.9.1.1 Source Anchor Verification

Before any citation is written, the agent confirms it is reading source at commit `566ad786`:

- `git -C <repo-root> rev-parse HEAD` MUST yield a hash beginning with `566ad7869ee3c8b6993e1f880e0a50eae18c66ac` (verified during environment setup).
- If the working tree is at a different commit, the agent MUST checkout commit `566ad786` before proceeding (read-only operation, no commit modifications).

#### 0.9.1.2 Output Format Conventions

- Default format: GitHub-flavored markdown with Mermaid diagrams (per user mandate).
- Heading levels: `#` for document title, `##` for top-level sections, `###` for sub-sections, `####` for sub-sub-sections. No use of `#####` or deeper.
- Line endings: LF (`\n`) only. No CRLF in output.
- Encoding: UTF-8 without BOM.
- Tables: pipe-delimited with header row and separator row; no HTML tables.
- Lists: dash (`-`) for unordered bullets; numerals (`1.`, `2.`, …) for ordered lists only when sequence is semantically meaningful.
- Inline code: single backticks `` `like_this` `` for identifiers, function names, file paths.
- Fenced code blocks: triple-backtick with explicit language (`mermaid`, `text`, `c` where short illustrative code snippets are used — limited to two or three lines per spec rules).

#### 0.9.1.3 Citation Format

- Inline file:line citation: `[<repo-relative-path>:L<start>-L<end>]` placed immediately after the claim it supports. Single-line citations use `[<path>:L<n>]`.
- Key-path citations for config files: `[<path>:key.subkey]` (used only for the optional Doxyfile / Makefile cross-references — the in-scope source is all C).
- Section citations: `[<path>:§<heading>]` for Texinfo (used when citing `[doc/muxers.texi:§hls]`).
- Inferred citations: `[inferred — no direct source]` per section prompt's citation-discipline rule.
- Commit-anchor banner: the literal string "All references in this set are anchored to commit 566ad786" appears at the top of `README.md` and is implicitly inherited by every leaf document.

#### 0.9.1.4 Diagram Conventions

- Every diagram MUST be Mermaid in a fenced `mermaid` code block (per user mandate).
- Diagram element captions reference `[<path>:L<n>]` either inside the diagram (as node text) or in the narrative immediately following the diagram.
- Supported Mermaid diagram types used: `flowchart LR`, `flowchart TB`, `sequenceDiagram`, `classDiagram`. No other diagram types (e.g., `gantt`, `pie`, `gitGraph`) are appropriate for this content.
- Diagram validation: the agent may render with `mmdc --input <md-file> --output /tmp/<diagram>.svg` for syntax verification; the rendered output is NOT shipped. Only the source markdown is the deliverable.

#### 0.9.1.5 Decision-Table Convention

- Every codec/format logic claim MUST be expressed as a decision table (per user mandate). Pseudocode is forbidden.
- Decision-table column convention: input conditions in left columns, outcome in rightmost column; each row independently captures one branch.
- Source citation in every decision-table row (rightmost column or appended note).

#### 0.9.1.6 Plain-Language-First Convention

- Every section opens with an executive summary readable without deep FFmpeg knowledge.
- Technical detail follows the executive summary in a clearly demarcated sub-section.
- Library integrators (Layer 1 readers) should be able to read the executive summaries of every Layer 1 document and understand what the pipeline does without reading any technical detail.

#### 0.9.1.7 Information Extraction Process

For each documented claim, the extraction process is:

1. Identify the source-of-truth source file and line range using the file inventory in 0.6.2.
2. Read the relevant source range using `read_file` (full file for the first read of each file, scoped ranges thereafter).
3. Translate the C-level mechanism into plain language for the executive summary.
4. Translate the same mechanism into engineer-facing technical detail.
5. Append the `[<path>:L<n>-L<m>]` citation.
6. If the claim is not directly traceable, tag with `[inferred — no direct source]`.

### 0.9.2 Build and Preview Commands

#### 0.9.2.1 Documentation Build Command

No build command. The deliverables are plain markdown rendered by the consumer's markdown viewer (GitHub UI, GitLab UI, IDE preview, etc.). No static-site generator is configured or invoked.

#### 0.9.2.2 Documentation Preview Command

- Local preview (optional, for the documentation author): any markdown viewer (e.g., `python3 -m http.server` over a directory containing the markdown plus a generic Mermaid-aware viewer; or VS Code Markdown Preview Mermaid Support extension; or pushing to a feature branch and using GitHub's renderer).
- Mermaid block validation (optional): `mmdc --input docs/hls-pipeline/<file>.md --output /tmp/<file>.svg` parses each fenced `mermaid` block; non-zero exit indicates a syntax error.

#### 0.9.2.3 Diagram Generation Command

Mermaid diagrams are embedded as markdown source, not pre-rendered. No diagram-generation step is required at build time. The Mermaid renderer in the consumer's viewer renders on demand.

#### 0.9.2.4 Documentation Deployment Command

None. The new tree lives in the repository and is consumed directly. No deployment pipeline change.

### 0.9.3 Validation Commands

| Validation | Command | Expected Result |
|------------|---------|-----------------|
| All 14 files exist | `for f in docs/hls-pipeline/README.md docs/hls-pipeline/functionality/*.md docs/hls-pipeline/technical/*.md docs/hls-pipeline/api-contracts/*.md; do test -f "$f" || echo "MISSING: $f"; done` | No output (every file present) |
| File count | `find docs/hls-pipeline -name '*.md' \| wc -l` | `14` |
| No non-markdown files in tree | `find docs/hls-pipeline -type f ! -name '*.md'` | Empty |
| No source-code changes | `git diff 566ad786..HEAD -- libavformat/ libavutil/` | Empty patch |
| No build-config changes | `git diff 566ad786..HEAD -- '*.mak' configure Makefile 'libavformat/allformats.c'` | Empty patch |
| Every Mermaid block parses | `for f in $(find docs/hls-pipeline -name '*.md'); do mmdc -i "$f" -o /tmp/check.svg 2>&1; done` | All blocks parse cleanly |
| Markdown structure valid | `python3 -c "import markdown; markdown.markdown(open('<file>').read())"` per file | No exceptions |
| Internal links resolve | Manual check or `markdown-link-check` (optional) | All resolve |
| Citation anchors valid | Mechanical grep for `[<path>:L<n>-L<m>]` and verify each path/range exists at commit `566ad786` | All resolve |

The validation suite is run by the documentation author before declaring the work complete. No automated CI hook is added (out of scope).

### 0.9.4 Style Guide

- Style: technical reference style — precise, descriptive, no marketing language, no narrative voice changes between sections.
- Person: third-person plural / impersonal ("the muxer", "the pipeline", "the demuxer") — not "you" or "we".
- Tense: present tense for describing current behavior ("the muxer writes…"), past tense only for historical context.
- Terminology consistency: the README glossary is the canonical source. Every term defined in the glossary appears with consistent capitalization throughout (e.g., "EXT-X-VERSION" is always all-caps with hyphens, never "Ext-X-Version" or "ext_x_version").
- Acronym discipline: every acronym is spelled out on first use in each document (e.g., "HLS (HTTP Live Streaming)"), then abbreviated.
- File-path consistency: repository-relative paths always start at the repository root (`libavformat/hlsenc.c`, never `./libavformat/hlsenc.c` or `/path/to/repo/libavformat/hlsenc.c`).

### 0.9.5 Default Parameter Inventory

Documentation-author defaults applied throughout the deliverable:

| Parameter | Default | Source / Rationale |
|-----------|---------|--------------------|
| Output format | Markdown | User mandate "Markdown" |
| Diagram syntax | Mermaid in fenced `mermaid` blocks | User mandate "All diagrams MUST use Mermaid syntax" |
| Logic expression style | Decision tables | User mandate "Use decision tables for codec/format logic — not pseudocode" |
| Section opening | Plain-language executive summary | User mandate "Each section opens with a plain-language executive summary" |
| Citation form | `[path:Lstart-Lend]` inline | Section prompt "Citation discipline (PRIMARY)" |
| Anchor commit | `566ad786` | User mandate "All source references MUST be anchored to commit 566ad786" |
| Code-snippet length | ≤ 2-3 lines | Master execution rule "Keep your code snippet examples short and brief, not exceeding 2-3 lines" |
| Inline link form | Relative within `docs/hls-pipeline/` | Self-contained tree |
| Heading depth | Up to `####` (4 levels) | Standard markdown / readability |
| Diagram render check | `mmdc --input … --output /tmp/…` | Optional pre-ship validation |


## 0.10 Rules for Documentation

The rules below are derived from the user's prompt. They are the binding constraints for every documentation generation step. Each rule is presented with its origin, technical meaning, and downstream effect on the deliverable.

### 0.10.1 Commit Anchor Rule

- **Rule:** Every source reference (file path, line number) MUST be anchored to commit `566ad786`. The line numbers cited in this Agent Action Plan and in every leaf documentation file refer to the state of the repository at that exact commit.
- **Origin:** User input — "All source references (file paths, line numbers) MUST be anchored to commit 566ad786 (current HEAD at documentation time). Prepend each code reference block with the commit hash."
- **Technical Meaning:** The agent operates with the working tree at commit `566ad786`. The commit-anchor banner appears once in `[docs/hls-pipeline/README.md]` and is inherited by reference; every leaf document need not repeat the hash inline, but every line citation MUST be valid at that commit.
- **Downstream Effect:** If the source code changes after this documentation is produced, the documentation does not auto-update; downstream readers who need current line numbers re-anchor by computing `git diff 566ad786..HEAD -- <cited-file>`.

### 0.10.2 Mermaid-Only Rule

- **Rule:** Every diagram MUST use Mermaid syntax in fenced `mermaid` code blocks for consistent markdown rendering.
- **Origin:** User input — "All diagrams MUST use Mermaid syntax (fenced `mermaid` blocks) for consistent markdown rendering."
- **Technical Meaning:** No ASCII art, no PlantUML, no rendered PNG/SVG embeds, no Graphviz dot, no external image references. All diagrammatic content is text inside `mermaid` fenced blocks.
- **Downstream Effect:** Any consumer with a Mermaid-aware markdown viewer renders the diagrams natively. The diagrams remain editable as plain text and diffable in version control.

### 0.10.3 Decision-Table Rule (No Pseudocode)

- **Rule:** Every codec/format logic and business-logic rule, condition, threshold, or branch MUST be expressed as a decision table. Pseudocode is forbidden.
- **Origin:** User input — "Use decision tables for codec/format logic — not pseudocode."
- **Technical Meaning:** `codec-logic.md` is structured entirely around decision tables with inputs in left columns and outcomes in right columns. The same rule extends to any logic exposition in other documents.
- **Downstream Effect:** A reader scanning a decision table can immediately enumerate every branch and every outcome. Pseudocode, by contrast, hides edge cases behind control flow; the user explicitly rejects it.

### 0.10.4 Plain-Language-First Rule

- **Rule:** Every section MUST open with a plain-language executive summary readable by a library integrator without deep FFmpeg knowledge, followed by technical detail for engineers.
- **Origin:** User input — "Each section opens with a plain-language executive summary (readable by a library integrator without deep FFmpeg knowledge), followed by technical detail (for engineers), with code references traceable to source — file paths and line numbers, not just function names."
- **Technical Meaning:** Each section has two clearly demarcated tiers: a plain-language tier (no FFmpeg jargon, no struct field names, no AVERROR codes) and a technical tier (full source citations).
- **Downstream Effect:** Layer 1 readers can skim executive summaries across the entire tree and form a system mental model without any C-level detail. Layer 2 and Layer 3 readers can drop into technical detail wherever they need.

### 0.10.5 Traceability Rule

- **Rule:** Every documented behavior, rule, and interface is traceable to specific source file paths and line numbers — not just function names.
- **Origin:** User input — "Code references traceable to source — file paths and line numbers, not just function names." Reinforced by SUCCESS CRITERIA — "Every documented behavior, rule, and interface is traceable to specific source file paths and line numbers, anchored to commit 566ad786."
- **Technical Meaning:** Citations use the form `[<repo-relative-path>:L<start>-L<end>]`. Function-name-only references (e.g., "see `hls_write_packet`") are insufficient — they must be accompanied by `[libavformat/hlsenc.c:L2410]`.
- **Downstream Effect:** A reader can `git show 566ad786:libavformat/hlsenc.c | sed -n '2410,2691p'` and arrive at the exact source the documentation claim is derived from.

### 0.10.6 Zero-Impact Rule

- **Rule:** No modifications to existing production logic or behavior. Comments and documentation only.
- **Origin:** User input — "Zero production impact — No modifications to existing logic or behavior. Comments and documentation only." Reinforced by SCOPE — "All existing production logic and behavior — document as-is. No refactoring, optimization, or interface changes."
- **Technical Meaning:** No edits to any `.c`, `.h`, `.cpp`, `.S`, `.asm`, `.mak`, `Makefile`, `configure`, `*.texi`, `Doxyfile`, or any non-markdown file under the entire repository. Even inline `/** Doxygen */` blocks in C source are NOT added under this delivery.
- **Downstream Effect:** A clean `git diff 566ad786..HEAD -- libavformat/ libavutil/` returns an empty patch. The HLS muxer/demuxer binary output is bit-identical to commit `566ad786`. Risk of regression is zero by construction.

### 0.10.7 Format-Hierarchy Rule

- **Rule:** Organize by functional component first, then technical detail.
- **Origin:** User input — "Organize by functional component first, then technical detail."
- **Technical Meaning:** Within each three-layer directory (functionality, technical, api-contracts), documents are organized by component (segment generation, playlist construction, encryption, variant streams, etc.) and within each component by depth (overview → behavior → invariants → contracts).
- **Downstream Effect:** Readers can navigate by feature ("I need to understand encryption") without having to traverse multiple unrelated topics.

### 0.10.8 Length-and-Completeness Rules

- **Rule:** Specific length and completeness targets per document (see 0.7.1 for the full list).
- **Origin:** User input — SUCCESS CRITERIA / "Length and completeness targets".
- **Technical Meaning:** The author respects the per-document length budgets where the user specified them (e.g., 1–2 pages per component, 3–5 pages for consumer dependencies). For documents where the user said "exhaustive" or "no limit" (codec-logic, data-model, functional-invariants), no truncation is permitted; every branch / field / invariant is captured.
- **Downstream Effect:** Documents that are length-capped remain skimmable; documents that are exhaustive serve as authoritative reference.

### 0.10.9 No-Summarizing Rule

- **Rule:** For inputs/outputs, data-model, and data-contracts documents, every I/O / field / contract is captured — no summarizing.
- **Origin:** User input — "Capture every I/O — no summarizing." / "Full dictionary — every field, no summarizing." / "Full reference, table format. Every contract — no summarizing."
- **Technical Meaning:** The author does NOT group related fields into "see X" or "and similar"; every field gets its own row.
- **Downstream Effect:** These documents are large by design but serve as authoritative tables.

### 0.10.10 Source-Code-As-Truth Rule

- **Rule:** The HLS pipeline source code at commit `566ad786` is the sole source of truth. No external speculation. Inferred claims (where source does not directly state a behavior but it can be reasonably deduced) are tagged `[inferred — no direct source]`.
- **Origin:** Section prompt — "Citation discipline (PRIMARY) … inferred claims are permitted but flagged so downstream stages can verify them before relying on them."
- **Technical Meaning:** Every claim about the existing system carries either a `[path:line]` citation or an `[inferred]` tag. Where the documentation describes downstream consumer expectations that are not in the in-scope source (e.g., "media players expect contiguous EXT-X-MEDIA-SEQUENCE"), the claim is anchored to RFC 8216 (referenced in `[libavformat/hls.c:L26]`) or tagged inferred.
- **Downstream Effect:** Future maintainers can mechanically verify every citation; inferred claims are flagged for re-validation when the surrounding source changes.

### 0.10.11 Scope-Adherence Rule

- **Rule:** Strictly observe the In Scope / Out of Scope / Untouchable classifications enumerated in 0.8.1, 0.8.2, and 0.8.3.
- **Origin:** User input — SCOPE section with explicit In scope / Out of scope / Untouchable lists.
- **Technical Meaning:** No source file outside the In Scope list is read for documentation extraction; no file outside the deliverable surface (the 14 new markdown files under `docs/hls-pipeline/`) is created or modified.
- **Downstream Effect:** The blast radius of the documentation effort is bounded to exactly the new markdown tree.

### 0.10.12 README-Reading-Order Rule

- **Rule:** The README index defines two reading orders — an integrator path (Layer 1 then Layer 3) and an engineer port-scoping path (Layer 1 functional-inventory then full Layer 2 then full Layer 3). Both are explicit.
- **Origin:** Inferred from the user's stated audience — "The primary audience is a mixed team of library integrators … and engineers planning to refactor, extend, or port subsystems."
- **Technical Meaning:** `[docs/hls-pipeline/README.md]` includes a Reading Order section enumerating the recommended document sequence for each audience.
- **Downstream Effect:** Readers self-select into the appropriate path without having to read the entire tree.

### 0.10.13 Highest-Risk Identification Rule

- **Rule:** Highest-risk and highest-complexity areas MUST be explicitly identified before any rewriting begins.
- **Origin:** User input — SUCCESS CRITERIA: "Risk identification — Highest-risk and highest-complexity areas (e.g., live sliding window logic, encryption key rotation, fMP4 vs. TS format selection branches) are explicitly identified before any rewriting begins."
- **Technical Meaning:** Each Layer 2 document (especially `codec-logic.md`, `process-flows.md`, `pipeline-orchestration.md`) calls out the high-risk areas with a "Risk" or "Complexity" annotation. The user-named candidates — live sliding window, encryption key rotation, fMP4 vs TS branch selection — are highlighted with explicit risk markers.
- **Downstream Effect:** A port-scoping reader can identify the top-N highest-effort modules without reading every line of source.

### 0.10.14 Two-Audience Validation Rule

- **Rule:** Documentation must be (a) confirmable by engineers familiar with HLS and FFmpeg internals and (b) usable by library integrators without deep FFmpeg knowledge.
- **Origin:** User input — SUCCESS CRITERIA: "Engineer validation — Blitzy produces a functional description of the pipeline that engineers familiar with the HLS spec and FFmpeg internals can read and confirm is accurate." + "Porting scoping readiness".
- **Technical Meaning:** Each section is dual-tier (plain language + technical detail). Engineers validate technical detail; integrators read executive summaries.
- **Downstream Effect:** A single deliverable serves both audiences without forking into two separate documentation sets.

### 0.10.15 Source-Citation Density Rule

- **Rule:** Source citations are dense enough to be useful, sparse enough to remain readable. A typical paragraph carries 1–3 citations; a typical decision-table row carries exactly one.
- **Origin:** Section prompt — "for every claim … include an inline citation".
- **Technical Meaning:** Citations cluster around the claims they support; redundant citations are avoided when the same line range supports multiple consecutive claims (cite once, then reference by proximity).
- **Downstream Effect:** Readers can verify any claim without being overwhelmed by citation chrome.

### 0.10.16 Mermaid-Only Caveat for Sequence Constraints

- **Rule:** Where the documentation describes ordering or timing constraints, the diagram MUST be a Mermaid `sequenceDiagram`, not a `flowchart`. This preserves the temporal semantics.
- **Origin:** Section prompt — "All diagrams MUST use Mermaid syntax." User input on `timing-dependencies.md` specifies "sequencing diagram".
- **Technical Meaning:** `timing-dependencies.md` uses `sequenceDiagram` (one or more). `process-flows.md` uses `flowchart LR/TB` for process-data-flow diagrams.
- **Downstream Effect:** Readers can distinguish "what happens" (flowchart) from "in what order" (sequence diagram) at a glance.


## 0.11 References

### 0.11.1 Citation Discipline Summary

Every claim about the existing system in this Agent Action Plan and in every downstream deliverable carries either:

- An inline citation of the form `[<repo-relative-path>:L<start>-L<end>]` resolving to commit `566ad786`, or
- A key-path citation `[<path>:<key>]` for configuration files, or
- A section citation `[<path>:§<heading>]` for Texinfo, or
- A literal `[inferred — no direct source]` tag when the claim cannot be directly grounded.

All inline citations in this Agent Action Plan have been verified to resolve to non-empty content at commit `566ad786`. Inferred claims are flagged so downstream stages can re-validate before relying on them.

### 0.11.2 Primary Source Files Examined

These are the in-scope source files read during the discovery and analysis phases to derive the documentation plan above. All file paths are repository-relative; all line counts and key entry points are anchored to commit `566ad786`.

| Path | Lines | Notes / Key Anchors |
|------|-------|---------------------|
| `libavformat/hlsenc.c` | 3,207 | Primary HLS muxer. Entry points: `ff_hls_muxer` registration (`L3191-L3207`), callbacks `hls_init` (`L2866`), `hls_write_header` (`L2301`), `hls_write_packet` (`L2410`), `hls_write_trailer` (`L2727`), `hls_deinit` (`L2693`). Core internals: `hls_start` (`L1675`), `hls_window` (`L1531`), `hls_append_segment` (`L1042`), `hls_mux_init` (`L773`), `hls_encryption_start` (`L714`). Constants: `KEYSIZE 16` (`L72`), `LINE_BUFFER_SIZE` (`L73`), `HLS_MICROSECOND_UNIT 1000000` (`L74`), `BUFSIZE 16384` (`L75`), `POSTFIX_PATTERN "_%d"` (`L76`). Enums: `StartSequenceSourceType` (`L58-L63`), `HLSFlags` (`L99-L114`), `SegmentType` (`L116-L119`). Structs: `HLSSegment` (`L80-L96`), `VariantStream` (`L120-L194`), `ClosedCaptionsStream` (`L196-L200`), `HLSContext` (`L202-L268`). AVOptions array: `L3121-L3181`. |
| `libavformat/hls.c` | 2,912 | Primary HLS demuxer. Entry points: `ff_hls_demuxer` registration (`L2900-L2912`), callbacks `hls_probe` (`L2814`), `hls_read_header` (`L2144`), `hls_read_packet` (`L2546`), `hls_read_seek` (`L2709`), `hls_close` (`L2127`). RFC 8216 reference (`L26`). Constants: `MPEG_TIME_BASE 90000` (`L56`), `MPEG_TIME_BASE_Q` (`L57`). Enums: `KeyType` (`L71-L74`), `PlaylistType` (demuxer-side, `L92-L96`). Structs: `segment` (`L77-L87`), `playlist` (`L101-L172`), `rendition` (`L184-L191`), `variant` (`L194-L204`), `HLSContext` (`L204+`). Playlist parser: `L863-L979`. |
| `libavformat/hlsplaylist.c` | 206 | Tag-writer source. `ff_hls_write_playlist_version` (`L32`), `_audio_rendition` (`L40`), `_subtitle_rendition` (`L58`), `_stream_info` (`L78`), `_playlist_header` (`L110`), `_init_file` (`L134`), `_file_entry` (`L144`), `_end_list` (`L201`). Every EXT-X-* line emission is one `avio_printf` call inside these functions. |
| `libavformat/hlsplaylist.h` | 65 | `PlaylistType` enum (`L32-L36`: `PLAYLIST_TYPE_NONE`, `_EVENT`, `_VOD`, `_NB`). Writer prototypes. |
| `libavformat/hls_sample_encryption.c` | 396 | Sample encryption transform implementation. Implements `ff_hls_senc_*` declared in the header. |
| `libavformat/hls_sample_encryption.h` | 65 | Constants `HLS_MAX_ID3_TAGS_DATA_LEN 138` and `HLS_MAX_AUDIO_SETUP_DATA_LEN 10` (`L37-L38`). Structs: `HLSCryptoContext` (`L42-L46`), `HLSAudioSetupInfo` (`L48-L55`). Function prototypes (`L57-L62`). |
| `libavformat/segment.c` | 1,136 | Alternative generic segment muxer. `ff_segment_muxer` (`L1107`), `ff_stream_segment_muxer` (`L1123`). Referenced for breakage-analysis context only. |
| `libavformat/mpegtsenc.c` | 2,424 | MPEG-TS muxer used as a child mux when `segment_type=mpegts`. `ff_mpegts_muxer` (`L2408`). |
| `libavformat/mpegts.c` | 3,735 | MPEG-TS demuxer / sub-demuxer. `ff_mpegts_demuxer` (`L3710`). |
| `libavformat/mpegts.h` | 306 | `STREAM_TYPE_HLS_SE_VIDEO_H264 0xdb`, `STREAM_TYPE_HLS_SE_AUDIO_AAC 0xcf`, `STREAM_TYPE_HLS_SE_AUDIO_AC3 0xc1`, `STREAM_TYPE_HLS_SE_AUDIO_EAC3 0xc2` at `L242-L246`. |
| `libavformat/avformat.h` | 3,163 | Public API contract — `avformat_write_header` / `av_write_frame` / `av_interleaved_write_frame` / `av_write_trailer` summary at `L186-L188`. `AVFormatContext`, `AVStream`, `AVOutputFormat`, `AVInputFormat` definitions throughout. |
| `libavutil/opt.h` | 1,194 | AVOption infrastructure — referenced for option-type semantics in `data-contracts.md`. |
| `libavutil/aes.h` | 69 | AES public interface — `av_aes_alloc` (`L40`), `av_aes_init` (`L50`), `av_aes_crypt` (`L61`). |
| `libavutil/dict.h` | 242 | AVDictionary public interface — referenced where `hls_segment_options` (`AV_OPT_TYPE_DICT`) propagates options to the sub-muxer. |

Total in-scope source surface: approximately 19,120 lines across 14 files at commit `566ad786`.

### 0.11.3 Auxiliary Source References

Files read for cross-reference but NOT modified, and NOT a primary documentation extraction target:

| Path | Locator | Reason for Reference |
|------|---------|----------------------|
| `libavformat/Makefile` | `:L189` | DASH muxer (`dashenc.o`) reuses `hlsplaylist.o` — cross-format coupling cited in `consumer-dependencies.md`. |
| `libavformat/Makefile` | `:L276-L277` | HLS demuxer + muxer object set declaration. |
| `libavformat/allformats.c` | `:L216-L217` | `extern const FFInputFormat ff_hls_demuxer; extern const FFOutputFormat ff_hls_muxer;` — format registration cited in `pipeline-orchestration.md` and `data-contracts.md`. |
| `doc/muxers.texi` | `:L1887` (start of `@section hls`) | Canonical Texinfo user-facing description of HLS muxer options — cross-referenced in `consumer-dependencies.md` and `inputs-outputs.md` for option names and defaults. NOT modified. |
| `doc/Doxyfile` | (configuration file existence) | Doxygen 1.8.8 configuration confirms repository documentation infrastructure exists; cited once in 0.2 Documentation Discovery. NOT modified. |

### 0.11.4 External Standards Referenced

The HLS pipeline is specified in part by external standards. These are referenced (not copied) inside the deliverable:

- **RFC 8216 (HTTP Live Streaming)**: The HLS spec. The source code reference at `[libavformat/hls.c:L26]` cites it; `functional-invariants.md` and `data-contracts.md` reference it for client/spec expectations.
- **ISO/IEC 14496-12 (ISO Base Media File Format)**: Underlies the fMP4 segment format used when `segment_type=fmp4`.
- **ISO/IEC 13818-1 (MPEG-2 TS)**: Underlies the TS segment format used when `segment_type=mpegts`.
- **AES-128 / NIST FIPS 197**: Block cipher used for segment encryption.

These references are NOT shipped as additional documentation files; they are cited inline where invariants depend on them.

### 0.11.5 Existing FFmpeg Documentation Infrastructure (Inventory)

The repository ships with the following documentation infrastructure, none of which is modified:

- `doc/Doxyfile` — Doxygen 1.8.8 configuration. Targets C source. Output unaffected by this work.
- `doc/muxers.texi` — Texinfo source for `ffmpeg-muxers` manpage; HLS section at `:L1887+`.
- `doc/demuxers.texi` — Texinfo source for `ffmpeg-demuxers` manpage; HLS demuxer section exists.
- `doc/protocols.texi` — Texinfo for protocols (`http`, `https`, `file`, `crypto`).
- `doc/t2h.pm`, `doc/t2h.init` — Texinfo → HTML transformation. Not modified.
- `doc/Makefile` — Documentation build pipeline (manpages, HTML). Not modified.

The new `docs/hls-pipeline/` tree lives outside this existing infrastructure deliberately — it targets a different audience (engineer port-scoping + integrator deep understanding) than the existing Texinfo user manual.

### 0.11.6 Search Log Appendix

This is the comprehensive log of every file and folder examined to derive the documentation plan in this Agent Action Plan. Each entry records the path and the purpose of the read.

#### 0.11.6.1 Repository Root Reconnaissance

- `/tmp/blitzy/blitzy-FFmpeg/reverse-engineering-demo_3846f5/` — root listing to confirm FFmpeg layout and locate `doc/`, `libavformat/`, `libavutil/`, `tests/`, `Makefile`, `configure`.
- `git rev-parse HEAD` — verified commit `566ad786...` matches user-specified anchor.
- `.blitzyignore` search via `find` — no `.blitzyignore` file present anywhere in the tree (confirmed during Phase 4 context gathering).

#### 0.11.6.2 Documentation Infrastructure Reconnaissance

- `doc/` — listing to enumerate Texinfo, Doxyfile, Makefile, t2h transformation scripts.
- `doc/Doxyfile` — read to confirm Doxygen 1.8.8 configuration; not modified by this work.
- `doc/Makefile` — read to confirm manpage/HTML build flow; not modified.
- `doc/muxers.texi` — located `@section hls` at `:L1887`.
- `doc/demuxers.texi` — located HLS demuxer description.
- `doc/protocols.texi` — located protocol-handler descriptions.
- `README.md` (repository root) — confirmed FFmpeg project README is not modified.

#### 0.11.6.3 Source Folder Reconnaissance

- `libavformat/` — full folder listing to identify all `hls*.c`, `hls*.h`, `segment.c`, `mpegts*.c`, `mpegts.h`, `avformat.h`, `Makefile`, `allformats.c`.
- `libavutil/` — full folder listing to identify `opt.h`, `aes.h`, `dict.h`.
- `libavformat/Makefile` — examined object-set declarations at `L189`, `L276-L277` to capture build relationships.
- `libavformat/allformats.c` — examined format registration at `L216-L217`.

#### 0.11.6.4 In-Scope Source File Reads

- `libavformat/hlsenc.c` — full file (3,207 lines).
- `libavformat/hls.c` — full file (2,912 lines).
- `libavformat/hlsplaylist.c` — full file (206 lines).
- `libavformat/hlsplaylist.h` — full file (65 lines).
- `libavformat/hls_sample_encryption.c` — full file (396 lines).
- `libavformat/hls_sample_encryption.h` — full file (65 lines).
- `libavformat/segment.c` — full file (1,136 lines).
- `libavformat/mpegtsenc.c` — full file (2,424 lines).
- `libavformat/mpegts.c` — full file (3,735 lines).
- `libavformat/mpegts.h` — full file (306 lines).
- `libavformat/avformat.h` — full file (3,163 lines).
- `libavutil/opt.h` — full file (1,194 lines).
- `libavutil/aes.h` — full file (69 lines).
- `libavutil/dict.h` — full file (242 lines).

#### 0.11.6.5 Out-of-Scope Source Folders (NOT Read)

Per the user's SCOPE section and rule 0.10.11 (Scope-Adherence), the following subtrees were NOT read:

- `libavcodec/vulkan*`
- `libavfilter/vf_*_vulkan.c`
- `libavcodec/x86/`, `libavcodec/aarch64/`, `libavcodec/arm/` (SIMD)
- `libavformat/` containers other than the in-scope set (`mov.c`, `matroska.c`, `flacenc.c`, etc.)
- `tests/` (fate harnesses + reference data)
- `compat/` (compatibility shims)
- All external codec wrappers in `libavcodec/*_wrapper.c`

#### 0.11.6.6 Technical Specification Sections Retrieved

The following sections of the parent Technical Specification document were retrieved during context gathering:

- §1.2 System Overview — confirmed FFmpeg multimedia framework context.
- §1.3 Scope — confirmed HLS muxer/demuxer pipeline boundaries align with the user's stated In Scope set.
- §2.1 Feature Catalog — confirmed HLS as a documented feature.
- §5.1 High-Level Architecture — confirmed pipeline architecture context.

### 0.11.7 Attachments and External Metadata

#### 0.11.7.1 User-Provided Attachments

No file attachments were provided by the user for this task. The directory `/tmp/environments_files/` was checked and confirmed empty (no attachments to enumerate).

#### 0.11.7.2 Figma Screens

No Figma URLs or screen references were provided. The HLS pipeline is a C library / CLI subsystem with no UI surface; design-system or visual-design references are not applicable. The Design System Alignment Protocol from the section prompt is therefore N/A for this task.

#### 0.11.7.3 URLs and External References Provided in User Input

The user input does not contain external URLs. All references are internal to the FFmpeg repository at commit `566ad786`.

### 0.11.8 Environment Variables, Setup Instructions, and Secrets

- Setup instructions provided by the user: none.
- Environment variables provided by the user: none.
- Secrets provided by the user: none.
- Required runtime: the documentation effort is markdown-authoring only; the FFmpeg build environment (compilers, libraries) is not required to produce the deliverable. Optional tooling on the build host: `mmdc` 10.6.1 for Mermaid syntax validation, `python3` 3.12.3 for optional markdown structural checks, `node` 22.22.2 + `npm` 11.1.0 (host of `mmdc`), `git` for commit-anchor verification — all confirmed present.

### 0.11.9 Conflict Resolution Log

During Phase 2 (Prompt and Section Analysis), the following potential conflicts between the AAP section prompt and the user input were identified and resolved:

| Conflict | Resolution |
|----------|-----------|
| Section prompt mentions a `mkdocs.yml` / `docusaurus.config.js` / `.readthedocs.yml` documentation generator update; user input does not require a generator. | Resolved in favor of user input — no generator configuration is added (see 0.5.4 and 0.6.1). |
| Section prompt suggests Mermaid as an option among others; user input mandates Mermaid-only. | Resolved in favor of user input — Mermaid-only rule (0.10.2). |
| Section prompt allows pseudocode in some cases; user input forbids it for codec/format logic. | Resolved in favor of user input — decision-table rule (0.10.3). |
| Section prompt allows inline source citations via function names; user input requires file path and line number. | Resolved in favor of user input — traceability rule (0.10.5). |
| Section prompt allows optional Doxygen comment additions; user input prohibits any source-code modification. | Resolved in favor of user input — zero-impact rule (0.10.6). |
| Section prompt's Design System Alignment Protocol prescribes a multi-step component-library investigation; the HLS pipeline is a C library with no UI. | Marked N/A; no Design System Compliance sub-section is produced. |

All resolutions favor the user's explicit constraints over the section prompt's default options, consistent with the section prompt's own instruction to honor user-specified rules and examples exactly.


