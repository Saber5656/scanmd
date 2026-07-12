# Title

ImageFileSource: image files and stdin

## Summary

Implement `ImageFileSource: ScanSource` decoding image files (PNG/JPEG/HEIC/TIFF/GIF/
WebP) and raw stdin bytes via ImageIO, with EXIF orientation normalization and limit
enforcement.

## Context

First untrusted-input boundary implementation (B1). Also the delegate target for
ClipboardSource file URLs (Issue 13) and the loader for ScreenRegionSource temp files
(Issue 15) — keep the decode path a reusable internal function.

## Scope

- `Sources/ScanMDKit/Sources/ImageFileSource.swift` (+ internal `ImageDecoder`).
- Tests over the fixture corpus incl. malformed files.

## Detailed Requirements

1. Construction: `ImageFileSource(path: String)` (kind `.imageFile`) or
   `ImageFileSource(stdinData: Data)` (kind `.stdin`).
2. Preflight for paths, in order: file exists & readable (`.noInput` naming basename
   only), regular file (reject directories/FIFOs → `.unsupportedInput`), size ≤
   `limits.maxFileSizeMB` (`.limitExceeded` BEFORE reading contents; use
   `FileManager.attributesOfItem`).
3. Decode via `ImageDecoder.decode(data:) -> CGImage`:
   - `CGImageSourceCreateWithData` with type hinting off; must never call a custom
     parser. Nil source / zero images → `.unsupportedInput("undecodable image")`.
   - **Dimension gate before pixel decode**: read
     `CGImageSourceCopyPropertiesAtIndex(_, 0, kCGImageSourceShouldCache: false)` and
     reject `pixelWidth × pixelHeight > limits.maxImagePixels` → `.limitExceeded`
     (this is what makes `huge-dimensions.png` cheap to reject).
   - Apply EXIF orientation: read `kCGImagePropertyOrientation`, transform to upright
     pixels (draw into a CGContext with the corresponding affine transform; the eight
     orientation cases must be handled — table them in code).
   - GIF/animated: first frame only.
4. Stdin variant: `Data` already in memory; enforce `maxFileSizeMB` against `data.count`
   and the same dimension gate. (Reading stdin happens in the CLI, Issue 17 — this type
   just accepts `Data` so it stays testable.)
5. `acquire()` returns a `SourceAcquisition` with exactly one `PagePayload.image`.
6. `SourceMetadata.originPath` = absolute path (nil for stdin); `pageCount = 1`.
7. Tests: every supported format fixture decodes · orientation: generate a rotated
   TIFF fixture with EXIF orientation 6 and assert the decoded image is upright
   (width/height swapped) · `not-an-image.png` → `.unsupportedInput` ·
   `truncated.png` → `.unsupportedInput` · `huge-dimensions.png` → `.limitExceeded`
   without high memory use (assert wall-clock < 1 s) · oversized file (create sparse
   file > limit in temp) → `.limitExceeded` · directory path → `.unsupportedInput` ·
   missing path → `.noInput` · error messages contain basename, never full path
   (Issue 04 rule).

## Acceptance Criteria

- [ ] All format + malformed + limit tests green; no crash on any corpus file.
- [ ] Dimension gate proven cheap (timing assertion above).
- [ ] `ImageDecoder.decode` is internal-reusable (no file-system coupling) and consumed
      later by Issues 13/15 without modification.
- [ ] Full-path leak test passes.

## Validation

`swift test --filter ImageSourceTests`; PR shows recognizer+layout+renderer smoke over
`ja-paragraphs.png` producing Markdown (first full local pipeline!).

## Dependencies

Issues 05, 06 (Limits), 07 (fixtures).

## Non-goals

Clipboard reading (Issue 13), CLI `image` subcommand and stdin plumbing (Issue 17),
multi-frame/animated handling (v2).

## Design References

DESIGN §7.1, §10.1-B1, §10.4.
