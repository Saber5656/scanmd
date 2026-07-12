# Title

ClipboardSource: pasteboard image and file URLs

## Summary

Implement `ClipboardSource: ScanSource` reading `NSPasteboard.general` once per explicit
invocation, resolving image data first, then image/PDF file URLs, with the privacy rules
of DESIGN §7.2.

## Context

Trust boundary B5: reads happen only on explicit user command; plain-text clipboards are
rejected by design (OCR of text would be nonsense; passthrough is v2).

## Scope

- `Sources/ScanMDKit/Sources/ClipboardSource.swift` (+ `PasteboardReading` protocol fake
  for tests).
- Unit tests via fake pasteboard; one guarded real-pasteboard integration test.

## Detailed Requirements

1. Abstraction: `protocol PasteboardReading { func data(forType:) -> Data?;
   func readObjects(forClasses:options:) -> [Any]?; var types: [NSPasteboard.PasteboardType]? { get } }`
   — production wraps `NSPasteboard.general`; tests inject fakes. The pasteboard is read
   **once** inside `acquire()`; no observation, no polling, no change-count loops.
2. Resolution order (first match wins):
   a. Image data: try types in order `.png`, `.tiff` → decode via Issue 12's
      `ImageDecoder` (same limits & orientation handling) → single image payload,
      kind `.clipboard`, `originPath` nil.
   b. File URLs: `readObjects(forClasses: [NSURL.self])` with
      `NSPasteboardURLReadingFileURLsOnlyKey: true`; take the **first** URL; determine
      UTType via `URLResourceValues.contentType`:
      image → delegate to `ImageFileSource(path:)`; PDF → delegate to `PDFSource(path:)`
      (Issue 14; to keep this issue mergeable independently, the PDF delegation is added
      behind a factory closure injected at pipeline assembly — default factory lands in
      Issue 18). Metadata keeps kind `.clipboard` but `originPath` = the file's path.
      More than one URL → proceed with first, count reported via a stderr-visible
      warning string in the receipt path (CLI prints it; no logging of names).
   c. Anything else (plain text, empty) →
      `.noInput("clipboard has no image, file, or PDF")` (DESIGN §7.2 verbatim reason).
3. Never write to the pasteboard here (that is `ClipboardSink`).
4. Delegated file reads enforce all Issue 12/14 limits unchanged (no bypass).
5. Tests (fake pasteboard): png data path · tiff data path · image-file URL path ·
   pdf URL path (factory stub) · multiple URLs → first + warning · text-only → `.noInput`
   with exact message · empty → `.noInput` · oversized image data → `.limitExceeded` ·
   file URL to missing file → `.noInput`. Real-pasteboard integration test: writes a
   fixture PNG to a **freshly created named pasteboard** (never `.general` in tests),
   reads it back through the source; marked to skip when running headless CI if flaky.

## Acceptance Criteria

- [ ] Resolution order and all fake-pasteboard tests green.
- [ ] Grep: no `NSPasteboard.general` reference outside the production wrapper +
      `ClipboardSink`.
- [ ] Rejection message matches DESIGN §7.2 exactly.
- [ ] No pasteboard write APIs referenced in this file.

## Validation

`swift test --filter ClipboardSourceTests`. Manual: copy a screenshot (⌃⇧⌘4), run the
wave-3 CLI later; for this issue a small test harness target invocation suffices —
paste transcript in PR.

## Dependencies

Issues 05, 12. (PDF delegation completed when 14/18 land; factory keeps this issue
independent.)

## Non-goals

Plain-text passthrough (v2, DESIGN §13), clipboard *writing* (Issue 11), CLI subcommand
(Issue 18).

## Design References

DESIGN §7.2, §10.1-B5.
