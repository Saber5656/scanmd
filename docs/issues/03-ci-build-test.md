# Title

CI workflow: build, test, lint, no-network guard

## Summary

Add `.github/workflows/ci.yml` running build, tests, format lint, and the zero-network
source guard on every PR and push to `main`, with least-privilege permissions and
SHA-pinned actions.

## Context

DESIGN §10.1-B6 makes "no networking code" a CI-enforced security invariant. DESIGN §10.6
requires pinned actions and minimal token permissions from the first workflow onward.

## Scope

- `.github/workflows/ci.yml` only (CodeQL and release workflows are Issues 24 and 26).
- `Scripts/check-no-network.sh` (the guard, reusable locally).

## Detailed Requirements

1. Triggers: `pull_request` (all branches) and `push` to `main`. Add
   `concurrency: { group: ci-${{ github.ref }}, cancel-in-progress: true }`.
2. Top-level `permissions: { contents: read }` — nothing else.
3. Single job `build-test` on the newest stable macOS runner (e.g. `macos-15`; record
   choice per KU-1):
   - checkout (`actions/checkout` pinned to a full commit SHA, `persist-credentials: false`),
   - select Xcode explicitly (`sudo xcode-select -s /Applications/Xcode_<ver>.app` or
     `maxim-lobanov/setup-xcode` pinned by SHA),
   - `swift build -c release`,
   - `swift test`,
   - `Scripts/format.sh`,
   - `Scripts/check-no-network.sh`.
4. `Scripts/check-no-network.sh` (bash, `set -euo pipefail`): greps product sources
   (`Sources/`, later also `App/ScanMD/`) with an extended denylist and fails if any match:
   imports/frameworks (`import Network`, `CFNetwork`, `Network.framework`), request/session
   APIs (`URLSession`, `URLRequest`, `NSURLConnection`, `NWConnection`, `NWListener`,
   `NWEthernetChannel`, `WebSocket`, `AsyncHTTPClient`), network URL reads
   (`Data\\s*\\(\\s*contentsOf:\\s*URL\\s*\\(`, `String\\s*\\(\\s*contentsOf:\\s*URL\\s*\\(`),
   stream/socket APIs (`InputStream\\s*\\(\\s*url:`, `OutputStream\\s*\\(\\s*url:`,
   `getaddrinfo`, `connect\\s*\\(`), and shell-outs (`curl `, `wget `,
   `Process\\(.*(curl|wget)`). The guard also fails on `http://` or `https://` string
   literals in product sources unless explicitly allowlisted. Allow an annotated escape
   hatch ONLY via `// scanmd:allow-network-symbol <reason>` on the same line, which the
   script counts and reports (target count for v1: 0; a nonzero count still fails unless
   the line also appears in `Scripts/no-network-allowlist.txt`). Test sources are exempt.
5. Fail-fast quality: workflow must complete < 15 min; cache SwiftPM
   (`~/.swiftpm`, `.build`) keyed on `Package.resolved` hash with a pinned cache action.
6. Add a status badge to README (one line; full README rewrite stays in Issue 28).

## Acceptance Criteria

- [ ] CI runs on a test PR and passes; failing any of build/test/lint/guard fails the run.
- [ ] Every `uses:` is pinned to a 40-char commit SHA with a trailing version comment.
- [ ] Workflow permissions block is exactly `contents: read`.
- [ ] Introducing `URLSession` into `Sources/ScanMDKit` in a scratch commit makes CI fail
      (demonstrate in PR, then drop the scratch commit).
- [ ] Runner image + Xcode version documented in workflow comments (KU-1).

## Validation

Open a draft PR with an intentional guard violation to show the red run, then push the
clean version showing green. Link both runs in the PR.

## Dependencies

Issue 01 (package must build).

## Non-goals

CodeQL, Dependabot, scorecards (Issue 24); release artifacts (Issue 26); app build job
(added by Issue 19).

## Design References

DESIGN §10.1-B6, §10.6, §11; ADR-003 (enforcement); ISSUE_PLAN KU-1.
