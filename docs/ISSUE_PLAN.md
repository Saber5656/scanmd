# scanmd v1 Issue Plan

Status: Active
Last updated: 2026-07-12
Derived from: `docs/DESIGN.md` (normative). GitHub Issues are generated from
`docs/issues/NN-*.md`; if they diverge, these files win. GitHub numbering is
offset by one from the file numbering (PR #1 took the first number): draft
`NN` = GitHub issue `#(NN+1)`, i.e. #2–#29.

---

## 1. v1 completion statement

v1 (`v1.0.0`) is complete when **all 28 issues below are closed and their Validation
sections pass**, which means:

1. `brew install` (from the tap formula) or a downloaded release artifact yields a working
   universal `scanmd` CLI, and `ScanMD-<ver>.zip` yields a signed, notarized menu bar app.
2. All five input sources work on macOS 14+: screen region, image file (+stdin),
   clipboard, PDF (text layer + OCR fallback), camera (built-in and Continuity iPhone).
3. Output is structured Markdown (headings/lists/paragraphs heuristics, optional YAML front
   matter) delivered to stdout, clipboard, and/or template-named files, plus `txt` and
   `json` formats, with the exact flag surface and exit codes of DESIGN §8.
4. The menu bar app provides hotkey region capture, all menu actions, settings (output /
   OCR / shortcut / login item), notifications, and auto-save — sharing the CLI's config.
5. The zero-network guarantee holds and is CI-enforced; TCC permissions are requested
   lazily with remediation guidance; the threat model doc matches the shipped code.
6. CI (build+test+lint+CodeQL), release automation with checksums, and the documented
   manual signing gate are in place; README documents install, permissions, and privacy.

Anything not covered by an issue below is a v2 item (§7) or a known unknown (§8) — no v1
behavior lives only in prose outside this plan.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave | GitHub |
|---|---|---|---|---|
| 01 | `issues/01-swiftpm-scaffold.md` | Scaffold SwiftPM package with ScanMDKit and scanmd targets | 0 | [#2](https://github.com/Saber5656/scanmd/issues/2) |
| 02 | `issues/02-community-files-license.md` | Add LICENSE (MIT, approval-gated) and community health files | 0 | [#3](https://github.com/Saber5656/scanmd/issues/3) |
| 03 | `issues/03-ci-build-test.md` | CI workflow: build, test, lint, no-network guard | 0 | [#4](https://github.com/Saber5656/scanmd/issues/4) |
| 04 | `issues/04-error-taxonomy-logging.md` | Error taxonomy, exit codes, and privacy-safe logging | 0 | [#5](https://github.com/Saber5656/scanmd/issues/5) |
| 05 | `issues/05-core-model-protocols.md` | Core data model, stage protocols, and pipeline orchestrator | 0 | [#6](https://github.com/Saber5656/scanmd/issues/6) |
| 06 | `issues/06-config-module.md` | Config schema v1: load, validate, defaults, init | 0 | [#7](https://github.com/Saber5656/scanmd/issues/7) |
| 07 | `issues/07-test-fixtures.md` | Test fixture corpus and golden-test harness | 0 | [#8](https://github.com/Saber5656/scanmd/issues/8) |
| 08 | `issues/08-vision-recognizer.md` | VisionTextRecognizer (Vision OCR wrapper) | 1 | [#9](https://github.com/Saber5656/scanmd/issues/9) |
| 09 | `issues/09-layout-reconstruction.md` | Layout reconstruction: lines → blocks heuristics | 1 | [#10](https://github.com/Saber5656/scanmd/issues/10) |
| 10 | `issues/10-markdown-renderer.md` | Markdown renderer and front matter builder | 1 | [#11](https://github.com/Saber5656/scanmd/issues/11) |
| 11 | `issues/11-output-sinks.md` | Output sinks: stdout, clipboard, file (templates) | 1 | [#12](https://github.com/Saber5656/scanmd/issues/12) |
| 12 | `issues/12-source-image-file.md` | ImageFileSource: image files and stdin | 2 | [#13](https://github.com/Saber5656/scanmd/issues/13) |
| 13 | `issues/13-source-clipboard.md` | ClipboardSource: pasteboard image and file URLs | 2 | [#14](https://github.com/Saber5656/scanmd/issues/14) |
| 14 | `issues/14-source-pdf.md` | PDFSource: text layer extraction with OCR fallback | 2 | [#15](https://github.com/Saber5656/scanmd/issues/15) |
| 15 | `issues/15-source-screen-region.md` | ScreenRegionSource: interactive capture via screencapture | 2 | [#16](https://github.com/Saber5656/scanmd/issues/16) |
| 16 | `issues/16-source-camera.md` | CameraSource: AVFoundation still capture (CLI path) | 2 | [#17](https://github.com/Saber5656/scanmd/issues/17) |
| 17 | `issues/17-cli-skeleton.md` | CLI skeleton: root command, global flags, config & image commands | 3 | [#18](https://github.com/Saber5656/scanmd/issues/18) |
| 18 | `issues/18-cli-subcommands-e2e.md` | CLI: remaining subcommands, JSON format, e2e test suite | 3 | [#19](https://github.com/Saber5656/scanmd/issues/19) |
| 19 | `issues/19-app-scaffold-xcodegen.md` | App scaffold: XcodeGen project, MenuBarExtra shell | 4 | [#20](https://github.com/Saber5656/scanmd/issues/20) |
| 20 | `issues/20-app-capture-actions.md` | App capture actions and state machine | 4 | [#21](https://github.com/Saber5656/scanmd/issues/21) |
| 21 | `issues/21-app-camera-window.md` | App camera preview window | 4 | [#22](https://github.com/Saber5656/scanmd/issues/22) |
| 22 | `issues/22-app-hotkey-settings.md` | Global hotkey, Settings window, launch at login | 4 | [#23](https://github.com/Saber5656/scanmd/issues/23) |
| 23 | `issues/23-app-result-delivery.md` | App result delivery: clipboard, notifications, auto-save | 4 | [#24](https://github.com/Saber5656/scanmd/issues/24) |
| 24 | `issues/24-repo-hardening-config.md` | Repository hardening: Dependabot, CodeQL, action pinning | 5 | [#25](https://github.com/Saber5656/scanmd/issues/25) |
| 25 | `issues/25-threat-model-doc.md` | SECURITY-MODEL.md: full threat model mapped to code | 5 | [#26](https://github.com/Saber5656/scanmd/issues/26) |
| 26 | `issues/26-release-pipeline.md` | Release pipeline: universal builds, checksums, signing gate | 5 | [#27](https://github.com/Saber5656/scanmd/issues/27) |
| 27 | `issues/27-homebrew-packaging.md` | Homebrew formula and tap publication runbook | 5 | [#28](https://github.com/Saber5656/scanmd/issues/28) |
| 28 | `issues/28-docs-readme-v1.md` | v1 documentation: README, permissions guide, CHANGELOG | 5 | [#29](https://github.com/Saber5656/scanmd/issues/29) |

## 3. Dependency table

`hard` = must be merged first. `soft` = content depends on outcome but work can start.

| Issue | Depends on (hard) | Soft |
|---|---|---|
| 01 | — | — |
| 02 | — | license text needs owner approval |
| 03 | 01 | — |
| 04 | 01 | — |
| 05 | 01, 04 | — |
| 06 | 01, 04 | — |
| 07 | 01 | — |
| 08 | 05, 07 | — |
| 09 | 05, 07 | — |
| 10 | 05, 07 | — |
| 11 | 05, 06 | — |
| 12 | 05, 06 | 07 (fixtures used in tests) |
| 13 | 05, 12 | — |
| 14 | 05, 06, 07 | — |
| 15 | 04, 05, 06, 12 | — |
| 16 | 04, 05, 06 | KU-2 validation gates approach |
| 17 | 04, 05, 06, 08, 09, 10, 11, 12 | — |
| 18 | 13, 14, 15, 16, 17 | — |
| 19 | 05 | 03 (CI job added here) |
| 20 | 12, 13, 14, 15, 19 | — |
| 21 | 16, 19, 20 | — |
| 22 | 06, 19 | — |
| 23 | 11, 20 | — |
| 24 | 03 | — |
| 25 | — | 20 (verify claims against real code before closing) |
| 26 | 03, 17, 19 | 02 (license in artifacts) |
| 27 | 26 | — |
| 28 | 18, 23, 25, 26 | 27 |

## 4. Implementation waves

| Wave | Issues | Theme | Exit criterion |
|---|---|---|---|
| 0 | 01–07 | Foundation: package, CI, errors, model, config, fixtures | `swift build && swift test` green in CI; protocols compile; fixtures committed |
| 1 | 08–11 | Recognition & rendering core | image bytes → Markdown string works via unit-level pipeline |
| 2 | 12–16 | All five sources | each source unit-tested; TCC manual checklists recorded |
| 3 | 17–18 | CLI product | full CLI surface of DESIGN §8, e2e suite green |
| 4 | 19–23 | Menu bar app | hotkey capture → notification round-trip on a dev machine |
| 5 | 24–28 | Security, release, docs | signed v1.0.0 draft release + docs complete |

Within a wave, issues are parallelizable unless the dependency table says otherwise
(1 issue = 1 branch = 1 PR, reviewed before merge).

## 5. Coverage table (DESIGN.md → issues)

| DESIGN section | Covered by |
|---|---|
| §2 platforms/toolchain | 01, 03 |
| §3 repo layout | 01, 19 |
| §4 architecture/pipeline/concurrency | 05 |
| §5 data model & protocols | 05 |
| §6 recognition | 08 |
| §7.1 image / §7.2 clipboard / §7.3 screen / §7.4 pdf / §7.5 camera | 12 / 13 / 15 / 14 / 16 |
| §8.1–8.2 CLI grammar & flags | 17, 18 |
| §8.3 formats & rendering rules | 09, 10, 18 |
| §8.4–8.6 errors, exit codes, remediation | 04, 17 |
| §8.7 config | 06 |
| §9 app spec (§9.1–9.2 / §9.3 / §9.4 / §9.5 / §9.6) | 19–20 / 21 / 20 / 23 / 22 |
| §10 security model (B1–B7, p1–p4, limits, hardening, supply chain) | 24, 25 + per-issue security acceptance criteria (every boundary issue restates its controls) |
| §11 testing strategy | 07 + Validation sections of every issue |
| §11.5 performance targets | 26 (release checklist measurement) |
| §12 distribution/release | 02, 26, 27 |
| §13 v2 deferrals | §7 below (no v1 issue) |
| §14 doc map | 25, 28 |

## 6. Whole-product validation strategy

1. **Deterministic core**: layout/renderer/config/template/error mapping are pure and
   golden-tested (wave 0–1) — this is where correctness lives.
2. **OCR tolerance**: fixture-based token-recall ≥ 0.9 assertions, never exact OCR strings
   (KU-3).
3. **e2e**: Issue 18's suite runs the real binary over the fixture corpus asserting stdout,
   exit codes, and JSON schema; runs in CI on every PR from wave 3 on.
4. **TCC / hardware flows**: manual validation checklists inside Issues 15, 16, 21, 22, 23,
   26 — each checklist result must be pasted into the PR before merge.
5. **Security regressions**: CI no-network grep (03), malformed-input corpus in the normal
   test suite (07, 12, 14), CodeQL (24), threat-model-vs-code review gate (25).
6. **Release gate**: Issue 26's checklist (performance targets §11.5, signing,
   notarization, checksum verification, install-from-artifact smoke test) blocks `v1.0.0`.

## 7. Deferred v2 items (not planned as issues)

LLM formatter (opt-in; Keychain/env key; new ADR + threat model required) · table
reconstruction · custom region-selection overlay (ScreenCaptureKit) · Continuity Camera
document-scan sheet · URL/HTML ingestion · audio transcription · clipboard plain-text
passthrough · batch/watch mode · capture history UI · Homebrew cask · Sparkle/auto-update ·
App Sandbox / App Store · UI localization · PDF password option · multi-candidate OCR.
Seams that keep these cheap are listed in DESIGN §13.

## 8. Known unknowns (may spawn new issues during implementation)

| ID | Unknown | Where handled | Contingency |
|---|---|---|---|
| KU-1 | Exact Xcode/Swift pin available on CI runners at implementation time | 01, 03 | pick newest stable at Issue-01 time; record in `README` + `ci.yml` |
| KU-2 | Camera TCC prompt attribution for a terminal-spawned CLI with `-sectcreate` Info.plist | 16 (validated first) | fallback: CLI camera returns guided error pointing to app; app path (21) unaffected |
| KU-3 | Vision OCR output drift across macOS versions | 07, 08 | recall-threshold tests; adjust fixtures only |
| KU-4 | KeyboardShortcuts library version compatible with macOS 14 target | 22 | pin known-good version; worst case implement Carbon hotkey wrapper (new issue) |
| KU-5 | `screencapture -i` cancel exit-status/file matrix across macOS 14/15 | 15 | tolerant detection (nonzero exit OR missing/empty file) + manual matrix |
| KU-6 | Final bundle id + Developer ID team for signing | 19, 26 | owner supplies at signing time; CI artifacts stay unsigned regardless |
| KU-7 | Notification delivery reliability for unsigned dev builds | 23 | manual checklist uses signed dev build if needed |
| KU-8 | Heading/list heuristic thresholds vs real-world captures | 09 | thresholds are constants in one file + calibration fixtures; retune may spawn follow-up issue |
