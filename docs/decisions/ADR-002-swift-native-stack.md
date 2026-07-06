# ADR-002: Swift-native stack (Vision OCR, SwiftPM, macOS 14+, universal binary)

- Status: Accepted
- Date: 2026-07-07
- Decider: repository owner (explicit answer to design questionnaire)

## Context

The pipeline needs on-device OCR with strong Japanese accuracy, screen/camera/PDF access,
and simple distribution. Candidate stacks: Swift native, Python (+ ocrmac), Rust/Go
(+ Tesseract).

## Decision

- Language/runtime: **Swift**, SwiftPM package, Swift 5.10+ with strict concurrency.
- OCR: **Apple Vision** `VNRecognizeTextRequest` (`.accurate`, language correction,
  auto language detection; priority `["ja-JP","en-US"]`).
- OS integration: ImageIO (images), PDFKit (PDF), AVFoundation (camera incl. Continuity
  Camera devices), `/usr/sbin/screencapture` (region, see ADR-004), AppKit pasteboard.
- Minimum deployment target: **macOS 14.0**; artifacts are universal (arm64 + x86_64).
- Distribution: single-binary CLI via Homebrew formula; signed/notarized `.app` via GitHub
  Releases.

## Consequences

- macOS-only product; cross-platform is out of scope for v1 and likely ever (Vision-bound).
- Japanese OCR quality tracks Apple's models — no model files to ship or update ourselves.
- Zero runtime dependencies beyond the OS; third-party Swift deps limited to
  `swift-argument-parser` (CLI) and `KeyboardShortcuts` (app hotkey).
- OCR output is not bit-stable across macOS versions; tests must assert token recall, not
  exact strings (DESIGN §11, KU-3).

## Alternatives considered

- Python + ocrmac: fastest to prototype, but distribution (pipx/uv), startup latency, and
  TCC prompt attribution for interpreter processes are all worse; rejected.
- Rust/Go + Tesseract: portable, but materially worse Japanese accuracy and no integrated
  screen/camera path on macOS; rejected.
- New macOS 26 `RecognizeDocumentsRequest` (structured document OCR): promising for v2
  table support but too new to require in v1; would raise the OS floor drastically.
