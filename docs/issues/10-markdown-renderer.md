# Title

Markdown renderer and front matter builder

## Summary

Implement `DefaultMarkdownRenderer: MarkdownRendering` — blocks → Markdown text per
DESIGN §8.3 rules r1–r7 — including escaping modes, page separators, and the YAML front
matter builder.

## Context

The renderer is the last deterministic stage before delivery; its output is the product.
It must be golden-testable and never depend on OCR (pure function over `ScanDocument`).

## Scope

- `Sources/ScanMDKit/Render/DefaultMarkdownRenderer.swift`, `FrontMatter.swift`,
  `MarkdownEscape.swift`.
- Golden + unit tests.

## Detailed Requirements

1. Rendering rules (normative, DESIGN §8.3):
   - r1 `heading(level:text:)` → `#`×level + space + text, blank line before and after.
   - r2 `paragraph` → text, single blank line between top-level blocks.
   - r3 `listItem(ordered:indent:text:)` → `"  "×indent` + (`- ` | `1. `) + text.
     Consecutive listItems at any indent form one list (no blank lines between); a
     non-list block after a list gets a blank line.
   - r7 `pageBreak` → `options.pageSeparator` verbatim (default `\n---\n`), collapsing
     surrounding blank lines so output never has 3+ consecutive newlines.
   - Document ends with exactly one trailing newline; never leading whitespace.
2. Escaping (`MarkdownEscape.apply(_:mode:)`, DESIGN §8.3-r5):
   - `none`: passthrough.
   - `minimal` (default): escape a line-leading `#`, `>`, `-`, `+`, `*`, `` ` ``, and
     `^\d{1,9}[.)]` (backslash before the token / before the `.`/`)`); applies to
     paragraph text lines only — renderer-generated heading/list markers are never
     escaped.
   - `full`: minimal + escape inline `` * _ [ ] ( ) ~ ` `` everywhere in block text.
   - Escaping applies after layout, so heading/list *classification* sees raw text.
3. Front matter (r6) when `options.frontmatter`:
   ```yaml
   ---
   source: screen            # SourceKind rawValue
   path: /abs/path or null
   captured: 2026-07-07T14:03:22+09:00   # ISO-8601 with local offset
   pages: 2
   languages: [ja-JP, en-US]
   tool: scanmd/1.0.0        # options.toolVersion
   ---
   ```
   Hand-build the YAML (fixed key order above); string values are double-quoted iff they
   contain YAML-special characters (`: # " '` or leading/trailing space); no third-party
   YAML dependency.
4. `txt` format support: `renderPlainText(_:)` — block texts joined by blank lines,
   list items prefixed `- `/`1. ` without indentation escaping, pageBreak → blank line;
   no front matter ever (DESIGN §8.3).
5. Golden tests (`Tests/Fixtures/golden/render/*.golden.md`): one per feature —
   headings+paragraphs, nested mixed lists, escaping in all 3 modes (input containing
   `# fake heading`, `1. fake list`, `*emph*`), front matter on/off, multi-page with
   custom separator, CJK content byte-exactness (no mojibake, NFC preserved).
6. Unit tests: blank-line discipline property (never `\n\n\n` in output), trailing
   newline property, YAML quoting edge cases (`title: with: colon`, leading space,
   Japanese text unquoted).

## Acceptance Criteria

- [ ] All golden files match; goldens reviewed as readable Markdown in the PR.
- [ ] The two output properties (no triple newline, single trailing newline) hold across
      all tests including randomized block-sequence property test (≥ 1000 cases).
- [ ] Front matter output parses under a YAML parser in a test... **without** adding a
      YAML dependency to the product: test-target-only dependency is acceptable ONLY if
      already transitively present; otherwise assert against fixed expected strings.
- [ ] `txt` renderer covered for every Block case.

## Validation

`swift test --filter RenderTests`; attach one rendered sample (mixed-headings fixture via
Issues 08+09) to the PR description.

## Dependencies

Issues 05, 07.

## Non-goals

Layout classification (Issue 09), JSON format (Issue 18 owns the envelope; it reuses
`ScanDocument` Codable from Issue 05), table rendering (v2).

## Design References

DESIGN §8.3 (r1–r7), §8.7 (escape mode config); ADR-003 (rule-based v1).
