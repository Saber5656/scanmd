# scanmd — v1 Design

Status: Draft for v1 implementation
Last updated: 2026-07-07
Owner: Saber5656
Source of truth: this file and the other documents under `docs/` in this repository.

---

## 1. Product overview

scanmd is a macOS tool that turns text captured from anywhere — a screen region, the camera,
image files, the clipboard, or PDFs — into clean Markdown.

It ships as two user-facing surfaces built on one shared core:

| Surface | Binary | Audience / use |
|---|---|---|
| CLI | `scanmd` | Terminal users, scripts, agents, launcher integrations (Raycast, Shortcuts) |
| Menu bar app | `ScanMD.app` | Always-on capture with a global hotkey, no terminal needed |

Both surfaces call the same Swift library, `ScanMDKit`, which implements the whole
capture → recognize → structure → render → deliver pipeline.

### 1.1 Design principles

1. **Local-only by default.** v1 performs zero network I/O. All OCR runs on-device via Apple
   Vision. This is a security guarantee, not an optimization (see §10).
2. **Pipe-friendly.** The CLI behaves like a good Unix citizen: Markdown to stdout, errors to
   stderr, meaningful exit codes, `-` for stdin.
3. **Markdown out, always.** The output is honest Markdown: paragraphs, headings, lists,
   optional YAML front matter. No proprietary formats.
4. **Small, auditable core.** Minimal third-party dependencies; Apple frameworks preferred.
5. **Weaker-agent executable.** Every module is specified precisely enough (schemas, thresholds,
   exact flags) that a low-capability implementation agent can build it without guessing.

### 1.2 Primary use cases

| # | Scenario | Surface | Flow |
|---|---|---|---|
| U1 | Grab a paragraph from a webinar slide into notes | App hotkey | ⌃⌥⌘M → drag region → Markdown lands on clipboard + notification |
| U2 | Convert a scanned handout PDF to Markdown | CLI | `scanmd pdf handout.pdf --out notes/` |
| U3 | OCR a photo of a whiteboard | CLI | `scanmd image IMG_0123.heic --copy` |
| U4 | Digitize a paper document with iPhone/webcam | App or CLI | Menu → From Camera… / `scanmd camera` |
| U5 | Agent pipeline ingestion | CLI | `scanmd image page.png --format json \| jq …` |
| U6 | Clipboard screenshot → Markdown | Either | `scanmd clipboard` or menu item |

### 1.3 Non-goals for v1 (deferred, see §13)

- LLM-assisted formatting (design extension point only — `Formatter` protocol, §5.6)
- Table structure reconstruction (emit table text as plain paragraphs in v1)
- URL / HTML ingestion, audio transcription
- Windows / Linux support
- Custom in-app region-selection overlay (v1 shells out to `screencapture -i`, ADR-004)
- Mac App Store distribution and App Sandbox (v1 is Developer ID + hardened runtime, ADR-005)
- Watch-folder / daemon mode, capture history UI, localization of app UI strings

---

## 2. Supported platforms and toolchain

| Item | Value | Rationale |
|---|---|---|
| Minimum macOS | 14.0 (Sonoma) | `MenuBarExtra`, Vision auto language detection, Continuity Camera as AVCapture device all stable ≥14 |
| Architectures | arm64 + x86_64 (universal) | Homebrew default expectation |
| Language | Swift (5.10+, concurrency enabled) | ADR-002 |
| Package manager | SwiftPM for `ScanMDKit` + `scanmd`; XcodeGen-generated Xcode project for `ScanMD.app` | ADR-005 |
| Third-party deps (v1, exhaustive) | `swift-argument-parser` (Apple), `KeyboardShortcuts` (sindresorhus, app only) | §10.6 |

Exact toolchain versions are pinned when Issue 01 lands (known unknown KU-1, ISSUE_PLAN §8).

---

## 3. Repository layout (target state)

```
scanmd/
├── Package.swift                  # ScanMDKit (library) + scanmd (executable)
├── Package.resolved               # committed, pinned
├── Sources/
│   ├── ScanMDKit/
│   │   ├── Model/                 # RecognizedLine, Block, ScanDocument, SourceMetadata…
│   │   ├── Pipeline/              # Pipeline orchestrator, protocols
│   │   ├── Recognize/             # VisionTextRecognizer
│   │   ├── Layout/                # LayoutReconstructor
│   │   ├── Render/                # MarkdownRenderer, FrontMatterBuilder
│   │   ├── Sources/               # ImageFileSource, ClipboardSource, PDFSource,
│   │   │                          #   ScreenRegionSource, CameraSource
│   │   ├── Sinks/                 # StdoutSink, ClipboardSink, FileSink
│   │   ├── Config/                # ConfigLoader, ConfigSchema
│   │   └── Support/               # ScanMDError, ExitCode, Logging, Limits, Subprocess
│   └── scanmd/                    # CLI target (ArgumentParser commands only)
│       └── Info.plist             # embedded via -sectcreate (camera usage string)
├── App/
│   ├── project.yml                # XcodeGen spec (committed); .xcodeproj is generated
│   ├── ScanMD/                    # app sources (SwiftUI MenuBarExtra, Settings, windows)
│   ├── ScanMD/Info.plist
│   └── ScanMD/ScanMD.entitlements
├── Tests/
│   ├── ScanMDKitTests/            # unit + golden tests
│   └── Fixtures/                  # corpus (see Issue 07)
├── Packaging/
│   └── homebrew/scanmd.rb         # formula template
├── Scripts/                       # release-sign.sh, make-universal.sh, gen-fixtures…
├── .github/workflows/             # ci.yml, codeql.yml, release.yml
└── docs/                          # this design, ADRs, research, issue plan, issue drafts
```

---

## 4. Architecture

### 4.1 Pipeline

Every capture, on either surface, runs the same five stages:

```
┌────────┐   ┌────────────┐   ┌──────────────┐   ┌───────────────┐   ┌────────┐
│ Source │ → │ Recognizer │ → │ Layout       │ → │ Markdown      │ → │ Sinks  │
│        │   │ (Vision)   │   │ Reconstructor│   │ Renderer      │   │        │
└────────┘   └────────────┘   └──────────────┘   └───────────────┘   └────────┘
 acquire()     recognize()      blocks(from:)      render(_:)        deliver(_:)
 → [Page]      → [RecognizedLine]  → [Block]       → String          (stdout/clipboard/file)
```

- A `Source` yields one or more **pages**. A page is either an image (needs OCR) or
  pre-extracted text (PDF text layer — skips Recognizer and Layout, §7.4).
- Stages are pure protocol boundaries; each is one issue and independently testable.
- The `Pipeline` orchestrator owns ordering, per-page progress, limits, and error mapping.

### 4.2 Module dependency graph

```
scanmd (CLI) ─────────┐
                      ├──▶ ScanMDKit ──▶ Apple frameworks (Vision, PDFKit, AVFoundation,
ScanMD.app ───────────┘                   ImageIO, AppKit, CoreGraphics, UserNotifications)
        └──▶ KeyboardShortcuts (app only)
```

`ScanMDKit` never imports SwiftUI and never depends on the app. The CLI never links
KeyboardShortcuts. Anything usable by both surfaces lives in the Kit.

### 4.3 Concurrency model

- The pipeline is `async`; OCR of multiple PDF pages runs with a `TaskGroup` bounded to
  `min(4, ProcessInfo.activeProcessorCount)` concurrent page recognitions.
- The app serializes captures: one pipeline run at a time (state machine, §9.4).
- All Kit types are `Sendable` or actor-isolated; no shared mutable globals.

---

## 5. Core data model and protocols (normative)

Implementation agents must use these names and shapes. File: `Sources/ScanMDKit/Model/`.

### 5.1 Geometry convention

Vision returns normalized bounding boxes with **bottom-left origin**. Immediately after
recognition, convert to **top-left origin** (`y' = 1 − (y + height)`) and use top-left
everywhere downstream. Store as `NormalizedRect { x, y, width, height: Double }` (0…1).

### 5.2 Types

```swift
public enum SourceKind: String, Codable, Sendable {
  case screen, camera, imageFile, clipboard, pdf, stdin
}

public struct SourceMetadata: Codable, Sendable {
  public var kind: SourceKind
  public var originPath: String?      // absolute path for file/pdf; nil otherwise
  public var capturedAt: Date
  public var pageCount: Int
  public var ocrLanguages: [String]   // BCP-47, e.g. ["ja-JP", "en-US"]
}

public struct RecognizedLine: Codable, Sendable, Equatable {
  public var text: String
  public var bbox: NormalizedRect     // top-left origin (§5.1)
  public var confidence: Double       // 0…1 (Vision candidate confidence)
}

public enum Block: Codable, Sendable, Equatable {
  case heading(level: Int, text: String)          // level 1…3
  case paragraph(text: String)
  case listItem(ordered: Bool, indent: Int, text: String)  // indent 0…3
  case pageBreak                                   // PDF page boundary
}

public struct PageResult: Sendable {
  public var index: Int               // 0-based
  public var lines: [RecognizedLine]  // empty when text came from a PDF text layer
  public var blocks: [Block]
}

public struct ScanDocument: Sendable {
  public var source: SourceMetadata
  public var pages: [PageResult]
  public var stats: ScanStats         // lineCount, meanConfidence, elapsed per stage
}
```

### 5.3 Page payload

```swift
public enum PagePayload: Sendable {
  case image(CGImage)                 // needs OCR
  case text(String)                   // PDF text layer; skips OCR + layout
}
```

### 5.4 Stage protocols

```swift
public protocol ScanSource: Sendable {
  var kind: SourceKind { get }
  func acquire() async throws -> [PagePayload]     // ≥1 page or throws
}

public protocol TextRecognizer: Sendable {
  func recognize(_ image: CGImage, options: OCROptions) async throws -> [RecognizedLine]
}

public protocol LayoutReconstructing: Sendable {
  func blocks(from lines: [RecognizedLine], options: LayoutOptions) -> [Block]
}

public protocol MarkdownRendering: Sendable {
  func render(_ document: ScanDocument, options: RenderOptions) -> String
}

public protocol OutputSink: Sendable {
  func deliver(_ markdown: String, document: ScanDocument) throws -> DeliveryReceipt
}
```

### 5.5 `DeliveryReceipt`

`struct DeliveryReceipt { var sink: String; var location: String? }` — `FileSink` reports the
final written path (after collision suffixing); CLI prints receipts to stderr in verbose mode.

### 5.6 `Formatter` extension point (v2 hook, ships inert in v1)

```swift
public protocol MarkdownFormatter: Sendable {
  func format(_ markdown: String, document: ScanDocument) async throws -> String
}
public struct PassthroughFormatter: MarkdownFormatter { /* returns input unchanged */ }
```

The pipeline calls exactly one formatter between render and deliver. v1 registers only
`PassthroughFormatter`. An LLM formatter (v2) plugs in here without touching any v1 module
(ADR-003). No network-capable formatter may be added without a new ADR and human approval.

---

## 6. Recognition (Vision OCR)

Implementation: `VisionTextRecognizer` wrapping `VNRecognizeTextRequest`.

| Option | Default | Config key | Notes |
|---|---|---|---|
| recognitionLevel | `.accurate` | `ocr.recognitionLevel` (`"accurate"\|"fast"`) | |
| recognitionLanguages | `["ja-JP", "en-US"]` | `ocr.languages` | order = priority |
| automaticallyDetectsLanguage | `true` | `ocr.autoDetectLanguage` | macOS 14 API |
| usesLanguageCorrection | `true` | — (fixed) | |
| minimumTextHeight | `0` (off) | — (fixed v1) | |
| Candidate | `topCandidates(1)` | — | multi-candidate is v2 |

Rules:

- Discard observations whose top candidate confidence < `ocr.minConfidence` (default `0.0`,
  i.e. keep all; the value is recorded per line for stats and JSON output).
- Empty result (zero lines) is not an error at this layer; the pipeline maps a fully empty
  document to exit code 6 (§8.5).
- The recognizer is stateless and safe to call concurrently for different pages.

---

## 7. Sources (normative behavior per source)

Every source validates input against `Limits` (§10.4) **before** heavy work, and throws
`ScanMDError` cases only (§8.4).

### 7.1 ImageFileSource (`kind: .imageFile` / `.stdin`)

- Accepts PNG, JPEG, HEIC, TIFF, GIF (first frame), WebP — decoded via ImageIO
  (`CGImageSourceCreateWithURL` / `…WithData`); never a custom decoder.
- Applies EXIF orientation before OCR (`kCGImageSourceCreateThumbnailWithTransform` approach
  or `CGImagePropertyOrientation` mapping — must produce upright pixels).
- `-` reads raw image bytes from stdin into memory (same limits apply).
- Rejects: file > `limits.maxFileSizeMB`; decoded pixels > `limits.maxImagePixels`;
  undecodable data → `.unsupportedInput`.

### 7.2 ClipboardSource (`kind: .clipboard`)

- Reads `NSPasteboard.general` **once, only when the user explicitly runs the command /
  menu item** (privacy: never poll, never observe changes).
- Resolution order: (1) image data (`.tiff`/`.png` types), (2) file URLs whose UTType conforms
  to image or PDF → delegate to ImageFileSource / PDFSource for the first URL.
- Plain-text-only clipboard → `.noInput("clipboard has no image, file, or PDF")`
  (deliberate: passing text through OCR would be lossy nonsense; text passthrough is v2).

### 7.3 ScreenRegionSource (`kind: .screen`) — ADR-004

- Preflight: `CGPreflightScreenCaptureAccess()`; if false call
  `CGRequestScreenCaptureAccess()` once, re-check; still false → `.permissionDenied(.screen)`
  with remediation text (§8.6).
- Spawn `/usr/sbin/screencapture` via `posix_spawn` (argv array, no shell):
  `screencapture -i -x <tmpPath>` where `tmpPath = <NSTemporaryDirectory>/scanmd-<UUID>.png`.
- Outcome mapping: process exit ≠ 0 **or** file missing/empty → user cancelled →
  `.cancelled`. File present → load via ImageFileSource path, `chmod 0600` first.
- Always delete `tmpPath` in `defer` (success, cancel, and error paths).
- Hard timeout `limits.captureTimeoutSec` (default 120 s): kill child, `.timeout`.

### 7.4 PDFSource (`kind: .pdf`)

- Open with PDFKit; encrypted documents: attempt empty password, else
  `.unsupportedInput("encrypted PDF")` (password option is v2).
- Page selection: `--pages` accepts `N`, `N-M`, `N-`, comma lists (1-based, inclusive);
  invalid ranges → `.usage`.
- Per page: if `page.string` (trimmed) length ≥ `pdf.textLayerMinChars` (default 8) → emit
  `PagePayload.text`; else rasterize at `pdf.rasterDPI` (default 300, max 600) to CGImage →
  `PagePayload.image` (OCR fallback). `--force-ocr` rasterizes every page.
- Limits: page count after selection ≤ `pdf.maxPages` (default 500) else `.limitExceeded`.
- Text-layer pages bypass Layout (§4.1): the renderer splits text into paragraphs on blank
  lines, maps nothing else (no heading/list heuristics on extracted text in v1).
- Pages are joined with `Block.pageBreak`, rendered per `markdown.pageSeparator` (§8.3-r7).

### 7.5 CameraSource (`kind: .camera`)

- AVFoundation still capture: discovery session over
  `[.builtInWideAngleCamera, .continuityCamera, .external]`, video position `.unspecified`.
  Continuity Camera iPhones appear as regular devices — no special UI needed in v1.
- Preflight `AVCaptureDevice.authorizationStatus(for: .video)`; `.notDetermined` →
  `requestAccess` (async); denied → `.permissionDenied(.camera)`.
- CLI flow: `--list-devices` prints an indexed table and exits 0. Otherwise select device by
  `--device <index|name-substring>` or config `camera.device`, else first device.
  Warm up `camera.warmupMs` (default 800 ms) for exposure, print `3…2…1` countdown to stderr
  (suppressed by `--no-countdown`), capture one photo via `AVCapturePhotoOutput`, max wait
  `limits.captureTimeoutSec`.
- App flow reuses the same source behind a preview window (§9.3).
- CLI TCC note: camera permission requires a usage string; the CLI embeds `Info.plist` via
  linker `-sectcreate __TEXT __info_plist` (Issue 16). Attribution risk is KU-2.

---

## 8. CLI specification (normative)

Target: `scanmd` (swift-argument-parser). All human messages to **stderr**; only Markdown /
JSON payload goes to **stdout**.

### 8.1 Command grammar

```
scanmd                                  # = scanmd screen (flagship default)
scanmd screen        [global flags]
scanmd image  <path|-> [global flags]
scanmd clipboard     [global flags]
scanmd pdf    <path> [--pages <spec>] [--force-ocr] [global flags]
scanmd camera [--device <sel>] [--list-devices] [--no-countdown] [global flags]
scanmd config path                      # print resolved config file path
scanmd config init                      # write default config (never overwrites; see below)
scanmd --version | --help
```

`scanmd <path>` (bare path shorthand): if the argument is an existing file whose UTType
conforms to PDF → `pdf`, to image → `image`; otherwise exit 2 with usage help.

### 8.2 Global flags

| Flag | Type | Default | Meaning |
|---|---|---|---|
| `--copy` | bool | config `output.copyToClipboard` | also place result on clipboard |
| `--out <path>` | string | — | write to file (or into directory if `<path>` is an existing dir, using `output.filenameTemplate`) |
| `--no-stdout` | bool | false | suppress stdout payload (use with `--out`/`--copy`) |
| `--format <md\|txt\|json>` | enum | `md` | §8.3 |
| `--frontmatter` / `--no-frontmatter` | bool | config `markdown.frontmatter` | prepend YAML front matter |
| `--lang <codes>` | comma list | config `ocr.languages` | override OCR languages |
| `--fast` | bool | false | recognitionLevel = fast |
| `--quiet` | bool | false | suppress progress/receipts on stderr |
| `--verbose` | bool | false | stage timings + delivery receipts on stderr |
| `--config <path>` | string | resolved (§8.7) | explicit config file |

### 8.3 Output formats

- `md` — rendered Markdown (§8.3-rules below).
- `txt` — blocks joined as plain text, no Markdown syntax, no front matter.
- `json` — single JSON object:
  `{ "version": 1, "source": SourceMetadata, "markdown": String, "pages": [ { "index": Int, "lines": [ { "text", "bbox": {x,y,w,h}, "confidence" } ], "blocks": [Block as tagged JSON] } ], "stats": {…} }`.
  Exact Codable encoding fixed in Issue 18; `version` guards future changes.

Markdown rendering rules (normative, Issue 10):

- r1 heading → `#`/`##`/`###` + space + text; blank line before and after.
- r2 paragraph → text; paragraphs separated by one blank line.
- r3 listItem → `indent × 2 spaces` + (`- ` unordered | `1. ` ordered, always literal `1.`) + text.
- r4 CJK join: when merging physical lines inside a paragraph, insert no space if **both**
  boundary characters are CJK (Han, Hiragana, Katakana, full-width punct); else one space;
  Latin hyphenated line ends (`…-`) drop the hyphen and join with no space.
- r5 escaping (`markdown.escapeSpecials`, default `minimal`): escape only line-leading tokens
  that would change Markdown semantics (`#`, `>`, `-`, `+`, `*`, `` ` ``, digit sequences
  followed by `.` or `)`) with a backslash; mode `none` disables; mode `full` additionally
  escapes inline `*_[]()~\``. 
- r6 front matter (when enabled): YAML with keys `source` (kind), `path` (originPath or null),
  `captured` (ISO-8601 local with offset), `pages` (int), `languages` (list), `tool`
  (`scanmd/<semver>`). Nothing else in v1.
- r7 `Block.pageBreak` → `markdown.pageSeparator` default `"\n---\n"`.

### 8.4 Error taxonomy → exit codes (normative, Issue 04)

| Exit | `ScanMDError` case | Meaning / example |
|---|---|---|
| 0 | — | success (including `--no-stdout`) |
| 2 | `.usage` | bad flags, bad `--pages` spec (ArgumentParser validation errors map here) |
| 3 | `.permissionDenied(scope)` | TCC denied: screen / camera; message includes remediation (§8.6) |
| 4 | `.noInput(reason)` | file not found, clipboard empty, no camera device |
| 5 | `.unsupportedInput(reason)` | undecodable image, encrypted PDF, unknown UTType |
| 6 | `.emptyResult` | pipeline ran, zero text recognized (stdout stays empty) |
| 7 | `.deliveryFailed(reason)` | cannot write `--out`, clipboard write failed |
| 8 | `.cancelled` | user aborted region selection / camera countdown (Ctrl-C maps here) |
| 9 | `.timeout` | capture or OCR exceeded limit |
| 10 | `.limitExceeded(reason)` | file size / pixels / page count over `Limits` |
| 1 | `.internalError` | anything else; message asks to file a bug |

One line per error on stderr: `scanmd: error: <message>` (+ remediation lines where defined).

### 8.5 Empty-result rule

Recognized text totalling zero non-whitespace characters across all pages → exit 6, empty
stdout, stderr note `no text recognized`. Scripts can rely on `test -s`-style semantics.

### 8.6 Permission remediation messages (fixed strings, Issue 04)

- screen: `Screen Recording permission is required. Open System Settings → Privacy & Security → Screen Recording and enable the app that runs scanmd (e.g. your terminal), then retry.`
- camera: same pattern with `Camera` pane. The app surface adds a button that opens
  `x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture` /
  `…?Privacy_Camera`.

### 8.7 Config file

- Path resolution order: `--config <path>` → `$SCANMD_CONFIG` → `$XDG_CONFIG_HOME/scanmd/config.json` → `~/.config/scanmd/config.json` (ADR-006). The app uses the same file.
- Format: JSON (UTF-8), schema version key `"version": 1`. Unknown keys → warning to stderr
  (forward compatible); wrong types → `.usage` with JSON-path of the offending key.
- `scanmd config init` writes the full default config with 0600 permissions, creating
  `~/.config/scanmd/` (0700) if needed; refuses to overwrite an existing file (exit 7).
- Full schema with defaults (normative, Issue 06):

```jsonc
{
  "version": 1,
  "ocr":      { "languages": ["ja-JP", "en-US"], "autoDetectLanguage": true,
                "recognitionLevel": "accurate", "minConfidence": 0.0 },
  "markdown": { "headingDetection": true, "listDetection": true,
                "escapeSpecials": "minimal",            // "none" | "minimal" | "full"
                "frontmatter": false, "pageSeparator": "\n---\n" },
  "output":   { "copyToClipboard": false, "directory": null,   // null = no auto-save (app)
                "filenameTemplate": "{yyyy}{MM}{dd}-{HHmmss}-{source}.md",
                "collision": "suffix" },                 // "suffix" | "overwrite" | "fail"
  "pdf":      { "maxPages": 500, "rasterDPI": 300, "textLayerMinChars": 8 },
  "camera":   { "device": null, "warmupMs": 800 },
  "limits":   { "maxFileSizeMB": 100, "maxImagePixels": 40000000,
                "captureTimeoutSec": 120, "ocrTimeoutSecPerPage": 30 },
  "app":      { "hotkey": null,                          // managed by KeyboardShortcuts store
                "showNotification": true, "autoSave": false, "launchAtLogin": false }
}
```

- Filename template variables (exactly these in v1): `{yyyy} {MM} {dd} {HH} {mm} {ss}
  {HHmmss} {source} {slug}` where `{slug}` = first 24 chars of first heading/paragraph,
  NFKC-normalized, `[^A-Za-z0-9぀-ヿ一-鿿-]` → `-`, collapsed.

---

## 9. Menu bar app specification

### 9.1 App identity

- Bundle id `dev.saber5656.scanmd` (final value confirmed in Issue 19; KU-6). `LSUIElement =
  true` (no Dock icon). SwiftUI `MenuBarExtra` with template SF Symbol icon
  (`text.viewfinder`).

### 9.2 Menu structure (top to bottom)

```
Capture Region            ⌃⌥⌘M (global, configurable)
From Clipboard
From Image File…                     (NSOpenPanel, images)
From PDF…                            (NSOpenPanel, pdf)
From Camera…
────────────────────────
Settings…                 ⌘,
About scanmd
Quit                      ⌘Q
```

### 9.3 Camera window (app)

Borderless-titled window with `AVCaptureVideoPreviewLayer`, device picker (popup listing the
same discovery session as CLI, iPhones included), Capture button (Return) and Cancel (Esc).
Captured frame feeds the standard pipeline.

### 9.4 Capture state machine (app-global, one instance)

| State | Event | Next | Effect |
|---|---|---|---|
| idle | menu/hotkey action | acquiring | menu icon switches to filled variant |
| acquiring | payload ready | recognizing | progress spinner in menu |
| acquiring | cancel / error | idle | error → notification (§9.5) |
| recognizing | done | delivering | |
| recognizing | error/timeout | idle | notification |
| delivering | done | idle | success notification |
| any busy state | hotkey/menu action | (unchanged) | `NSSound.beep()`, event dropped |

### 9.5 Result delivery (app)

- Always: Markdown → clipboard (`NSPasteboard.general.clearContents(); setString`).
- `app.showNotification`: `UNUserNotificationCenter` banner — title `Copied as Markdown`,
  body = first 120 chars of result; on `.emptyResult` title `No text found`; on permission
  errors a notification whose action button opens the right Settings pane (§8.6).
- `app.autoSave && output.directory != null`: also write via `FileSink` (same template
  rules as CLI). Delivery failures → error notification, clipboard still holds the result.

### 9.6 Settings window (SwiftUI `Settings` scene, 3 tabs)

| Tab | Controls (bound to config file, live-reloaded) |
|---|---|
| General | copy-to-clipboard toggle (app always copies — control disabled+on in v1), auto-save toggle + folder picker, notification toggle, launch at login (`SMAppService.mainApp`) |
| OCR | language priority list (ja/en reorder, add BCP-47 code), auto-detect toggle, accurate/fast picker |
| Shortcuts | `KeyboardShortcuts.Recorder` for Capture Region |

Config writes go through `ConfigLoader.save()` (atomic temp+rename, 0600). The app watches
the file with a dispatch source and reloads on external change (last-writer-wins, no merge).

---

## 10. Security model

Threat-model detail lives in `docs/SECURITY-MODEL.md` (Issue 25); this section is the
normative summary. scanmd processes **whatever the user can see** — screenshots may contain
credentials, medical data, anything. The design treats captured content as sensitive by
default.

### 10.1 Trust boundaries

| # | Boundary | Direction | Controls |
|---|---|---|---|
| B1 | Untrusted input files (images/PDF/stdin/clipboard) → parsers | in | Apple-framework parsers only (ImageIO/PDFKit); size/pixel/page limits before decode where possible; malformed → typed error, never crash (fuzz-lite corpus, Issue 07) |
| B2 | TCC-gated OS capture (screen, camera) | in | preflight + explicit user action per capture; no background/scheduled capture in v1 |
| B3 | Child process `/usr/sbin/screencapture` | out/in | absolute path, `posix_spawn` argv (no shell), env cleared to minimal, 0600 temp file, guaranteed cleanup, timeout+kill |
| B4 | Output writes (files) | out | path canonicalization; template output confined to resolved target dir (reject `..`/absolute escapes from `{slug}` — template vars are sanitized §8.7); atomic write; 0600 files / 0700 dirs; `collision=suffix` default so nothing is silently overwritten |
| B5 | Clipboard | out (in for clipboard source) | write on success is core UX; read only on explicit command; never log clipboard contents |
| B6 | Network | — | **none in v1**: no networking symbol may appear in ScanMDKit/scanmd/app (`URLSession`, `Network`, `NW…`); CI greps for these and fails (Issue 03); app has no network entitlement need |
| B7 | Config file | in | JSON schema validation, type errors rejected; config contains no secrets in v1 |

### 10.2 Privacy rules (normative)

- p1 No captured pixels or recognized text is ever written to logs. `Logger` wrapper marks
  all content interpolations `.private`; log only lengths, counts, timings, error kinds.
- p2 Temp images: 0600, unique names under the per-user temp dir, deleted in `defer`.
- p3 No analytics, telemetry, crash reporting, or update checks in v1.
- p4 README + PRIVACY section state the zero-network guarantee (Issue 28).

### 10.3 Permissions (least privilege)

| Permission | Requested by | When | Never |
|---|---|---|---|
| Screen Recording | terminal (CLI) / app | first region capture | at launch |
| Camera | CLI binary / app | first camera capture | at launch |
| Notifications | app | first delivery | — |
| Accessibility, mic, location, network | — | never in v1 | — |

### 10.4 Resource limits (DoS resistance) — `Limits` struct defaults

`maxFileSizeMB 100 · maxImagePixels 40e6 · pdf.maxPages 500 · rasterDPI ≤ 600 ·
captureTimeoutSec 120 · ocrTimeoutSecPerPage 30`. All configurable; hard caps enforced in
code (config cannot exceed 4× default; exceeding → `.usage`).

### 10.5 App hardening (ADR-005)

Developer ID signing + notarization + hardened runtime for `ScanMD.app`; no App Sandbox in
v1 (auto-save to arbitrary user folders + `screencapture` child would require bookmark
machinery; revisit in v2). No `com.apple.security.cs.*` exception entitlements except those
notarization requires for the toolchain defaults. CLI distributed signed when maintainer
signs releases (manual gate, §12).

### 10.6 Supply chain

- Dependencies: exactly `swift-argument-parser` and `KeyboardShortcuts` (app only), pinned
  via committed `Package.resolved`; version bumps only via Dependabot PRs.
- GitHub Actions pinned to commit SHAs; workflow `permissions:` least-privilege
  (`contents: read` default); CodeQL (Swift) + Dependabot + secret scanning + push
  protection enabled (Issue 24; some toggles are manual admin steps).
- Releases publish SHA-256 checksums; signing keys never enter CI in v1 — signing and
  notarization run locally by the maintainer (user-held credentials, per repository owner
  policy that agents never handle secrets).

### 10.7 Abuse cases considered

| Case | Position |
|---|---|
| Bulk-OCR of on-screen third-party content | Same capability as built-in screenshots; every capture needs explicit user action; no batch/watch mode in v1 |
| Malicious fixture/PR against CI | CI has read-only token, no secrets, fork PRs get no elevated workflows |
| Crafted image/PDF exploiting parsers | Apple parsers + limits + graceful-failure tests (B1) |
| Tampered release binary | checksums + Developer ID signature + notarization |

---

## 11. Testing strategy

| Layer | Approach | Determinism |
|---|---|---|
| Layout, renderer, config, templates, error mapping | pure unit + golden tests (`Tests/ScanMDKitTests`) | fully deterministic — the bulk of coverage lives here |
| Vision OCR | fixture images (Issue 07: synthetic CoreGraphics-drawn ja/en text + a few real photos); assert **token recall ≥ 0.9** for expected token lists, never exact-match OCR output | tolerant of OS-version drift (KU-3) |
| Sources | ImageFile/PDF: fixtures incl. malformed + oversized (must fail with the right exit code). Screen/Camera: subprocess + AV layers behind protocol fakes for unit tests; real TCC flows via manual checklists in issues | CI-safe |
| CLI e2e | run the built binary on fixtures; assert stdout golden (for text-layer PDFs), exit codes, `--format json` schema validity | deterministic subset only |
| App | XCUITest smoke (launches, menu present); capture state machine unit-tested via extracted reducer | minimal UI testing |
| Security regressions | CI greps forbidden networking symbols (B6); fuzz-lite malformed corpus runs in normal test suite | deterministic |

Manual validation checklists (TCC prompts, notarization, Continuity Camera) are explicit
sections in the relevant issues — CI cannot grant TCC.

## 11.5 Performance targets (p50 on Apple Silicon, `.accurate`)

| Flow | Target |
|---|---|
| region capture → clipboard (1000×800 px) | ≤ 2.5 s after selection |
| single image OCR (12 MP photo) | ≤ 4 s |
| 100-page text-layer PDF | ≤ 5 s |
| memory ceiling at limits | < 1 GB RSS |

Not CI-gated in v1; measured once in Issue 26's release checklist.

---

## 12. Distribution and release

- Versioning: SemVer, tags `vX.Y.Z`; `v1.0.0` = ISSUE_PLAN v1 completion statement satisfied.
- CI release workflow (tag push): build universal CLI + app archive **unsigned**, produce
  `scanmd-<ver>-macos-universal.tar.gz`, `ScanMD-<ver>.zip`, `SHA256SUMS`, draft GitHub
  Release. Maintainer then runs `Scripts/release-sign.sh` locally (Developer ID + notarytool;
  credentials stay on the maintainer's machine), replaces artifacts, publishes.
- Homebrew: formula template `Packaging/homebrew/scanmd.rb` (builds from source);
  publication to a `Saber5656/homebrew-tap` repo is a documented manual step (Issue 27).
  Cask for the app is v2.
- License: **MIT proposed** — requires explicit maintainer approval before the LICENSE
  commit lands (Issue 02 acceptance gate).

---

## 13. Deferred to v2 (with the v1 seam that enables each)

| Item | v1 seam |
|---|---|
| LLM Markdown formatter (opt-in, key via Keychain/env) | `MarkdownFormatter` protocol (§5.6) + ADR-003 |
| Table reconstruction | bbox data already preserved in `RecognizedLine` |
| Custom region-selection overlay (replace `screencapture -i`) | `ScanSource` protocol |
| Continuity Camera document-scan sheet (`importFromDevice:`) | CameraSource swap in app |
| URL/HTML → Markdown, audio transcription | new `ScanSource` kinds |
| Clipboard plain-text passthrough, batch/watch mode, capture history UI | — |
| Homebrew cask, Sparkle updates, App Sandbox, localization | §10.5, §12 |
| PDF password option, multi-candidate OCR, `minimumTextHeight` tuning | §6, §7.4 |

---

## 14. Document map

| Document | Content |
|---|---|
| `docs/DESIGN.md` | this file — normative v1 design |
| `docs/ISSUE_PLAN.md` | waves, ordering, dependencies, coverage, validation strategy |
| `docs/issues/NN-*.md` | one implementable issue each (28 total) |
| `docs/decisions/ADR-001…006` | product shape, stack, local-only, screencapture, app packaging/hardening, config format |
| `docs/research/prior-art.md` | comparable tools and the gap scanmd fills |
| `docs/research/macos-capture-ocr-apis.md` | API landscape, TCC attribution notes, verification status |
| `docs/SECURITY-MODEL.md` | (created by Issue 25) full threat model |
