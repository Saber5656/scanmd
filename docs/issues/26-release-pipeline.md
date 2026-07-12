# Title

Release pipeline: universal builds, checksums, signing gate

## Summary

Add the tag-triggered release workflow producing unsigned universal artifacts with
SHA-256 checksums and a draft GitHub Release, plus the maintainer-local signing/
notarization script and the RELEASING.md runbook with the v1.0.0 gate checklist.

## Context

DESIGN §12: CI stays keyless; signing credentials never leave the maintainer's machine
(owner policy). The release gate also carries the §11.5 performance measurements.

## Scope

- `.github/workflows/release.yml`, `Scripts/make-universal.sh`,
  `Scripts/release-sign.sh`, `Scripts/verify-release.sh`, `docs/runbooks/RELEASING.md`,
  version stamping.

## Detailed Requirements

1. Version stamping: single source `Sources/ScanMDKit/Version.swift`
   (`public enum ScanMDVersion { public static let current = "X.Y.Z" }`, with optional
   prerelease suffix such as `"0.9.0-rc.1"` for rehearsal releases) consumed by
   CLI `--version`, front matter `tool:`, and app `MARKETING_VERSION` (xcodegen reads it
   via a small generation step or the release checklist keeps them in sync — pick one,
   document it, and enforce with a CI check comparing the three). Tag validation strips
   the leading `v` and accepts either `X.Y.Z` or `X.Y.Z-<prerelease>`; prerelease tags are
   allowed only for draft/rehearsal releases and must not satisfy the final `v1.0.0` gate.
2. `release.yml`: trigger `push: tags: ['v*']`; permissions
   `{ contents: write }` (release upload only); jobs:
   - build CLI: `swift build -c release --arch arm64 --arch x86_64` →
     `scanmd-<ver>-macos-universal.tar.gz` (binary + LICENSE + README excerpt).
   - build app: xcodegen + `xcodebuild -configuration Release CODE_SIGN_IDENTITY=-`
     → `ScanMD-<ver>-unsigned.zip` (ditto -c -k).
   - `SHA256SUMS` over both artifacts; create **draft** release with generated notes
     template (changelog section reference + checksum block + "artifacts are unsigned
     until the maintainer completes the signing gate" warning).
   - Tag-vs-Version.swift consistency check: mismatch fails the workflow.
3. `Scripts/release-sign.sh` (maintainer-local, requires Developer ID cert + notarytool
   keychain profile; **script must never echo secrets and takes no credentials as
   args** — it uses the local keychain/profile by name):
   unpack `scanmd-<ver>-macos-universal.tar.gz` → codesign CLI binary (hardened runtime,
   timestamp) → package the signed CLI into a temporary notarization zip
   `scanmd-<ver>-macos-universal-notary.zip` with `ditto -c -k --keepParent` → codesign app
   (entitlements from Issue 19) → `ditto` zip the app → `xcrun notarytool submit --wait`
   the CLI notary zip and app zip → staple app → rebuild the distributable CLI tarball
   `scanmd-<ver>-macos-universal.tar.gz` from the signed binary → rebuild `SHA256SUMS` →
   print upload commands (`gh release upload vX.Y.Z … --clobber`). The notary zip is an
   intermediate submission container and is not uploaded as the public CLI artifact.
4. `Scripts/verify-release.sh <version>`: downloads the published artifacts, verifies
   checksums, `codesign --verify --deep --strict`, `spctl --assess` (app), runs
   `./scanmd --version`, prints a table. Used by the runbook's final step.
5. `docs/runbooks/RELEASING.md` — ordered checklist: preflight (CI green on `main`,
   CHANGELOG updated, version bump PR, §11.5 performance table measured on Apple
   Silicon and pasted), tag push, wait for draft, run sign script, upload, run verify
   script, publish release, post-publish (Homebrew bump — Issue 27, GitHub Release link
   check). The §11.5 measurement procedure gets exact commands (`time scanmd …` on the
   fixture corpus, region capture measured by the stage timings in `--verbose`).
6. First release exercised end-to-end with `v0.9.0-rc.1` (pre-v1 rehearsal tag) — the
   PR for this issue links the rehearsal draft release with verify-script output.

## Acceptance Criteria

- [ ] Rehearsal tag produces a draft release with both artifacts + SHA256SUMS; version
      consistency check demonstrated (mismatched tag fails — show run).
- [ ] Sign script runs on the maintainer machine; `spctl --assess` passes on the
      stapled app; notarization log links recorded in the runbook run.
- [ ] `verify-release.sh` passes against the rehearsal release.
- [ ] No credential material appears in repo, CI logs, or script output (reviewed).
- [ ] RELEASING.md checklist complete incl. §11.5 measurement table format.

## Validation

Rehearsal release links + verify script output + maintainer sign-off comment in the PR.

## Dependencies

Issues 03, 17, 19. Soft: 02 (LICENSE inside artifacts).

## Non-goals

Homebrew publication (Issue 27), auto-update (v2), CI-side signing (v2 decision needing
a new ADR + owner approval), SBOM.

## Design References

DESIGN §11.5, §12, §10.6; ADR-005; ISSUE_PLAN KU-6.
