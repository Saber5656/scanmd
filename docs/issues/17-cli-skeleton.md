# Title

CLI skeleton: root command, global flags, config & image commands

## Summary

Build the `scanmd` command surface: root command with global flags, error/exit-code
rendering, stdin handling, `config path|init`, and the first fully wired end-to-end
subcommand (`image`), including the bare-path shorthand.

## Context

DESIGN §8 is the CLI contract. This issue delivers the frame plus one real end-to-end
path (image → Markdown) so review exercises the whole pipeline; the remaining
subcommands land in Issue 18.

## Scope

- `Sources/scanmd/`: `ScanMD.swift` (root), `GlobalOptions.swift`, `ErrorRenderer.swift`,
  `PipelineAssembly.swift`, `Commands/Image.swift`, `Commands/Config.swift`.
- e2e tests for the image path.

## Detailed Requirements

1. Root `ScanMD` (`AsyncParsableCommand`): abstract = one line from README concept;
   subcommands `screen` (stub exiting 2 "not yet implemented" until Issue 18 — hidden
   from help until wired... NO: to keep help stable, register only implemented commands;
   Issue 18 adds the rest), `image`, `config`; default subcommand = `screen` **once
   Issue 18 lands** — until then no default. `--version` prints the SemVer injected via
   a `Version.swift` constant (`0.1.0-dev` here; release stamping in Issue 26).
2. `GlobalOptions` (ParsableArguments, shared via `@OptionGroup`): exactly the DESIGN
   §8.2 table — `--copy`, `--out <path>`, `--no-stdout`, `--format md|txt|json`
   (json errors until Issue 18: `--format json` on image is delivered here since the
   Codable envelope exists in Kit — implement it now, DESIGN §8.3), `--frontmatter/
   --no-frontmatter`, `--lang <codes>` (comma-split, validated via Issue 08's
   `validate(languages:)`), `--fast`, `--quiet`, `--verbose`, `--config <path>`.
   Validation: `--no-stdout` without `--out`/`--copy` → usage error; `--quiet` +
   `--verbose` → usage error.
3. Startup sequence (shared `run` prelude): install `signal(SIGPIPE, SIG_IGN)` → load
   config (Issue 06; print unknown-key warnings to stderr unless `--quiet`) → merge
   flags over config into `OCROptions/RenderOptions/LayoutOptions/sink list` →
   `PipelineAssembly.make(...)` constructs pipeline with real stages.
4. Sink orchestration (DESIGN §8.2): default stdout ON; `--out` adds FileSink
   (existing-dir → directory mode with template; else explicitFile); `--copy` adds
   ClipboardSink; `--no-stdout` removes StdoutSink. Delivery receipts to stderr when
   `--verbose` (`wrote: /abs/path`).
5. Error rendering (`ErrorRenderer`): catches `ScanMDError` + ArgumentParser errors;
   prints `scanmd: error: <userMessage>` (+ remediation lines) to stderr; exits with
   §8.4 codes (ArgumentParser validation → 2). Ctrl-C (SIGINT) → cancel the pipeline
   task → exit 8. `--verbose` adds stage timings from `ScanStats` to stderr.
6. `image <path|->` command: `-` reads stdin fully (cap: `limits.maxFileSizeMB`,
   enforced while reading — stop at limit+1 bytes → `.limitExceeded`); otherwise
   `ImageFileSource(path:)`. Bare-path shorthand at root: if argv[1] exists as a file
   and no known subcommand matches, rewrite to `image <path>` / `pdf <path>` by UTType
   (pdf rewrite activates when Issue 18 lands; until then non-image files → exit 2).
7. `config path` prints the resolved path (honoring `--config`/env); `config init`
   writes `defaultConfigJSON()` (Issue 06), refuses overwrite → exit 7, success prints
   the path to stderr and nothing to stdout.
8. `--help` texts: every flag documented with its default; help output committed as a
   golden test (update-on-purpose only).
9. e2e tests (spawn the built binary via `Process` in tests, or `swift run` wrapper
   script): `image ja-paragraphs.png` → stdout contains ≥ 80% of expected tokens ·
   `--format txt` strips markdown syntax · `--format json` parses, `version == 1`,
   markdown field non-empty, blocks tagged per Issue 05 encoding · `--out <tmpdir>`
   writes templated file (receipt path exists) · `--no-stdout --out` → empty stdout ·
   exit codes: missing file 4, `not-an-image.png` 5, blank image 6, bad flag combo 2 ·
   stdin: `cat fixture.png | scanmd image -` works · SIGPIPE: `scanmd image f.png |
   head -c 1` exits without crash.

## Acceptance Criteria

- [ ] Full §8.2 flag surface parses with documented defaults; help golden committed.
- [ ] All e2e cases above green in CI (fixtures; no TCC needed for the image path).
- [ ] Exit codes verified against the §8.4 table in tests.
- [ ] stdout carries payload only (all diagnostics on stderr) — piped-usage test proves
      byte-clean output.

## Validation

CI e2e job output; manual transcript: `scanmd image Tests/Fixtures/ja-paragraphs.png
--frontmatter --copy --verbose` pasted in PR.

## Dependencies

Issues 04, 05, 06, 08, 09, 10, 11, 12.

## Non-goals

`screen`/`clipboard`/`pdf`/`camera` subcommands, default-subcommand switch, shell
completions (Issue 18); release version stamping (Issue 26).

## Design References

DESIGN §8.1, §8.2, §8.3, §8.4, §8.5, §8.7.
