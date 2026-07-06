# Title

Layout reconstruction: lines → blocks heuristics

## Summary

Implement `HeuristicLayoutReconstructor: LayoutReconstructing` — the pure function that
turns positioned `RecognizedLine`s into `Block`s (paragraphs, headings, list items) using
the normative thresholds below.

## Context

This module is scanmd's differentiator versus plain-text OCR tools
(research/prior-art.md takeaway 1). It is deliberately a pure, deterministic function so
its behavior is fully unit-testable without OCR (DESIGN §11). Thresholds are constants in
one file because they may be retuned (KU-8).

## Scope

- `Sources/ScanMDKit/Layout/HeuristicLayoutReconstructor.swift`, `LayoutTuning.swift`.
- Exhaustive unit tests with hand-built `RecognizedLine` arrays (no images involved).

## Detailed Requirements

All constants live in `struct LayoutTuning` with these defaults (single source of truth):
`paragraphGapFactor = 0.6`, `hardBreakFactor = 1.8`, `headingHeightFactor = 1.35`,
`h1Factor = 1.9`, `h2Factor = 1.6`, `headingMaxTokens = 12`, `indentFactor = 1.5`,
`maxIndentLevels = 3`.

Algorithm (normative, implement in this order):

1. **Empty input** → `[]`.
2. **Median line height** `h_med` = median of `bbox.height` (population median; single
   line → its own height).
3. **Sort** lines by `(bbox.y ascending, bbox.x ascending)` (top-left origin per §5.1).
4. **Group into visual runs**: iterate sorted lines; gap `g = y_next − (y_prev +
   h_prev)`. `g > paragraphGapFactor × h_med` starts a new run. (Negative g — overlapping
   or same-row lines — never starts a run; same-row fragments were already x-sorted.)
5. **Classify each run**:
   - *List item* (if `options.listDetection`): first line's text matches
     unordered regex `^[•・●○◦▪▸*–—-]\s+` or ordered regex
     `^(\(?\d{1,3}[.)]|\(\d{1,3}\)|[①-⑳])\s*`. Strip the marker from the text. Indent
     level = `min(maxIndentLevels, floor((x_line − x_min_of_all_lines) / (indentFactor ×
     h_med)))` where `x` is the line's `bbox.x`.
   - *Heading* (if `options.headingDetection`, checked only for single-line runs that are
     not list items): line height `> headingHeightFactor × h_med` AND token count
     ≤ `headingMaxTokens` (tokens: whitespace-split for latin; CJK counts 2 chars = 1
     token, rounded up). Level: height ≥ `h1Factor × h_med` → 1; ≥ `h2Factor` → 2;
     else 3.
   - *Paragraph* otherwise.
6. **Join lines inside a paragraph run** (DESIGN §8.3-r4): no space when both boundary
   chars are CJK (Han/Hiragana/Katakana/full-width punct, i.e. Unicode blocks CJK Unified
   Ideographs, Hiragana, Katakana, Halfwidth and Fullwidth Forms, CJK Symbols and
   Punctuation); latin line ending in `-` → drop hyphen, join with no space; else join
   with single space.
7. **Consecutive list items** at the same indent merge into one visual list (renderer
   concern is Issue 10; here each line stays its own `listItem` block).
8. Whitespace: text is trimmed per line before classification; lines that trim to empty
   are dropped before step 2.

Testing matrix (minimum; hand-built inputs, exact expected `[Block]`):
two-paragraph split · hard-break vs paragraph gap · heading H1/H2/H3 by height · heading
rejected by token count · bullet each marker variant · ordered each variant incl `①` ·
nested indent 0/1/2 and clamp at 3 · CJK join / latin space join / hyphen join ·
same-row x-sorted fragments · single line · empty input · listDetection=false ·
headingDetection=false.

## Acceptance Criteria

- [ ] All matrix cases pass with exact expected block arrays.
- [ ] Function is pure (`static`/value semantics), no Foundation date/locale dependence,
      deterministic across runs.
- [ ] All thresholds referenced only from `LayoutTuning` (grep: no magic numbers inline).
- [ ] End-to-end sanity: recognizer (Issue 08) + this module on `mixed-headings.png`
      yields ≥ 1 heading block and ≥ 2 paragraphs; on `bullets.png` yields ≥ 5 listItems
      including ≥ 1 with indent ≥ 1 (recall-style tolerance, not exact).
- [ ] Property test: output block count ≤ input line count; every input's text content
      (ignoring markers/whitespace) appears in some block.

## Validation

`swift test --filter LayoutTests`; PR includes the printed block structure for the two
fixture sanity runs.

## Dependencies

Issues 05, 07 (fixtures for the sanity checks; core tests need none).

## Non-goals

Tables and multi-column reading order (v2, DESIGN §13); markdown escaping/rendering
(Issue 10).

## Design References

DESIGN §8.3-r4 (join rules), §11; ISSUE_PLAN KU-8; research/prior-art.md (gap analysis).
