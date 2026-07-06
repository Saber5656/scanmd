# Title

Scaffold SwiftPM package with ScanMDKit and scanmd targets

## Summary

Create the Swift package skeleton: a `ScanMDKit` library target, a `scanmd` executable
target, a test target, formatting configuration, and `.gitignore` — everything compiling
and testing green on macOS 14+.

## Context

The repository currently contains only `README.md`. Every other issue builds on this
package layout (DESIGN §3). The app target is *not* part of this issue (Issue 19).

## Scope

- `Package.swift`, directory skeleton under `Sources/` and `Tests/`, lint/format config,
  `.gitignore`, minimal placeholder code proving the target graph works.

## Detailed Requirements

1. `Package.swift`:
   - `swift-tools-version` = newest stable available at implementation time (KU-1); record
     the chosen Swift/Xcode versions in the PR description.
   - `platforms: [.macOS(.v14)]`.
   - Products: `.library(name: "ScanMDKit", targets: ["ScanMDKit"])`,
     `.executable(name: "scanmd", targets: ["scanmd"])`.
   - Dependency: `swift-argument-parser` (from Apple), pinned with `exact:` or
     `from:` + committed `Package.resolved`. No other dependencies.
   - Targets: `ScanMDKit` (no dependencies), `scanmd` (depends on `ScanMDKit` +
     `ArgumentParser`), `ScanMDKitTests` (depends on `ScanMDKit`).
   - Enable strict concurrency: `swiftSettings: [.enableExperimentalFeature("StrictConcurrency")]`
     or the stable equivalent flag for the chosen toolchain, on both non-test targets.
2. Directory skeleton (empty enum namespaces or doc-comment placeholder files are fine —
   compilable, no logic): `Sources/ScanMDKit/{Model,Pipeline,Recognize,Layout,Render,Sources,Sinks,Config,Support}/`,
   `Sources/scanmd/main entry` (ArgumentParser `@main` struct named `ScanMD`, one working
   `--version` flag printing `0.0.1-dev`), `Tests/ScanMDKitTests/SmokeTests.swift` with one
   test asserting `true` plus one instantiating a Kit symbol.
3. `.gitignore`: `.build/`, `.swiftpm/`, `DerivedData/`, `*.xcodeproj` (generated later by
   XcodeGen, ADR-005), `.DS_Store`, `Tests/.tmp/`.
4. Formatting: commit `.swift-format` (Apple swift-format) configuration; add
   `Scripts/format.sh` running `swift format lint --strict --recursive Sources Tests`
   (adjust invocation to the toolchain's bundled swift-format).
5. Do not implement any pipeline logic, OCR, or CLI subcommands here.

## Acceptance Criteria

- [ ] `swift build` and `swift test` succeed on a clean checkout (macOS 14+).
- [ ] `swift run scanmd --version` prints `0.0.1-dev` and exits 0.
- [ ] `Package.resolved` is committed; the only dependency is swift-argument-parser.
- [ ] Directory skeleton matches DESIGN §3 exactly (paths listed above exist).
- [ ] `Scripts/format.sh` passes on the committed tree.
- [ ] PR description records chosen Swift tools version and Xcode version (KU-1).

## Validation

Run locally: `swift build && swift test && swift run scanmd --version &&
Scripts/format.sh`. Paste output into the PR.

## Dependencies

None (first issue).

## Non-goals

App target (Issue 19), CI workflow (Issue 03), any real pipeline code (Issues 04–11).

## Design References

DESIGN §2 (toolchain), §3 (repo layout); ADR-002; ISSUE_PLAN KU-1.
