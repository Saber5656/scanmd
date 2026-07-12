# ADR-005: App target via XcodeGen; Developer ID + hardened runtime, no sandbox in v1

- Status: Accepted
- Date: 2026-07-12
- Decider: Fable (design), conservative default

## Context

`ScanMDKit` and the CLI are a clean SwiftPM package, but a menu bar `.app` needs a bundle
(Info.plist usage strings, entitlements, icon, signing). Building `.app` bundles from pure
SwiftPM requires custom packaging scripts and is fragile around TCC usage strings; raw
`.xcodeproj` files are hostile to mechanical edits by implementation agents.

## Decision

1. The app target lives in an Xcode project **generated from a committed
   `App/project.yml` via XcodeGen** (`brew install xcodegen`; dev-time tool, not a runtime
   dependency). The generated `.xcodeproj` is **not** committed (`.gitignore`d);
   CI runs `xcodegen generate && xcodebuild …`.
2. The app depends on `ScanMDKit` as a local Swift package reference.
3. Signing posture (v1): **Developer ID + notarization + hardened runtime**;
   **App Sandbox off** (auto-save to arbitrary folders and the `screencapture` child
   process make sandboxing costly; revisit for v2/App Store). No extra
   `com.apple.security.cs.*` exception entitlements.
4. Signing and notarization run **locally by the maintainer** (`Scripts/release-sign.sh`);
   signing credentials never enter CI or agent hands (owner policy).

## Consequences

- Project definition is reviewable YAML; low-capability agents can edit `project.yml`
  safely, and merge conflicts in `.pbxproj` are eliminated.
- CI needs `xcodegen` installed (brew step in workflow) — small, pinned.
- Unsandboxed v1 must be stated in the security model (DESIGN §10.5) and README honestly.
- CI produces unsigned artifacts only; the release flow has an explicit manual signing gate
  (DESIGN §12).

## Alternatives considered

- Pure SwiftPM + hand-rolled bundle script: workable but bespoke and brittle around
  entitlements/notarization; rejected for v1.
- Committed `.xcodeproj`: standard but agent-hostile and conflict-prone; rejected.
- Tuist: heavier tool for one small target; rejected.
