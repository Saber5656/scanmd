# Title

CLI: remaining subcommands, JSON format, e2e test suite

## Summary

Wire the remaining subcommands (`screen`, `clipboard`, `pdf`, `camera`), make `screen`
the default subcommand, finish the bare-path shorthand (PDF), add shell completions, and
land the full e2e suite that CI runs from now on.

## Context

Completes the DESIGN §8.1 grammar on top of Issue 17's frame. TCC-gated flows (screen,
camera) get manual checklists; everything else is CI-automatable.

## Scope

- `Sources/scanmd/Commands/`: `Screen.swift`, `Clipboard.swift`, `Pdf.swift`,
  `Camera.swift`; root updates; completions; `Tests/CLITests/` e2e suite growth.

## Detailed Requirements

1. `screen`: wraps `ScreenRegionSource` (Issue 15). Becomes the **default subcommand**
   (`scanmd` alone = `scanmd screen`, DESIGN §8.1). No extra flags beyond globals.
2. `clipboard`: wraps `ClipboardSource` (Issue 13); wire the real PDF delegation factory
   (Issue 13 req 2b) here; multiple-URL warning prints to stderr unless `--quiet`.
3. `pdf <path> [--pages <spec>] [--force-ocr]`: wraps `PDFSource` (Issue 14); `--pages`
   grammar errors render as usage (exit 2) naming the bad token; progress on stderr for
   multi-page OCR (`page 3/12…`, suppressed by `--quiet`, single-line rewrite when TTY).
4. `camera [--device <sel>] [--list-devices] [--no-countdown]`: wraps `CameraSource`
   (Issue 16). `--list-devices` prints an aligned index/name/type table to **stdout**
   (it IS the payload of that mode) and exits 0. Countdown `3…2…1` on stderr via the
   source's tick callback; `--no-countdown` skips. If Issue 16's KU-2 fallback was
   activated, this command prints the documented guidance and exits 3 — the flag surface
   still exists.
5. Bare-path shorthand completes: PDF UTType → `pdf <path>` (image case landed in 17);
   nonexistent path or other type → exit 2 with `unknown input: <basename>` + help hint.
6. Shell completions: `scanmd --generate-completion-script {zsh|bash|fish}` (ArgumentParser
   built-in) documented in help; smoke test asserts zsh script generation is non-empty.
7. e2e suite additions (CI-safe subset): `pdf text-layer.pdf` → stdout golden
   byte-exact (`text-layer.expected.md`) — the fully deterministic path · `pdf mixed.pdf`
   → contains page separator + recall tokens from the scanned page · `--pages` each
   grammar case → correct page subset (assert via golden/tokens) and exit-2 cases ·
   `pdf encrypted.pdf` → exit 5 · `clipboard` with a text-only pasteboard shim → exit 4
   with the exact §7.2 message (use a dedicated pasteboard injected via env var
   `SCANMD_TEST_PASTEBOARD=<name>` — production code reads `.general` unless this
   test-only env var is set; document it in code as test-only, no other behavior) ·
   `camera --list-devices` on CI (no camera) → exit 0 with empty table or devices —
   assert no crash · `scanmd nonexistent.xyz` → exit 2.
8. Manual checklist (paste in PR): `scanmd` (default screen) full flow → Markdown on
   stdout · Esc → exit 8, silent stdout · TCC denied → exit 3 + §8.6 message ·
   `scanmd camera` on real hardware (or fallback message per KU-2 outcome) ·
   `scanmd clipboard` after ⌃⇧⌘4 screenshot-to-clipboard.

## Acceptance Criteria

- [ ] `scanmd --help` shows all five capture subcommands + config; default = screen;
      help golden updated once.
- [ ] Full DESIGN §8.1 grammar accepted; anything else exits 2.
- [ ] CI e2e suite green (deterministic subset); manual checklist evidence attached.
- [ ] `text-layer.pdf` golden is byte-exact (locks front matter off / separator / NFC
      normalization behavior).
- [ ] `SCANMD_TEST_PASTEBOARD` is referenced only in `ClipboardSource` wiring + tests.

## Validation

CI run link + manual transcript for the four TCC/hardware cases.

## Dependencies

Issues 13, 14, 15, 16, 17.

## Non-goals

App surface (Issues 19–23), man page (Issue 28 decides README-level docs; a `.1` roff
page is v2), watch/batch modes (v2).

## Design References

DESIGN §7.2, §8.1–8.5; ISSUE_PLAN KU-2, KU-5.
