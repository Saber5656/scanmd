# Title

Repository hardening: Dependabot, CodeQL, action pinning

## Summary

Add the config-as-code security layer — Dependabot (SwiftPM + Actions), CodeQL for
Swift, an action-pinning audit — and document the admin-console toggles the maintainer
must set by hand.

## Context

DESIGN §10.6 requires supply-chain controls before v1 release. Some controls are
repository files (this issue); some are GitHub admin settings that only the owner can
toggle (documented as a manual runbook, per the owner's rule that agents don't change
security-sensitive settings unilaterally).

## Scope

- `.github/dependabot.yml`, `.github/workflows/codeql.yml`,
  `docs/runbooks/repo-security-settings.md`, pinning audit of existing workflows.

## Detailed Requirements

1. `dependabot.yml`: ecosystems `swift` (root `/`, weekly, grouped minor+patch) and
   `github-actions` (`/`, weekly). Labels: `dependencies`. PR limit 5.
2. `codeql.yml`: language `swift`, build via the same `swift build` used in CI
   (macOS runner); triggers: PR to `main`, push to `main`, weekly cron; permissions
   exactly `{ contents: read, security-events: write }`; actions SHA-pinned.
   Note: Swift CodeQL requires a build — reuse the cache strategy from Issue 03; if
   CodeQL's Swift support on macOS runners proves broken at implementation time, record
   the blocker in the PR, disable the workflow with a dated comment, and open a
   follow-up issue (do not ship a red default branch).
3. Pinning audit: verify every workflow (`ci.yml`, `codeql.yml`, and Issue 19's app job)
   uses 40-char SHA pins with version comments; add
   `Scripts/check-action-pins.sh` (fails on any `uses:` with a tag/branch ref) and run
   it in CI.
4. `docs/runbooks/repo-security-settings.md` — manual checklist for the maintainer with
   exact console paths (Settings → Code security and analysis, Settings → Rules):
   secret scanning ON, push protection ON, Dependabot alerts ON, private vulnerability
   reporting ON, branch ruleset for `main` (require PR, require CI + CodeQL checks,
   block force pushes — matching the owner's standing repo rules), Actions default
   token = read-only, fork PR workflow approval required. Each item gets a checkbox and
   a "verified on <date> by <user>" line the maintainer fills in.
5. This issue must NOT change repository admin settings itself; it ships files + the
   runbook, and its PR description pings the maintainer to execute the runbook.

## Acceptance Criteria

- [ ] Dependabot opens its first update PRs (or "no updates" state visible in the
      Insights → Dependency graph → Dependabot tab).
- [ ] CodeQL run green on `main` (or the documented-blocker path taken).
- [ ] `Scripts/check-action-pins.sh` green in CI and proven to fail on a tag ref
      (scratch commit demonstrated in PR, then dropped).
- [ ] Runbook committed; maintainer confirms execution in the PR (checklist filled).

## Validation

Links to: Dependabot tab, CodeQL run, CI run with the pin check. Maintainer comment
confirming runbook completion.

## Dependencies

Issue 03.

## Non-goals

OpenSSF Scorecard, SBOM, signing (Issue 26), SECURITY-MODEL content (Issue 25).

## Design References

DESIGN §10.6; owner policy (agents don't flip security settings; ruleset already
protects `main`).
