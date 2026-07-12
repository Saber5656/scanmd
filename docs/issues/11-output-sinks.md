# Title

Output sinks: stdout, clipboard, file (templates)

## Summary

Implement the three `OutputSink`s — `StdoutSink`, `ClipboardSink`, `FileSink` — with
template-based naming, collision handling, atomic 0600 writes, and path-safety guards.

## Context

Sinks are trust boundary B4 (DESIGN §10.1): the only place scanmd writes user files.
Both surfaces share them (CLI flags §8.2; app auto-save §9.5).

## Scope

- `Sources/ScanMDKit/Sinks/StdoutSink.swift`, `ClipboardSink.swift`, `FileSink.swift`.
- Unit tests in a temp directory sandbox.

## Detailed Requirements

1. `StdoutSink`: writes the payload to `FileHandle.standardOutput` exactly (no added
   newline beyond the renderer's trailing one); receipt `location: nil`. Write failures
   (closed pipe → EPIPE) map to `.deliveryFailed("stdout")` — callers treat SIGPIPE:
   the CLI must install `signal(SIGPIPE, SIG_IGN)` (wired in Issue 17; note here for the
   error path test).
2. `ClipboardSink`: `NSPasteboard.general.clearContents()` then
   `setString(markdown, forType: .string)`; false return → `.deliveryFailed("clipboard")`.
   Never reads the pasteboard (B5).
3. `FileSink(target: FileTarget)` where
   `enum FileTarget { case explicitFile(String), directory(String, template: String) }`:
   - Path normalization: expand `~`, resolve to absolute, standardize `..` components, and
     resolve existing symlinks in the final parent directory **before** any filesystem
     operation. The containment check compares symlink-resolved parent paths.
   - `explicitFile`: parent directory must exist (`.deliveryFailed` if not — do NOT
     mkdir for explicit paths); write the file.
   - `directory`: create the directory (and intermediates) 0700 if missing, and reject a
     pre-existing target directory whose permissions are broader than 0700; render
     filename via `FilenameTemplate` (Issue 06) with `slugSource` = first heading text
     else first paragraph's first 24 chars else `untitled`; **containment check**: the final resolved
     path's parent must equal the resolved target directory (defense-in-depth on top of
     template sanitization; violation → `.deliveryFailed("path escapes target
     directory")`).
   - Collision policy (config `output.collision`): `suffix` (default) → try `name.md`,
     `name-1.md` … `name-99.md`, then `.deliveryFailed`; `overwrite` → replace;
     `fail` → `.deliveryFailed("exists")`.
   - Atomic write: temp file in the same directory (`.<name>.tmp-<uuid>`) + `rename`;
     chmod 0600 before rename. With `overwrite` + existing symlink at destination:
      `rename` replaces the symlink itself (never follows it) — test this.
   - Receipt returns the final absolute path (used by `--verbose` and app notifications).
4. All sinks `Sendable`; no logging of content (lengths only, Issue 04 wrapper).
5. Tests (temp dir): template naming end-to-end · each collision mode · suffix exhaustion
   at 99 · containment check triggers on a hostile template result (simulate by injecting
   a template rendering `../esc`) · atomic overwrite of symlink replaces link not target ·
   permissions 0600/0700 asserted · missing parent for explicitFile fails · unwritable
   dir (chmod 0500) fails with `.deliveryFailed`.

## Acceptance Criteria

- [ ] All tests above green; no test touches the real pasteboard except one guarded
      integration test (skipped on CI if pasteboard unavailable).
- [ ] Grep confirms sinks never call `Log` with content strings.
- [ ] Receipt paths are absolute and reflect suffixed names.

## Validation

`swift test --filter SinkTests`. Manual: `swift run scanmd --version` unaffected;
pasteboard test run locally once, output pasted in PR.

## Dependencies

Issues 05, 06.

## Non-goals

CLI flag plumbing (`--out`, `--copy`, `--no-stdout` — Issue 17), app notification of
receipts (Issue 23).

## Design References

DESIGN §5.5, §8.2, §8.7 (template/collision), §10.1-B4/B5.
