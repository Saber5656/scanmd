# ADR-006: One JSON config file at `~/.config/scanmd/config.json`, shared by CLI and app

- Status: Accepted
- Date: 2026-07-12
- Decider: Fable (design), conservative default

## Context

CLI and app must share settings (OCR languages, output rules, limits). Candidate formats:
TOML (CLI-idiomatic, needs a third-party parser), JSON (Foundation-native, no comments),
plist (native, hand-editing hostile). Candidate locations: `~/.config` (XDG, CLI-idiomatic)
vs `~/Library/Application Support` (Apple convention) vs `UserDefaults` (app-idiomatic,
opaque to CLI users).

## Decision

- Format: **JSON** with an explicit `"version": 1` key, decoded via Foundation
  `JSONDecoder` — zero added dependencies, exact schema in DESIGN §8.7.
- Location: `--config` flag → `$SCANMD_CONFIG` → `$XDG_CONFIG_HOME/scanmd/config.json` →
  `~/.config/scanmd/config.json`. The app reads/writes the same file (atomic
  same-directory temp-file+rename, 0600; parent dir 0700) and watches the containing
  directory for external changes so temp+rename replacement does not detach the watcher.
- Unknown keys warn (forward compatibility); type errors fail with a JSON-path message.
- Exception: the global hotkey binding is stored by the `KeyboardShortcuts` library in its
  own `UserDefaults` store (its native mechanism); the config file's `app.hotkey` is
  reserved/null in v1.
- The config file must never contain secrets in v1 (no keys exist; see ADR-003).

## Consequences

- One file to document, back up, and sync; CLI users get a predictable dotfile path.
- No comments in JSON — mitigated by `scanmd config init` emitting the full commented
  default via docs (README) and self-describing key names.
- A v2 move to TOML (if ever) would key off the `version` field.

## Alternatives considered

- TOML: nicer to hand-edit but adds a third-party parser to the supply chain for marginal
  gain; rejected for v1.
- `UserDefaults` only: invisible to CLI users and hard to document/diff; rejected.
- Split configs per surface: guaranteed drift; rejected.
