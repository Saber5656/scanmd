# Title

PDFSource: text layer extraction with OCR fallback

## Summary

Implement `PDFSource: ScanSource` using PDFKit: per-page text-layer extraction when
present, rasterization to CGImage for OCR fallback when not, page-range selection, and
resource limits.

## Context

Trust boundary B1 for the PDF parser; the only multi-page source in v1. Per DESIGN §4.1
and §7.4, text-layer pages skip OCR entirely — the pipeline (Issue 05) already handles
`PagePayload.text`.

## Scope

- `Sources/ScanMDKit/Sources/PDFSource.swift`, `PageRange.swift`.
- Fixture tests (text-layer / scanned / mixed / encrypted / malformed).

## Detailed Requirements

1. `PDFSource(path: String, pages: PageRangeSpec?, forceOCR: Bool, config:…)`.
2. Preflight: exists/readable/regular file (as Issue 12), size ≤ `limits.maxFileSizeMB`.
   `PDFDocument(url:)` nil → `.unsupportedInput("not a valid PDF")`.
   `document.isEncrypted && !document.unlock(withPassword: "")` →
   `.unsupportedInput("encrypted PDF")` (DESIGN §7.4; password option is v2).
3. `PageRange.parse("3" | "2-5" | "7-" | "1,3-4,9")` → sorted unique 1-based page
   numbers; syntax error or any page > pageCount or `0` → `.usage` with the offending
   token. Absent spec = all pages. After selection: count ≤ `pdf.maxPages` else
   `.limitExceeded("pages: N > max M")`.
4. Per selected page (in ascending order):
   - `page.string`, trimmed; if `!forceOCR && trimmed.count >= pdf.textLayerMinChars`
     → `PagePayload.text(pageString)` (untrimmed original except normalized to NFC and
     CRLF→LF).
   - else prepare lazy rasterization → `PagePayload.lazyImage`:
     target pixel size = page media box size (points) × `pdf.rasterDPI / 72`, capped so
     `width×height ≤ limits.maxImagePixels` (reduce DPI proportionally if needed, floor
     150 DPI; below floor → `.limitExceeded`). Draw via CGContext bitmap
     (`page.draw(with: .mediaBox, to: context)`) with white background fill first,
     y-flip handled, `interpolationQuality = .high`.
5. Memory discipline: scanned pages are represented as `PagePayload.lazyImage` closures
   in the returned `SourceAcquisition`, not pre-rendered `CGImage` values. The pipeline
   invokes one lazy image at a time and releases it after recognition, so peak memory is
   one page bitmap (+ document and text payload descriptors). Assert peak behavior with
   a 50-page scanned fixture generated on the fly in temp (not committed).
6. Metadata: `pageCount` = selected count; `originPath` absolute.
7. Tests: `text-layer.pdf` → 2 text payloads, exact strings · `scanned.pdf` → 2 image
   payloads with DPI-derived pixel sizes · `mixed.pdf` → [text, image] ·
   `--force-ocr` on `text-layer.pdf` → images · every `PageRange` grammar case incl.
   errors · `encrypted.pdf` → `.unsupportedInput` · `zero-byte.pdf`/`deep-bomb.pdf` →
   typed error, no crash · maxPages enforcement · DPI cap reduction math unit-tested.

## Acceptance Criteria

- [ ] All fixture/grammar/limit tests green; no crash on malformed corpus.
- [ ] Text-layer path performs zero Vision calls (assert recognizer stub not invoked).
- [ ] Rasterization math (points→pixels, cap, floor) unit-tested to the pixel.
- [ ] Peak-memory test demonstrates sequential single-page rasterization.

## Validation

`swift test --filter PDFSourceTests`; PR shows `mixed.pdf` flowing through the full
local pipeline (text page verbatim paragraphs + OCR'd page) with `---` page separator.

## Dependencies

Issues 05, 06, 07.

## Non-goals

PDF password option, PDF outline→heading mapping (v2); CLI `pdf` subcommand (Issue 18);
OCR itself (pipeline concern).

## Design References

DESIGN §7.4, §4.1, §10.1-B1, §10.4; research/macos-capture-ocr-apis.md (PDF facts).
