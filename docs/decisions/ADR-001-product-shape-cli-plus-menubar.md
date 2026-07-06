# ADR-001: v1 ships both a CLI and a menu bar app on one shared core

- Status: Accepted
- Date: 2026-07-07
- Decider: repository owner (explicit answer to design questionnaire)

## Context

scanmd's README defines an "intake" that converts screen, camera, file, and other text
sources into Markdown. Two usage styles compete: terminal/pipeline usage (scripts, agents,
launchers) and instant hotkey capture during meetings/reading. A CLI-only v1 would be
smaller; an app-only v1 would lose automation.

## Decision

v1 ships **two surfaces**:

1. `scanmd` — a CLI executable (SwiftPM target), pipe-friendly.
2. `ScanMD.app` — an `LSUIElement` menu bar app with a global hotkey.

Both are thin frontends over a single library, `ScanMDKit`, which owns the entire
capture → OCR → layout → Markdown → delivery pipeline. Nothing user-visible may be
implemented twice; shared behavior belongs in the Kit.

## Consequences

- Roughly doubles v1 scope versus CLI-only: app scaffold, hotkey, settings UI, notifications,
  signing/notarization all become v1 issues (accepted knowingly by the owner).
- Requires an Xcode-buildable app target alongside SwiftPM (see ADR-005) and a Developer ID
  signing step at release (manual, maintainer-held credentials).
- Architecture rule: `ScanMDKit` must not import SwiftUI or depend on app-only frameworks;
  the CLI must not link the hotkey dependency. Enforced by target layout in `Package.swift`.

## Alternatives considered

- CLI only, GUI in v2 — rejected by owner: hotkey capture is a core daily-use case.
- Menu bar app only — rejected: kills agent/pipeline integration (`--format json`, stdout).
