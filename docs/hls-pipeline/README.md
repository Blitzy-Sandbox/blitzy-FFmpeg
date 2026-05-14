# FFmpeg HLS Pipeline — Engineering Documentation

A three-layer reverse-engineering documentation set covering the FFmpeg HLS (HTTP Live Streaming) muxer and demuxer pipeline, sourced from the C implementation in `libavformat/` and targeted at both library integrators and engineers planning to refactor, extend, or port the subsystem.

> **Commit Anchor:** All source references in this documentation set are anchored to commit `566ad786` (full hash `566ad7869ee3c8b6993e1f880e0a50eae18c66ac`). Line numbers cited as `[<path>:L<start>-L<end>]` are valid at this commit. Subsequent commits to the source tree may move line ranges; readers re-anchoring to a later commit can run `git diff 566ad786..HEAD -- <cited-file>` to identify rows that may have drifted.

---

## Documentation Map

The documentation is organized into three layers, each addressing a distinct question and a distinct audience.

- **Layer 1 — Functionality** (`functionality/`) answers *"What does this system do?"* and is written plain-language-first for library integrators. Engineers read it as orientation before descending into deeper layers.
- **Layer 2 — Technical** (`technical/`) answers *"How does it work under the hood?"* and is written at engineering depth for readers planning a port, rewrite, or substantial extension of the HLS pipeline.
- **Layer 3 — API Contracts** (`api-contracts/`) answers *"What must be preserved exactly?"* and serves as a zero-deviation checklist for any reimplementation that must produce byte-compatible HLS output or accept byte-compatible HLS input.

The set contains fourteen markdown files in total: this index plus thirteen leaf documents.

| Path | Layer | Purpose |
|------|-------|---------|
| `README.md` (this file) | Index | Entry point, commit anchor, reading order, citation format, glossary |
| [`functionality/functional-inventory.md`](functionality/functional-inventory.md) | 1 | One section per HLS component (segment generation, playlist construction, encryption, variant streams, captions, live vs VOD, discontinuity, demuxer probe and parser, sample encryption) with plain-language summary followed by technical detail |
| [`functionality/inputs-outputs.md`](functionality/inputs-outputs.md) | 1 | Tables of every input and output: muxer AVPacket and AVFormatContext fields, all AVOption entries with defaults, every emitted EXT-X-* tag and segment artifact, demuxer M3U8 line types parsed and AVPacket fields populated |
| [`functionality/consumer-dependencies.md`](functionality/consumer-dependencies.md) | 1 | Downstream consumers of HLS outputs: media players, CDN ingest, test suites, and in-tree libavformat consumers (notably the DASH muxer's reuse of `hlsplaylist.o` per `[libavformat/Makefile:L189]`) |
| [`functionality/exception-handling.md`](functionality/exception-handling.md) | 1 | Failure scenarios mapped to `AVERROR(*)` return codes from the 129 return sites in `libavformat/hlsenc.c` and the 68 in `libavformat/hls.c`, plus recovery paths and the `ignore_io_errors` interaction |
| [`technical/process-flows.md`](technical/process-flows.md) | 2 | Mermaid `flowchart` diagrams of each major process (segment generation, playlist update, live sliding window, encryption key rotation) plus the AVFormatContext / AVIOContext / segment-file-writer interaction view |
| [`technical/codec-logic.md`](technical/codec-logic.md) | 2 | Exhaustive decision tables for every codec and format branch: HLS version negotiation, segment-cut rule, TS vs fMP4 selection, EXT-X-TARGETDURATION computation, discontinuity placement, program-date-time injection, byterange mode, start-sequence source, second-level filename templating, HTTP method, periodic rekey, independent-segments emission, I-frames-only mode, append-list mode, demuxer key-type resolution |
| [`technical/data-model.md`](technical/data-model.md) | 2 | Full field-level dictionary of every in-scope struct (`HLSContext`, `VariantStream`, `HLSSegment`, `ClosedCaptionsStream`, demuxer `playlist`, `segment`, `variant`, `rendition`, `HLSCryptoContext`, `HLSAudioSetupInfo`), every enum (`HLSFlags`, `SegmentType`, `StartSequenceSourceType`, `PlaylistType`, `KeyType`), and every referenced field from `AVFormatContext`, `AVStream`, `AVOutputFormat`, `AVOption`, `AVDictionary` |
| [`technical/pipeline-orchestration.md`](technical/pipeline-orchestration.md) | 2 | Lifecycle and callback chain: `hls_init` → `hls_write_header` → `hls_write_packet` loop → `hls_write_trailer` → `hls_deinit`, plus the meta-muxer relationship via `VariantStream::avf` and the demuxer lifecycle. Includes a Mermaid lifecycle DAG and ownership class diagram |
| [`technical/integration-interfaces.md`](technical/integration-interfaces.md) | 2 | One page per external interface: AVIOContext file and HTTP writes, protocol handlers, AES-128 crypto pipeline, sample-encryption pipeline, filename templating, MPEG-TS and fMP4 sub-muxer integration, HTTP DELETE for segment cleanup, fMP4 initialization-segment resend |
| [`api-contracts/functional-invariants.md`](api-contracts/functional-invariants.md) | 3 | Zero-deviation behavior checklist: M3U8 header order, EXT-X-VERSION negotiation, segment naming, discontinuity placement, PTS/DTS passthrough, extradata injection, EXT-X-TARGETDURATION semantics, conditional emission rules for every EXT-X-* tag |
| [`api-contracts/data-contracts.md`](api-contracts/data-contracts.md) | 3 | Full data-contract reference in table form: every AVOption with type, default, bounds, and unit; timestamp unit conventions; codec extradata format; byterange offset and size encoding; encryption IV derivation; `HLSCryptoContext` and `HLSAudioSetupInfo` binary layouts; `STREAM_TYPE_HLS_SE_*` values; `FFOutputFormat` field assignments |
| [`api-contracts/timing-dependencies.md`](api-contracts/timing-dependencies.md) | 3 | Processing-order contracts (keyframe detection before segment cut, segment-file flush before playlist update, target-duration computation before first publish, temp-file atomic rename, fMP4 init before first media segment, AES-128 key install before first encrypted segment, ID3 timestamp parse before first packet emission), with a Mermaid sequence diagram |
| [`api-contracts/integration-contracts.md`](api-contracts/integration-contracts.md) | 3 | One page per external-system contract: AES-128 key URI fetch, fMP4 initialization-segment delivery, HTTP chunked transfer with PUT default and `http_persistent` connection reuse, variant-stream BANDWIDTH annotation, sample-encryption transport stream-type values, EXT-X-KEY METHOD line layout, HTTP DELETE for old segment cleanup, EXT-X-MEDIA rendition format, EXT-X-PROGRAM-DATE-TIME ISO-8601 format |

---

## Reading Order — Library Integrator Path

This sequence targets developers integrating FFmpeg HLS as a library — readers who need to understand what the system produces, how to drive it via options, and what guarantees the outputs make, but who do not need to refactor or port the internals. Each document opens with a plain-language executive summary that is sufficient on its own; deeper sections may be skipped on a first read.

1. [`functionality/functional-inventory.md`](functionality/functional-inventory.md) — what components exist and what each one does
2. [`functionality/inputs-outputs.md`](functionality/inputs-outputs.md) — what goes in (packets, options) and what comes out (playlists, segment files, HTTP requests)
3. [`functionality/consumer-dependencies.md`](functionality/consumer-dependencies.md) — who consumes the outputs and what they expect
4. [`functionality/exception-handling.md`](functionality/exception-handling.md) — what fails, how it surfaces as an `AVERROR(*)` code, and how to recover
5. [`api-contracts/functional-invariants.md`](api-contracts/functional-invariants.md) — what behaviors are guaranteed across versions
6. [`api-contracts/data-contracts.md`](api-contracts/data-contracts.md) — what data shapes are guaranteed at the API boundary
7. [`api-contracts/integration-contracts.md`](api-contracts/integration-contracts.md) — what external-system shapes are guaranteed (HTTP, AES, M3U8 wire format)

---

## Reading Order — Engineer Port-Scoping Path

This sequence targets engineers scoping a refactor, extension, or port (for example, to a different language or framework). It traverses all thirteen leaf documents in dependency order: Layer 1 establishes the functional surface, Layer 2 develops the internal mechanism, and Layer 3 pins down the contracts that any reimplementation must preserve.

1. [`functionality/functional-inventory.md`](functionality/functional-inventory.md)
2. [`functionality/inputs-outputs.md`](functionality/inputs-outputs.md)
3. [`functionality/consumer-dependencies.md`](functionality/consumer-dependencies.md)
4. [`functionality/exception-handling.md`](functionality/exception-handling.md)
5. [`technical/process-flows.md`](technical/process-flows.md)
6. [`technical/codec-logic.md`](technical/codec-logic.md)
7. [`technical/data-model.md`](technical/data-model.md)
8. [`technical/pipeline-orchestration.md`](technical/pipeline-orchestration.md)
9. [`technical/integration-interfaces.md`](technical/integration-interfaces.md)
10. [`api-contracts/functional-invariants.md`](api-contracts/functional-invariants.md)
11. [`api-contracts/data-contracts.md`](api-contracts/data-contracts.md)
12. [`api-contracts/timing-dependencies.md`](api-contracts/timing-dependencies.md)
13. [`api-contracts/integration-contracts.md`](api-contracts/integration-contracts.md)

---

## Citation Format

Every claim in the documentation set that derives from source code carries an inline citation immediately after the claim. The set defines four citation forms, used uniformly across all thirteen leaf documents.

- **Inline file-and-line range citation:** `` `[<repo-relative-path>:L<start>-L<end>]` `` for claims supported by a contiguous block of source. Example: `` `[libavformat/hlsenc.c:L2410-L2691]` `` references the body of `hls_write_packet`.
- **Single-line citation:** `` `[<path>:L<n>]` `` for claims supported by exactly one line. Example: `` `[libavformat/hls.c:L26]` `` references the RFC 8216 URL.
- **Texinfo section citation:** `` `[<path>:§<heading>]` `` for cross-references into the existing FFmpeg Texinfo documentation. Example: `` `[doc/muxers.texi:§hls]` `` references the user-facing HLS muxer option reference.
- **Inferred-claim tag:** `` `[inferred — no direct source]` `` when a behavior is reasonably deducible from the surrounding code but not directly stated by any single source range. Inferred claims are flagged so downstream maintainers know to re-validate them when surrounding source changes.

All citations resolve against commit `566ad786` (see the banner above). Repository-relative paths begin at the repository root — for example `libavformat/hlsenc.c`, never `./libavformat/hlsenc.c` and never an absolute path.

---

## Cross-Document Dependencies

The thirteen leaf documents are not strictly independent — several pairs share line locators or build on shared definitions. The dependencies below explain the cross-references a reader will encounter while moving between documents.

- **`technical/data-model.md`** is the canonical struct dictionary for the set. Every Layer 2 and Layer 3 document references its field definitions rather than re-stating them. Readers consulting an unfamiliar struct field encountered in another document are expected to land on `data-model.md` for the authoritative definition.
- **`technical/process-flows.md`** and **`technical/pipeline-orchestration.md`** describe overlapping subject matter from complementary angles: the former is diagram-driven (Mermaid flowcharts of each process), and the latter is narrative-driven (lifecycle phases and callback chains). Both are cited by `api-contracts/timing-dependencies.md`, which lifts the ordering assertions and re-expresses them as zero-deviation timing contracts.
- **`api-contracts/functional-invariants.md`** and **`api-contracts/data-contracts.md`** frequently cite the same line locators (for example, the EXT-X-VERSION negotiation range in `libavformat/hlsenc.c`) from different perspectives — one as a behavioral invariant, the other as a data-shape contract. **`api-contracts/integration-contracts.md`** cites the same locators a third time, framed as external-system contracts.
- **`functionality/exception-handling.md`** sources its `AVERROR(*)` catalog from the return-statement sites in `libavformat/hlsenc.c` and `libavformat/hls.c`. The resulting failure-scenario descriptions are referenced (without restating) inside `api-contracts/integration-contracts.md` where HTTP, key-fetch, or sub-muxer failures appear in the external-contract narrative.

---

## Glossary of HLS-Specific Terms

The terms below are used throughout the documentation set with the meanings given here. Each term is defined once; subsequent occurrences are not re-introduced.

- **HLS** — HTTP Live Streaming, the Apple-defined adaptive bitrate streaming protocol specified in RFC 8216 (URL referenced inline in `[libavformat/hls.c:L26]`).
- **Muxer** — In FFmpeg parlance, the component that combines audio, video, and subtitle streams into a container format for output. The HLS muxer source is `libavformat/hlsenc.c`.
- **Demuxer** — The component that parses a container format and extracts elementary streams. The HLS demuxer source is `libavformat/hls.c`.
- **M3U8 (or .m3u8)** — The playlist file format used by HLS, a UTF-8 encoded extension of the M3U playlist format. M3U8 files are produced by the HLS muxer and consumed by the HLS demuxer.
- **Segment** — A single media file (TS or fMP4) referenced by a line entry in an M3U8 playlist. Segments are produced one at a time by the muxer and fetched one at a time by HLS clients.
- **Variant** — An entry under `EXT-X-STREAM-INF` in a master playlist, representing one quality or bitrate level in an adaptive bitrate ladder. Each variant is itself a complete media playlist.
- **Rendition** — An entry under `EXT-X-MEDIA` in a master playlist, representing an alternate audio, video, or subtitle track that can be selected alongside a variant.
- **Target Duration** — The maximum segment duration declared in a media playlist via the `EXT-X-TARGETDURATION` tag. Clients use it to size their request scheduling and rebuffer thresholds.
- **Discontinuity** — A break in segment continuity (in timing, encoding parameters, or program identity) marked by the `EXT-X-DISCONTINUITY` tag immediately preceding the affected segment entry.
- **EXT-X-*** — The family of M3U8 extension tags defined by the HLS specification. The set used by this pipeline includes `EXT-X-VERSION`, `EXT-X-TARGETDURATION`, `EXT-X-MEDIA-SEQUENCE`, `EXT-X-KEY`, `EXT-X-MAP`, `EXT-X-BYTERANGE`, `EXT-X-DISCONTINUITY`, `EXT-X-ENDLIST`, `EXT-X-MEDIA`, `EXT-X-STREAM-INF`, `EXT-X-PROGRAM-DATE-TIME`, `EXT-X-ALLOW-CACHE`, `EXT-X-INDEPENDENT-SEGMENTS`, `EXT-X-I-FRAMES-ONLY`, and `EXT-X-PLAYLIST-TYPE`.
- **fMP4** — Fragmented MP4 / ISO BMFF segment format. Selected at the muxer via `hls_segment_type=fmp4`. fMP4 mode also forces an initialization segment (default filename `init.mp4`) referenced by `EXT-X-MAP`.
- **TS / MPEG-TS** — MPEG-2 Transport Stream segment format. Selected at the muxer via `hls_segment_type=mpegts`, which is the default. The TS sub-muxer (`libavformat/mpegtsenc.c`) is invoked per segment.
- **AVERROR()** — FFmpeg's negative-errno return convention from `libavutil/error.h`. Codes emitted from this pipeline include `AVERROR(ENOMEM)`, `AVERROR(EINVAL)`, `AVERROR(EIO)`, `AVERROR_EOF`, `AVERROR_MUXER_NOT_FOUND`, and `AVERROR_PATCHWELCOME`. A complete catalog with per-scenario recovery is given in `functionality/exception-handling.md`.
- **AVFormatContext / AVStream / AVPacket / AVOutputFormat / AVInputFormat** — Core FFmpeg libavformat types defined in `libavformat/avformat.h`. The HLS muxer is registered as an `FFOutputFormat ff_hls_muxer` and the HLS demuxer as an `FFInputFormat ff_hls_demuxer`.
- **AVOption** — FFmpeg's option-table infrastructure defined in `libavutil/opt.h`. The HLS muxer exposes 47 user-tunable options through this mechanism; their full reference is in `api-contracts/data-contracts.md` and `functionality/inputs-outputs.md`.
- **AVDictionary** — Key-value string dictionary defined in `libavutil/dict.h`. The HLS muxer accepts an `AV_OPT_TYPE_DICT` option (`hls_segment_options`) that propagates per-segment options to the sub-muxer.
- **Sample Encryption** — Apple's encryption scheme for HLS where individual samples (frames) inside the transport-stream payload are encrypted in place, distinct from the AES-128 full-segment encryption mode. The implementation lives in `libavformat/hls_sample_encryption.c` and `libavformat/hls_sample_encryption.h`; the four affected MPEG-TS stream types are defined as `STREAM_TYPE_HLS_SE_*` at `[libavformat/mpegts.h:L177-L180]`.
- **Inferred Claim** — A claim that cannot be directly cited to a specific source line range but can be reasonably deduced from surrounding code. Flagged inline with `[inferred — no direct source]` so downstream maintainers can re-validate when the surrounding source changes.

---

## Scope

This documentation set targets the HLS muxer and demuxer pipeline only, anchored to commit `566ad786`. No source-code modifications are introduced anywhere in the repository as part of this work — the deliverable is pure markdown text under `docs/hls-pipeline/`. Existing FFmpeg documentation infrastructure (the Texinfo and Doxygen pipelines under `doc/`) is unchanged and continues to publish the user-facing manuals; the new tree augments that infrastructure without replacing or overlapping it.

The following subjects are explicitly out of scope and are not covered: codec internals (encoders and decoders under `libavcodec/`), filter graph and video filters (`libavfilter/`), image scaling and color conversion (`libswscale/`, `libavutil`), audio resampling (`libswresample/`), hardware acceleration backends (Vulkan, CUDA, OpenCL filter implementations), platform-specific SIMD optimizations (`libavcodec/x86/`, `libavcodec/aarch64/`, `libavcodec/arm/`), container formats in `libavformat/` other than those directly invoked by the HLS pipeline, the test harness and FATE reference data under `tests/`, the compatibility shims under `compat/`, and the existing `doc/` Texinfo and Doxygen tooling. The HLS pipeline transitively depends on several of these subsystems at runtime, but the documentation treats them as opaque external dependencies rather than re-documenting them.

---

## In-Scope Source Files

The fourteen source files below are the sole source of truth for every documented claim. Line counts are verified at commit `566ad786`.

| Path | Lines | Role |
|------|-------|------|
| `libavformat/hlsenc.c` | 3,207 | Primary HLS muxer — `FFOutputFormat ff_hls_muxer` registration at `[libavformat/hlsenc.c:L3191]` with callbacks `hls_init`, `hls_write_header`, `hls_write_packet`, `hls_write_trailer`, `hls_deinit` |
| `libavformat/hls.c` | 2,912 | Primary HLS demuxer — `FFInputFormat ff_hls_demuxer` registration with callbacks `hls_probe`, `hls_read_header`, `hls_read_packet`, `hls_close`, `hls_read_seek` |
| `libavformat/hlsplaylist.c` | 206 | Playlist tag-writer source, shared with the DASH muxer per `[libavformat/Makefile:L189]` |
| `libavformat/hlsplaylist.h` | 65 | `PlaylistType` enum and `ff_hls_write_*` prototypes |
| `libavformat/hls_sample_encryption.c` | 396 | Sample-encryption transform implementation |
| `libavformat/hls_sample_encryption.h` | 65 | `HLSCryptoContext`, `HLSAudioSetupInfo`, and constants |
| `libavformat/segment.c` | 1,136 | Alternative generic segmenter, referenced for breakage analysis only |
| `libavformat/mpegtsenc.c` | 2,424 | MPEG-TS sub-muxer invoked when `hls_segment_type=mpegts` |
| `libavformat/mpegts.c` | 3,735 | MPEG-TS sub-demuxer |
| `libavformat/mpegts.h` | 306 | `STREAM_TYPE_HLS_SE_*` constants at `[libavformat/mpegts.h:L177-L180]` |
| `libavformat/avformat.h` | 3,163 | Public API contract surface — `AVFormatContext`, `AVStream`, `AVOutputFormat`, `AVInputFormat`, write entry points |
| `libavutil/opt.h` | 1,194 | AVOption infrastructure used by the HLS option tables |
| `libavutil/aes.h` | 69 | AES public interface — `av_aes_alloc` at `[libavutil/aes.h:L41]`, `av_aes_init` at `[libavutil/aes.h:L51]`, `av_aes_crypt` at `[libavutil/aes.h:L63]` |
| `libavutil/dict.h` | 242 | AVDictionary public interface |

Auxiliary references — read but never modified, cited from individual leaf documents — include `libavformat/Makefile` (build-time object-set declarations), `libavformat/allformats.c` (format registration), and `doc/muxers.texi` (existing Texinfo HLS option reference at `[doc/muxers.texi:§hls]`, line 1887 onward).
