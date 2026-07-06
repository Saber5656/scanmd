# Title

Homebrew formula and tap publication runbook

## Summary

Author the Homebrew formula template for the CLI (build-from-source), the tap
publication runbook, and the post-release bump procedure — tap repo creation itself is
a documented manual step for the maintainer.

## Context

DESIGN §12: CLI distribution via `brew install saber5656/tap/scanmd`. The tap lives in
a separate repository (`Saber5656/homebrew-tap`), which is outside this repo's scope —
hence template + runbook here, execution by the maintainer.

## Scope

- `Packaging/homebrew/scanmd.rb` (template), `docs/runbooks/HOMEBREW.md`,
  `Scripts/update-homebrew-formula.sh`.

## Detailed Requirements

1. `scanmd.rb` template:
   - `desc` from README concept line (English); `homepage` repo URL; `license "MIT"`
     (matches Issue 02 outcome — parameterized note if license changed);
   - `url "https://github.com/Saber5656/scanmd/archive/refs/tags/v{{VERSION}}.tar.gz"` +
     `sha256 "{{SHA256}}"`;
   - `depends_on xcode: ["<min version from Issue 01>", :build]`,
     `depends_on macos: :sonoma`;
   - `def install`: `system "swift", "build", "-c", "release",
     "--disable-sandbox"` … install `.build/release/scanmd` to `bin` — verify the
     `-sectcreate` Info.plist linker flags survive the Homebrew build (camera usage
     string check: `strings` test from Issue 16 repeated in `test do`);
   - `test do`: `assert_match version.to_s, shell_output("#{bin}/scanmd --version")`
     plus the strings check above.
2. `Scripts/update-homebrew-formula.sh <version>`: downloads the tag tarball, computes
   sha256, renders the template placeholders, writes the concrete formula to stdout or
   `--out <path>` (pointing at a local tap checkout). No network beyond the tarball
   download; no git operations (the maintainer commits to the tap).
3. `docs/runbooks/HOMEBREW.md`:
   - one-time: create `Saber5656/homebrew-tap` (public, `Formula/` layout) — exact
     `gh repo create` command listed but **executed by the maintainer**;
   - per release (after Issue 26's publish step): run the update script, commit the
     formula to the tap, `brew install --build-from-source saber5656/tap/scanmd`
     smoke test, `brew test scanmd`, `brew audit --strict scanmd` — all outputs pasted
     into the release checklist;
   - troubleshooting: Xcode CLT vs full Xcode note, `--disable-sandbox` rationale
     (SwiftPM fetching within brew sandbox).
4. Formula install must not require the app (CLI-only distribution; app ships via
   GitHub Releases zip — stated in the runbook and README wording for Issue 28).

## Acceptance Criteria

- [ ] Template renders into a formula that passes `brew audit --strict` and installs
      from the rehearsal (or first real) tag on a clean machine/VM.
- [ ] `brew test scanmd` green; `scanmd --version` matches the tag; camera usage string
      present in the brewed binary.
- [ ] Runbook executed once end-to-end by the maintainer (evidence: tap commit link +
      audit/install transcript in the PR).
- [ ] Update script is idempotent and network-minimal (tarball fetch only).

## Validation

Transcript of `brew audit/install/test` + tap repository link in the PR.

## Dependencies

Issue 26 (a published tag with artifacts must exist).

## Non-goals

Homebrew cask for the app (v2), homebrew-core submission (needs notability; revisit
post-v1), bottles/prebuilt binaries in the tap (v2).

## Design References

DESIGN §12; Issue 16 (embedded Info.plist survival); owner Git rules (maintainer
executes cross-repo actions).
