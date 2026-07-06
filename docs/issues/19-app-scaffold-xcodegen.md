# Title

App scaffold: XcodeGen project, MenuBarExtra shell

## Summary

Create `ScanMD.app`'s project structure: committed `App/project.yml` (XcodeGen), a
SwiftUI `MenuBarExtra` shell with the full menu (actions stubbed), Info.plist with usage
strings, entitlements, and the CI job that builds the app.

## Context

ADR-005 fixes the packaging approach (XcodeGen; Developer ID + hardened runtime; no
sandbox in v1). This issue makes the app build and show its menu; behavior arrives in
Issues 20–23.

## Scope

- `App/project.yml`, `App/ScanMD/` sources, `Info.plist`, `ScanMD.entitlements`,
  app icon placeholder, CI job addition, `.gitignore` already covers `*.xcodeproj`.

## Detailed Requirements

1. `App/project.yml` (XcodeGen):
   - project name `ScanMD`; one app target `ScanMD` (platform macOS, deployment 14.0).
   - `PRODUCT_BUNDLE_IDENTIFIER = dev.saber5656.scanmd` (KU-6: owner confirms or
     replaces at signing time; keep it a single build setting).
   - Local package dependency on the repo root package → product `ScanMDKit`.
   - Remote package `KeyboardShortcuts` (sindresorhus) pinned `exact` — added HERE so
     project.yml churn stays in one issue, used in Issue 22.
   - Build settings: `ENABLE_HARDENED_RUNTIME = YES`, `CODE_SIGN_STYLE = Automatic`
     with empty team (local dev signs ad-hoc; release signing is Issue 26),
     `SWIFT_STRICT_CONCURRENCY = complete`, `MARKETING_VERSION = 0.1.0`,
     `CURRENT_PROJECT_VERSION = 1`.
2. `Info.plist`: `LSUIElement = true`; `NSCameraUsageDescription` (same string as CLI,
   Issue 16); NO other usage strings (screen recording needs none — TCC prompts via
   preflight API); `CFBundleDisplayName = ScanMD`.
3. `ScanMD.entitlements`: hardened-runtime compatible, **empty of exceptions** (no
   `com.apple.security.cs.*`, no sandbox key). A comment header explains ADR-005.
4. App code (`App/ScanMD/`):
   - `ScanMDApp.swift`: `@main`, `MenuBarExtra("ScanMD", systemImage:
     "text.viewfinder")` with `.menuBarExtraStyle(.menu)`.
   - `AppMenu.swift`: the exact DESIGN §9.2 menu; every capture item calls
     `CaptureCoordinator.shared.request(.…)` — coordinator in this issue is a stub enum
     + `NSLog`-free no-op that shows a "Not implemented yet" alert; Settings item opens
     the (empty) `Settings` scene; Quit works; About shows version from bundle.
   - `CaptureCoordinator.swift`: skeleton `@MainActor final class` with the §9.4 state
     enum (`idle/acquiring/recognizing/delivering`) and a `state` published property —
     transitions implemented, effects stubbed (Issue 20 fills them). Menu icon switches
     to `text.viewfinder.fill` when not idle (SwiftUI binding).
5. CI (extends Issue 03's workflow): job `build-app` — `brew install xcodegen` (pinned
   formula version comment), `xcodegen generate --spec App/project.yml`,
   `xcodebuild -project App/ScanMD.xcodeproj -scheme ScanMD -configuration Release
   CODE_SIGN_IDENTITY=- build`. The no-network guard (Issue 03) now also scans
   `App/ScanMD/`.
6. Add `Scripts/dev-app.sh`: xcodegen + xcodebuild + `open` the built app for local
   testing.

## Acceptance Criteria

- [ ] `Scripts/dev-app.sh` produces a running menu bar app showing the full §9.2 menu;
      stub alert appears for capture items; Quit/About work.
- [ ] `xcodegen generate` is deterministic (running twice yields identical project) and
      `.xcodeproj` stays untracked.
- [ ] CI `build-app` job green; guard scans app sources.
- [ ] Entitlements file has no exception entitlements; `LSUIElement` true (no Dock icon
      at runtime — manual check).
- [ ] State machine transitions unit-tested (extracted reducer function, no UI).

## Validation

CI link + screenshot of the open menu + `codesign -d --entitlements - <app>` output in
the PR.

## Dependencies

Issue 05 (Kit builds). Soft: 03 (workflow file exists to extend).

## Non-goals

Real capture actions (20), camera window (21), hotkey/settings content (22),
notifications/delivery (23), release signing (26).

## Design References

DESIGN §9.1, §9.2, §9.4; ADR-005; ISSUE_PLAN KU-6.
