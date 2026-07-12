# macOS capture & OCR API landscape (v1 feasibility notes)

- Date: 2026-07-12
- Purpose: record the API facts and risks the v1 design depends on, with verification
  status. Implementation issues cite this file instead of re-deriving platform folklore.
- Legend: ✅ = well-established, safe to build on. ⚠️ = believed true, verify during the
  named issue (also tracked as known unknowns in ISSUE_PLAN §8).

## OCR — Vision

| Fact | Status |
|---|---|
| `VNRecognizeTextRequest` supports `ja-JP` with `.accurate` + `usesLanguageCorrection`; quality is the practical best on-device option for Japanese | ✅ |
| `automaticallyDetectsLanguage` is available from macOS 13 | ✅ |
| Results are line-level `VNRecognizedTextObservation` with normalized **bottom-left-origin** bounding boxes; per-candidate `confidence` | ✅ |
| OCR output text is not bit-identical across macOS releases (model updates) | ✅ — tests assert token recall ≥ 0.9, never golden OCR strings (DESIGN §11, KU-3) |
| `supportedRecognitionLanguages()` can enumerate languages at runtime for validation | ✅ (Issue 08 uses it to reject bad `--lang` values) |
| macOS 26-era structured document request (`RecognizeDocumentsRequest`) could give tables/paragraph structure natively | ⚠️ v2 candidate only; not required by any v1 issue |

## Screen capture

| Fact | Status |
|---|---|
| No public API exposes the system's interactive region-selection UI; `screencapture -i` is the standard workaround | ✅ (basis of ADR-004) |
| `screencapture` requires Screen Recording TCC; attribution goes to the **responsible process** (Terminal/iTerm for CLI use, the app itself for `ScanMD.app`) | ✅ concept / ⚠️ exact prompt behavior matrix — verify in Issue 15 manual checklist |
| Cancel signaling of `screencapture -i` (Esc): historically exit status 1 and/or no output file depending on macOS version | ⚠️ KU-5 — wrapper must treat nonzero exit **or** missing/empty file as cancel; verify on macOS 14 & 15 in Issue 15 |
| `CGPreflightScreenCaptureAccess()` / `CGRequestScreenCaptureAccess()` preflight works for both surfaces | ✅ |
| ScreenCaptureKit `SCScreenshotManager` (macOS 14+) is the v2 path for a custom overlay | ✅ (deferred) |

## Camera

| Fact | Status |
|---|---|
| Continuity Camera iPhones appear as ordinary `AVCaptureDevice`s (deviceType `.continuityCamera`, macOS 14+) — usable from CLI and app alike via `AVCapturePhotoOutput` | ✅ API / ⚠️ device discovery timing can be slow on first use — Issue 16 adds a discovery timeout |
| The document-scan sheet (`importFromDevice:` responder chain, "Scan Documents") is AppKit-menu integration, awkward outside a regular app responder chain | ✅ — deferred to v2 (DESIGN §13) |
| Camera TCC requires `NSCameraUsageDescription`; a bare executable can embed an Info.plist via linker flag `-sectcreate __TEXT __info_plist <plist>` | ✅ technique / ⚠️ KU-2: whether the camera prompt for a terminal-spawned CLI attributes to the terminal (which lacks the usage string) or the binary — must be validated early in Issue 16; fallback documented there (route camera capture through the app if CLI attribution proves unreliable) |
| `AVCaptureDevice.authorizationStatus(for: .video)` / `requestAccess` preflight | ✅ |

## PDF

| Fact | Status |
|---|---|
| PDFKit `PDFPage.string` returns the text layer where present; scanned PDFs return empty/whitespace | ✅ (threshold `pdf.textLayerMinChars` handles junk layers) |
| Rendering a page to CGImage at chosen DPI via `PDFPage.thumbnail(of:for:)` or `draw(with:to:)` into a bitmap context | ✅ (Issue 14 specifies the bitmap-context path for exact DPI control) |
| PDFKit tolerates malformed input reasonably but must be treated as an untrusted-input boundary (limits + graceful error mapping) | ✅ (DESIGN §10.1 B1) |

## Packaging & app plumbing

| Fact | Status |
|---|---|
| `MenuBarExtra` (SwiftUI) is stable for menu-bar-only apps with `LSUIElement` | ✅ |
| `SMAppService.mainApp` handles launch-at-login on macOS 13+ | ✅ |
| `UNUserNotificationCenter` works for signed menu bar apps; unsigned dev builds can be flaky about notification registration | ⚠️ noted in Issue 23 manual checklist |
| XcodeGen generates a valid app project from `project.yml`; CI: `brew install xcodegen` | ✅ (ADR-005) |
| `x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture` / `…?Privacy_Camera` deep links open the right Settings panes | ⚠️ believed stable through macOS 15; verify in Issue 23 |
| Notarization via `notarytool` with maintainer-local credentials; CI stays keyless in v1 | ✅ (ADR-005, DESIGN §12) |

## Risks worth designing around (summary)

1. **KU-2 CLI camera TCC attribution** — validate first thing in Issue 16; fallback path
   documented (app-only camera + CLI clear error) without changing any other module.
2. **KU-3 OCR drift across macOS versions** — recall-based tests only.
3. **KU-5 `screencapture` cancel matrix** — tolerant cancel detection, manual matrix in
   Issue 15.
4. First-run TCC prompts cannot be exercised in CI — every TCC-touching issue carries a
   manual validation checklist (DESIGN §11).
