# Title

VisionTextRecognizer (Vision OCR wrapper)

## Summary

Implement `VisionTextRecognizer: TextRecognizer` wrapping `VNRecognizeTextRequest` with
the DESIGN §6 option mapping, language validation, coordinate conversion, and
recall-based fixture tests.

## Context

This is the only module touching Vision. Everything downstream consumes
`[RecognizedLine]` in top-left normalized coordinates (DESIGN §5.1).

## Scope

- `Sources/ScanMDKit/Recognize/VisionTextRecognizer.swift`.
- Tests wiring the real recognizer into Issue 07's recall harness.

## Detailed Requirements

1. `recognize(_ image: CGImage, options: OCROptions) async throws -> [RecognizedLine]`:
   - `VNImageRequestHandler(cgImage:)`; request configured per DESIGN §6 table:
     `recognitionLevel` (accurate|fast from options), `recognitionLanguages` =
     `options.languages`, `automaticallyDetectsLanguage = options.autoDetect`,
     `usesLanguageCorrection = true`, `minimumTextHeight` untouched.
   - Perform on a background queue continuation; the API is `async` and thread-safe for
     concurrent pages (stateless — build a fresh request per call).
   - Map each observation: `topCandidates(1).first` → text + confidence; skip if below
     `options.minConfidence`; bbox via `NormalizedRect.fromVision` (§5.1 flip).
   - Output order: Vision's observation order preserved (layout sorting is Issue 09's
     job, not here).
2. Language validation helper:
   `static func validate(languages: [String]) throws` — checks each code against
   `VNRecognizeTextRequest.supportedRecognitionLanguages()` for the configured revision;
   unknown code → `ScanMDError.usage("unsupported OCR language: <code>; supported: …")`.
   (Issue 17 calls this for `--lang`.)
3. Errors: Vision failures map via `mapToScanMDError` to `.internalError`; an image with
   zero observations returns `[]` (NOT an error — DESIGN §6).
4. Determinism guardrails in tests: use Issue 07 synthetic fixtures with
   `assertRecall(threshold: 0.9)`; additionally assert for `mixed-headings.png` that the
   title line's bbox height > 1.5× median line height (feeds Issue 09's heuristics) and
   that all bboxes are within `[0,1]` with top-left origin (title y < body y).
5. No retained state, no caches, `Sendable` conformance under strict concurrency.

## Acceptance Criteria

- [ ] Recall ≥ 0.9 on all five synthetic fixtures and both photo fixtures (ja + en).
- [ ] Language validation rejects `xx-XX` with the documented message; accepts `ja-JP`.
- [ ] Bbox orientation test passes (title above body ⇒ smaller y).
- [ ] `minConfidence: 0.99` filters most lines on a photo fixture (filter path tested).
- [ ] Zero-text image (blank white fixture) → `[]`, no throw.

## Validation

`swift test --filter RecognizerTests` on macOS 14+ runner and locally; PR notes the macOS
version used (KU-3 context).

## Dependencies

Issues 05, 07.

## Non-goals

Reading order / paragraph grouping (Issue 09), multi-candidate selection and
`minimumTextHeight` tuning (v2, DESIGN §13).

## Design References

DESIGN §5.1, §6; research/macos-capture-ocr-apis.md (OCR facts, KU-3).
