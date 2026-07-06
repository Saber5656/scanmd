# Title

Add LICENSE (MIT, approval-gated) and community health files

## Summary

Add the open-source baseline: `LICENSE` (MIT — requires explicit maintainer approval
before merge), `CONTRIBUTING.md`, `SECURITY.md` (vulnerability reporting), and GitHub
issue/PR templates.

## Context

The repository is already public but has no license, which legally means "all rights
reserved" and blocks any reuse or Homebrew distribution. DESIGN §12 proposes MIT.

## Scope

- Root files: `LICENSE`, `CONTRIBUTING.md`, `SECURITY.md`, `CODE_OF_CONDUCT.md`.
- `.github/`: `PULL_REQUEST_TEMPLATE.md`, `ISSUE_TEMPLATE/bug_report.yml`,
  `ISSUE_TEMPLATE/feature_request.yml`, `ISSUE_TEMPLATE/config.yml`.

## Detailed Requirements

1. `LICENSE`: standard MIT text, copyright line
   `Copyright (c) 2026 Saber5656`.
   **Gate: the maintainer must explicitly approve the MIT choice in the PR before merge**
   (repository owner policy: legal/release decisions need human sign-off). If the
   maintainer picks a different license, update DESIGN §12 in the same PR.
2. `SECURITY.md`: supported versions table (v1.x), private reporting via GitHub Security
   Advisories ("Report a vulnerability" button), response target within 14 days, scope
   note (local-only app; no server component), link to `docs/SECURITY-MODEL.md`
   (created later by Issue 25 — link may be added as a follow-up line there).
3. `CONTRIBUTING.md`: build prerequisites (macOS 14+, Xcode version from Issue 01), how to
   run `swift build/test` and `Scripts/format.sh`, PR rules (1 issue = 1 branch = 1 PR,
   no direct pushes to `main`, review required), note that GitHub Issues derive from
   `docs/issues/` and design changes go through `docs/DESIGN.md`.
4. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 verbatim with contact = repository
   owner's GitHub profile.
5. Bug report form fields: macOS version, scanmd version (`scanmd --version`), surface
   (CLI/app), source kind, reproduction steps, expected/actual, **reminder not to attach
   captures containing sensitive content**.
6. PR template: linked issue, summary, test evidence checklist, security-impact question
   ("does this touch a trust boundary from docs/SECURITY-MODEL.md?").

## Acceptance Criteria

- [ ] All files above exist with the specified content.
- [ ] Maintainer has explicitly approved the license choice in a PR comment (link it).
- [ ] GitHub renders the issue forms correctly (check the "New issue" chooser).
- [ ] `SECURITY.md` appears in the repository Security tab.

## Validation

Open the GitHub UI: new-issue chooser shows both forms; Security tab shows the policy;
`LICENSE` is detected by GitHub (repo sidebar shows "MIT license").

## Dependencies

None (can run parallel to Issue 01). License approval by maintainer is a merge gate.

## Non-goals

Dependabot/CodeQL/workflow hardening (Issue 24), README rewrite (Issue 28),
threat model content (Issue 25).

## Design References

DESIGN §12 (license), §10.6; ISSUE_PLAN §3 (approval gate).
