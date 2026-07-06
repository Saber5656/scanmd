# Title

v1 documentation: README, permissions guide, CHANGELOG

## Summary

Rewrite `README.md` as the public front door (install, usage, permissions, privacy
guarantee), add the permissions guide with screenshots, bootstrap `CHANGELOG.md`, and
align all help texts — the final gate before `v1.0.0`.

## Context

The README currently holds one concept line (Japanese). As the repo is public and
OSS-released, the primary README becomes English with a Japanese section preserved
(repo language rule: issue drafts/GitHub artifacts are English; README may be
bilingual).

## Scope

- `README.md`, `README.ja.md`, `docs/PERMISSIONS.md`, `CHANGELOG.md`; help-text/doc
  consistency pass. No behavior changes (doc-only except trivial help-string fixes).

## Detailed Requirements

1. `README.md` (English, sections in order):
   - hero: one-paragraph pitch (the README concept line translated: "an intake that
     turns anything you can see — screen, camera, files — into Markdown"), badge row
     (CI, release, license), animated capture demo placeholder (asset optional in v1;
     placeholder issue-linked if omitted);
   - **Privacy guarantee** callout near the top: 100% on-device, zero network, verified
     in CI — link to `docs/SECURITY-MODEL.md` (ADR-003);
   - Install: Homebrew (CLI), GitHub Releases (app zip + Gatekeeper first-open note),
     build-from-source;
   - Quickstart: the six DESIGN §1.2 use cases as copy-paste commands / hotkey steps;
   - CLI reference: generated from `--help` (subcommands + global flags table matching
     DESIGN §8.2 exactly — a CI check greps flag names between README and `--help`
     output to prevent drift);
   - App tour: menu, hotkey, settings tabs (3 screenshots), notification behavior;
   - Configuration: full commented default config (from `defaultConfigJSON()`) with the
     ADR-006 path rules and template variables table;
   - Exit codes table (§8.4 verbatim);
   - Permissions summary linking to `docs/PERMISSIONS.md`;
   - Limitations (v1 honest list: no tables, single-column reading order, no sandbox,
     macOS 14+ only) and Roadmap-ish pointer to `docs/ISSUE_PLAN.md` §7;
   - Contributing/License footer.
2. `README.ja.md`: full Japanese translation (not a summary); `README.md` links to it
   at the top (`日本語版はこちら`).
3. `docs/PERMISSIONS.md`: per-permission page — why scanmd needs it, exactly when the
   prompt appears, which process must be granted (Terminal vs ScanMD.app — the
   ADR-004/KU-2 attribution findings, with the measured behavior from Issues 15/16),
   reset commands (`tccutil reset ScreenCapture`, `… Camera`), screenshots of both
   System Settings panes.
4. `CHANGELOG.md`: Keep a Changelog format; `[1.0.0]` section summarizing the product
   (not per-PR noise); `[Unreleased]` scaffold; release procedure cross-ref to
   RELEASING.md.
5. Consistency pass (checklist in PR): `--help` strings ↔ README ↔ DESIGN §8 tables;
   config keys ↔ §8.7 ↔ README; error messages ↔ §8.4/§8.6; version references single-
   sourced (Issue 26). Trivial help-string fixes allowed here; anything larger → issue.
6. All docs pass a markdown linter (`markdownlint` config committed, CI job added —
   docs-only, fast).

## Acceptance Criteria

- [ ] README renders correctly on GitHub (check anchors, tables, badges); English
      primary + complete Japanese version.
- [ ] README-vs-help drift check runs in CI and passes.
- [ ] PERMISSIONS.md contains real screenshots of both panes and measured attribution
      notes (not folklore).
- [ ] CHANGELOG `[1.0.0]` ready to cut; markdownlint green.
- [ ] A newcomer following README alone can install, grant permissions, and complete
      use cases U1–U3 (tested by the maintainer or a fresh-machine VM run — evidence in
      PR).

## Validation

Rendered-README review, CI link (drift + lint), fresh-environment walkthrough notes.

## Dependencies

Issues 18, 23, 26. Soft: 27 (brew command syntax final).

## Non-goals

Website/GitHub Pages, man page, localization beyond Japanese README (v2), video demo
production.

## Design References

DESIGN §1, §8, §12, §14; ADR-003 (privacy claim), ADR-006; ISSUE_PLAN §1 (completion
statement).
