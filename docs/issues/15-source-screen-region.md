# Title

ScreenRegionSource: interactive capture via screencapture

## Summary

Implement `ScreenRegionSource: ScanSource` that preflights Screen Recording permission
and drives `/usr/sbin/screencapture -i -x` through a hardened subprocess helper, mapping
cancel/timeout/permission outcomes to typed errors.

## Context

Flagship capture flow (ADR-004; trust boundaries B2 + B3). The subprocess helper built
here (`Support/Subprocess.swift`) is the only place scanmd spawns child processes.

## Scope

- `Sources/ScanMDKit/Support/Subprocess.swift` (generic, hardened).
- `Sources/ScanMDKit/Sources/ScreenRegionSource.swift`.
- Unit tests with a fake executor + manual TCC checklist.

## Detailed Requirements

1. `Subprocess.run(executable: String, args: [String], timeout: Duration) async throws
   -> (status: Int32)`:
   - `executable` must be an absolute path (precondition).
   - `posix_spawn` (via `Process` is acceptable if configured equivalently) with:
     argv exactly as given (no shell, ever), environment reduced to
     `["PATH": "/usr/bin:/bin:/usr/sbin:/sbin"]`, cwd = temp dir, stdin `/dev/null`,
     stdout+stderr captured to memory buffers capped at 64 KB (excess discarded).
   - Timeout → `SIGTERM`, 2 s grace, `SIGKILL` → throw `.timeout("screencapture")`.
   - Ctrl-C propagation: task cancellation kills the child (test via cooperative
     cancellation), mapping to `.cancelled` at the source layer.
2. `ScreenRegionSource.acquire()`:
   a. Preflight: `CGPreflightScreenCaptureAccess()`; if false →
      `CGRequestScreenCaptureAccess()` (triggers the system prompt at most once),
      re-check; still false → `.permissionDenied(.screen)`.
   b. Temp path: `FileManager.default.temporaryDirectory/scanmd-<UUID>.png`.
   c. Run `/usr/sbin/screencapture` args `["-i", "-x", tmpPath]`, timeout
      `limits.captureTimeoutSec`.
   d. Outcome mapping (KU-5 tolerant matrix): nonzero status **or** file missing **or**
      file size 0 → `.cancelled`. Else `chmod 0600` the file, decode via Issue 12's
      `ImageDecoder` (full limits apply), return one image payload, kind `.screen`.
   e. `defer { try? FileManager.default.removeItem(at: tmpURL) }` — runs on success,
      cancel, decode failure, and timeout paths alike (test all four with the fake).
3. Fake executor for tests: `SubprocessRunning` protocol; fake simulates (status, file
   side effect) combos. Real `screencapture` is never invoked in CI.
4. No content logging; log only event names + durations (Issue 04 wrapper).
5. Manual validation checklist (execute on macOS 14 AND 15; paste results in PR):
   - [ ] Fresh TCC state (`tccutil reset ScreenCapture <bundle-or-terminal>`): first run
         from Terminal triggers the system prompt; grant → capture succeeds.
   - [ ] Deny → `.permissionDenied` message matches DESIGN §8.6 and exit 3 (via
         harness/CLI once Issue 17 lands).
   - [ ] Esc during selection → exit-status/file behavior recorded in the PR table
         (fills KU-5) → mapped to `.cancelled`.
   - [ ] Space (window mode) capture works.
   - [ ] Temp dir contains no `scanmd-*` files after each of: success, cancel, kill -9
         of the parent mid-selection (document any leak the OS makes unavoidable).

## Acceptance Criteria

- [ ] Fake-executor tests cover the full outcome matrix (success/cancel×3 forms/timeout/
      decode-failure) incl. temp cleanup assertions.
- [ ] Subprocess helper rejects relative executables, never touches a shell, and caps
      output buffers (tested with a fake spewing > 64 KB).
- [ ] Manual checklist completed on both macOS versions with the KU-5 table filled.
- [ ] Grep: `/usr/sbin/screencapture` is the only spawned executable in the codebase.

## Validation

`swift test --filter ScreenSourceTests --filter SubprocessTests` + the manual checklist
evidence in the PR.

## Dependencies

Issues 04, 05, 06, 12. Issue 12's `ImageDecoder` is required for decoding the captured
PNG and is a hard merge-time dependency.

## Non-goals

Custom overlay UI (v2, ADR-004), window-id targeted capture flags, app-side invocation
(Issue 20).

## Design References

DESIGN §7.3, §10.1-B2/B3; ADR-004; ISSUE_PLAN KU-5.
