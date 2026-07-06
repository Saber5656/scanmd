# Title

Core data model, stage protocols, and pipeline orchestrator

## Summary

Implement the normative data types (`NormalizedRect`, `RecognizedLine`, `Block`,
`ScanDocument`, …), the five stage protocols, the `MarkdownFormatter` seam with
`PassthroughFormatter`, and the `Pipeline` orchestrator that wires stages together.

## Context

DESIGN §5 fixes names and shapes so all later issues implement against stable interfaces;
DESIGN §4 defines stage order and concurrency. This issue is pure types + orchestration —
every stage is injected, none implemented here.

## Scope

- `Sources/ScanMDKit/Model/*` (types of DESIGN §5.2, §5.3, §5.5).
- `Sources/ScanMDKit/Pipeline/Protocols.swift`, `Pipeline.swift`, `Options.swift`.
- Unit tests with stub stages.

## Detailed Requirements

1. Implement DESIGN §5.2 types verbatim (names, fields, Codable/Sendable conformances):
   `SourceKind`, `SourceMetadata`, `NormalizedRect { x,y,width,height: Double }`,
   `RecognizedLine`, `Block`, `PageResult`, `ScanDocument`, plus
   `ScanStats { lineCount: Int, meanConfidence: Double, stageMillis: [String: Int] }`.
2. `Block` Codable encoding: tagged single-key object form —
   `{"heading":{"level":1,"text":"…"}}`, `{"paragraph":{"text":"…"}}`,
   `{"listItem":{"ordered":false,"indent":0,"text":"…"}}`, `{"pageBreak":{}}` — with a
   round-trip test (this becomes the `--format json` contract, Issue 18).
3. Geometry: `NormalizedRect.fromVision(_ rect: CGRect) -> NormalizedRect` implementing
   the bottom-left → top-left flip of DESIGN §5.1, with tests
   (`(x:0.1,y:0.2,w:0.3,h:0.1)` → `y' = 0.7`).
4. Protocols exactly as DESIGN §5.4 + §5.6: `ScanSource`, `TextRecognizer`,
   `LayoutReconstructing`, `MarkdownRendering`, `OutputSink`, `MarkdownFormatter`,
   `PassthroughFormatter`. Options structs: `OCROptions { languages: [String],
   autoDetect: Bool, level: Level, minConfidence: Double }`,
   `LayoutOptions { headingDetection: Bool, listDetection: Bool }`,
   `RenderOptions { escaping: EscapeMode, frontmatter: Bool, pageSeparator: String,
   toolVersion: String }` — all `Sendable`, all with defaults mirroring DESIGN §8.7.
5. `Pipeline` (struct, injected stages):
   `run(source: any ScanSource, …) async throws -> (ScanDocument, markdown: String)`:
   - `acquire()` → for `.image` payloads run recognizer then layout; for `.text` payloads
     produce paragraph blocks by splitting on blank lines (DESIGN §7.4) with empty `lines`.
   - Concurrency: recognize pages via `withThrowingTaskGroup`, at most
     `min(4, ProcessInfo.processInfo.activeProcessorCount)` in flight; results reassembled
     in page order (test with a stub recognizer that returns out of order).
   - Per-page OCR timeout `ocrTimeoutSecPerPage` → `ScanMDError.timeout("ocr page N")`.
   - Insert `Block.pageBreak` between pages (not before first / after last).
   - Compute `ScanStats` (stage wall-clock millis via `ContinuousClock`).
   - Apply formatter (default `PassthroughFormatter`) to rendered markdown.
   - Empty-document rule: if all pages yield zero non-whitespace text →
     throw `ScanMDError.emptyResult` (DESIGN §8.5).
   - Sinks are NOT called by `Pipeline`; it returns the string (callers own delivery) —
     keeps CLI `--no-stdout` combinations simple.
6. No imports beyond Foundation/CoreGraphics in this issue's files.

## Acceptance Criteria

- [ ] All names/fields compile exactly as in DESIGN §5 (reviewer diff-checks the section).
- [ ] Block JSON round-trip test passes with the tagged encoding above.
- [ ] Pipeline stub-test proves: page order preserved, bounded concurrency, pageBreak
      placement, emptyResult thrown, stats populated.
- [ ] Strict concurrency: no `@unchecked Sendable` anywhere in this issue.

## Validation

`swift test --filter ModelTests --filter PipelineTests`; include the JSON fixture of an
encoded `Block` array in tests.

## Dependencies

Issues 01, 04.

## Non-goals

Real recognizer/layout/renderer/sinks/sources (Issues 08–16); config file I/O (Issue 06).

## Design References

DESIGN §4, §5, §7.4 (text-layer paragraphs), §8.5.
