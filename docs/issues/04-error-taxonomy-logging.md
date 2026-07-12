# Title

Error taxonomy, exit codes, and privacy-safe logging

## Summary

Implement `ScanMDError`, the error→exit-code mapping, fixed remediation messages, a
privacy-safe `Logger` wrapper, and the subprocess-free support utilities every later
module throws through.

## Context

DESIGN §8.4 defines a normative exit-code table; §8.6 fixes remediation strings; §10.2-p1
forbids captured content in logs. Centralizing this first means every subsequent module
just throws typed errors.

## Scope

- `Sources/ScanMDKit/Support/ScanMDError.swift`, `ExitCode.swift`, `Log.swift`.
- Unit tests for mapping and message rendering.

## Detailed Requirements

1. `public enum PermissionScope: String, Sendable { case screen, camera }`.
2. `public enum ScanMDError: Error, Sendable, Equatable`:
   `usage(String)`, `permissionDenied(PermissionScope)`, `noInput(String)`,
   `unsupportedInput(String)`, `emptyResult`, `deliveryFailed(String)`, `cancelled`,
   `timeout(String)`, `limitExceeded(String)`, `internalError(String)`.
3. `public extension ScanMDError { var exitCode: Int32 }` — exactly the DESIGN §8.4 table:
   usage=2, permissionDenied=3, noInput=4, unsupportedInput=5, emptyResult=6,
   deliveryFailed=7, cancelled=8, timeout=9, limitExceeded=10, internalError=1.
4. `var userMessage: String` — one line, no trailing period style consistent, English.
   `permissionDenied` appends the fixed remediation text from DESIGN §8.6 verbatim
   (screen and camera variants) as additional lines.
5. `var remediationURL: URL?` — `x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture`
   for screen, `…?Privacy_Camera` for camera, else nil (used by the app in Issue 23).
6. `Log.swift`: thin wrapper over `os.Logger` with subsystem `dev.saber5656.scanmd`,
   categories `pipeline|source|render|sink|config|app`. Public API only accepts
   non-content values: `Log.pipeline.event("ocr.done", count: Int, ms: Int)` style
   helpers. **No API accepts recognized text or image data** — make misuse impossible by
   the wrapper's signatures, not by convention. Interpolated strings use `privacy: .public`
   only for enum/int values.
7. `ScanMDError` must be the ONLY error type crossing module boundaries: add
   `func mapToScanMDError(_ error: Error, context: String) -> ScanMDError` that wraps
   foreign errors into `.internalError("context: <type>")` without embedding potentially
   sensitive descriptions (type name + code only, never full descriptions of file paths
   beyond basename).
8. Unit tests: every case maps to the documented exit code; remediation strings match
   DESIGN §8.6 byte-for-byte; `mapToScanMDError` never includes a full path (test with a
   fake `NSError` containing `/Users/secret/file.png` — output must contain at most
   `file.png`).

## Acceptance Criteria

- [ ] All ten cases exist with the exact exit codes above; exhaustive `switch` (no default).
- [ ] Remediation text matches DESIGN §8.6 exactly (tested).
- [ ] Log wrapper exposes no API capable of logging content (compile-level guarantee).
- [ ] `swift test` green; new tests cover 100% of `ScanMDError` cases.

## Validation

`swift test --filter ErrorTests` output pasted into PR; grep shows no `%{public}` applied
to string interpolations of user data (reviewer checklist item).

## Dependencies

Issue 01.

## Non-goals

CLI printing/formatting of errors (Issue 17), notification surfacing (Issue 23),
subprocess helper (Issue 15 creates `Support/Subprocess.swift`).

## Design References

DESIGN §8.4, §8.5, §8.6, §10.2-p1.
