# Title

Config schema v1: load, validate, defaults, init

## Summary

Implement `ScanMDConfig` (the DESIGN §8.7 JSON schema), path resolution, strict
validation with JSON-path errors, atomic save, hard caps on limits, and the pure logic
behind `scanmd config init/path`.

## Context

CLI and app share one config file (ADR-006). Everything configurable in the product flows
through this module; downstream issues consume typed sub-structs (`OCROptions`, `Limits`,
output rules).

## Scope

- `Sources/ScanMDKit/Config/ConfigSchema.swift`, `ConfigLoader.swift`,
  `FilenameTemplate.swift`.
- Unit tests incl. malformed-config corpus.

## Detailed Requirements

1. `ScanMDConfig: Codable, Sendable` mirroring DESIGN §8.7 exactly (sections `ocr`,
   `markdown`, `output`, `pdf`, `camera`, `limits`, `app`; all fields optional in JSON,
   defaults applied field-wise so a partial file works).
2. Path resolution (pure function, tested):
   explicit path → `$SCANMD_CONFIG` → `$XDG_CONFIG_HOME/scanmd/config.json` →
   `~/.config/scanmd/config.json`.
3. `ConfigLoader.load(path:)`:
   - missing file → full defaults (not an error);
   - unreadable/undecodable JSON → `ScanMDError.usage("config: <json-path>: <problem>")`;
   - unknown keys → collected into `warnings: [String]` returned alongside the config
     (CLI prints them to stderr; app logs them). Implement via decoding into
     `[String: JSONValue]` shadow pass or `userInfo`-based key tracking — any approach
     that reports the full JSON path (e.g. `output.colision`) works.
   - enum fields validate values (`recognitionLevel`, `escapeSpecials`, `collision`) with
     the offending value named in the error.
4. Hard caps (DESIGN §10.4): after defaults+file merge, each `limits.*` and `pdf.maxPages`
   `pdf.rasterDPI` value may not exceed 4× its default → `.usage` naming the key.
   `rasterDPI` additionally clamps to ≤ 600.
5. `ConfigLoader.save(_:to:)`: create parent dir 0700 if needed, write to
   `config.json.tmp-<uuid>` then `rename(2)`; final file chmod 0600; pretty-printed,
   `sortedKeys` for stable diffs.
6. `defaultConfigJSON()` — canonical full-default document used by `config init`
   (Issue 17 wires the subcommand): byte-stable output, includes `"version": 1`.
7. `FilenameTemplate.render(template:date:sourceKind:slugSource:) -> String`
   implementing exactly the variables of DESIGN §8.7 (`{yyyy}{MM}{dd}{HH}{mm}{ss}{HHmmss}
   {source}{slug}`); `{slug}` sanitization: NFKC normalize, keep `[A-Za-z0-9぀-ヿ一-鿿-]`,
   others → `-`, collapse repeats, trim `-`, max 24 chars, empty → `untitled`. Unknown
   `{token}` → `.usage("output.filenameTemplate: unknown variable {token}")`.
   The rendered name must never contain `/`, `\0`, or begin with `.` or `-` (test).
8. Config contains no secrets; add a doc comment stating v1 must keep it that way
   (ADR-003/ADR-006).

## Acceptance Criteria

- [ ] Partial config files merge over defaults field-wise (tested per section).
- [ ] Unknown key → warning with full JSON path; wrong type → `.usage` with path.
- [ ] Limit caps enforced (test 5× default rejected, 4× accepted; DPI clamp).
- [ ] Template renderer passes the variable, sanitization, and injection tests above.
- [ ] Save is atomic (tmp+rename) and 0600/0700 (asserted via `FileManager` attributes).
- [ ] Malformed corpus (truncated JSON, wrong types, huge numbers) all yield typed errors,
      never crashes.

## Validation

`swift test --filter ConfigTests --filter TemplateTests`; include a test writing then
reloading the default config byte-identically.

## Dependencies

Issues 01, 04.

## Non-goals

CLI `config` subcommand wiring (Issue 17), app live-reload file watching (Issue 22),
hotkey storage (KeyboardShortcuts owns it, ADR-006).

## Design References

DESIGN §8.7, §10.1-B7, §10.4; ADR-006.
