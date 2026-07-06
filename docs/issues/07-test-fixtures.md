# Title

Test fixture corpus and golden-test harness

## Summary

Build the deterministic fixture corpus (synthetic rendered text images, PDFs, malformed
files), the CoreGraphics fixture generator, token-recall assertion helpers, and the golden
file comparison harness used by all later tests.

## Context

DESIGN §11: correctness lives in deterministic unit/golden tests; OCR tests must assert
token recall (≥ 0.9), never exact strings (KU-3). Sources and e2e tests (Issues 08–18)
consume this corpus, so it lands before them.

## Scope

- `Tests/Fixtures/` corpus + `Tests/Fixtures/README.md` (inventory + regeneration).
- `Scripts/gen-fixtures.swift` (or a test-only target) — regenerates synthetic images.
- `Tests/ScanMDKitTests/Support/` — `TokenRecall.swift`, `Golden.swift`.

## Detailed Requirements

1. Synthetic image generator (CoreGraphics bitmap, deterministic: fixed fonts
   (`Helvetica`, `Hiragino Sans`), fixed sizes, white bg, black text, no antialias
   randomness dependence):
   - `en-paragraphs.png` — 3 paragraphs, 14 pt.
   - `ja-paragraphs.png` — 3 Japanese paragraphs (no latin), 14 pt.
   - `mixed-headings.png` — 28 pt title line, 20 pt subtitle, 14 pt body ×2 paragraphs.
   - `bullets.png` — `•` ×3 and `1. 2. 3.` ×3 lines, plus one indented `•` sub-item.
   - `ja-en-mixed.png` — paragraph mixing CJK and latin tokens (tests r4 joining later).
   - Each `<name>.expected.txt` — newline-separated **expected token list** (the exact
     strings drawn, tokenized: latin split on whitespace; CJK split per character).
2. Real-photo fixtures (small, committed, ≤ 500 KB each, content authored for this repo —
   no third-party text): `photo-receipt.heic`, `photo-whiteboard.jpg` + expected-token
   lists with recall threshold annotations.
3. PDFs: `text-layer.pdf` (2 pages, generated from attributed strings — stable text
   layer; committed alongside `text-layer.expected.md` golden for the full CLI path),
   `scanned.pdf` (2 pages: embedded page-size images of rendered text, no text layer),
   `mixed.pdf` (page 1 text layer, page 2 scanned), `encrypted.pdf` (any password).
4. Malformed/abuse corpus (files, each a few KB): `truncated.png`, `not-an-image.png`
   (text bytes, .png name), `zero-byte.pdf`, `deep-bomb.pdf` (valid header, garbage body),
   `huge-dimensions.png` (valid header declaring > `maxImagePixels`; tiny file).
5. `TokenRecall.assertRecall(image:expectedTokensFile:threshold:)` — runs a provided
   recognizer (parameterized; Issue 08 plugs the real one) and asserts
   `|recognized ∩ expected| / |expected| ≥ threshold` after NFKC casefold normalization.
   Failure message lists missing tokens.
6. `Golden.assert(_ actual: String, file: "name.golden.md")` — compares against
   `Tests/Fixtures/golden/`, with `SCANMD_UPDATE_GOLDEN=1` env to regenerate; diff shown
   on failure.
7. `Tests/Fixtures/README.md`: inventory table (file, purpose, consuming issues,
   regeneration command), rule that no fixture may contain personal or third-party
   copyrighted text.

## Acceptance Criteria

- [ ] All fixtures above committed; generator re-produces the synthetic set byte-stably
      on one machine (documented caveat: cross-machine font rendering may differ — the
      *committed* PNGs are canonical; the generator is for authoring new fixtures).
- [ ] Recall helper + golden helper unit-tested with stub inputs.
- [ ] Malformed corpus files load-fail cleanly when opened with ImageIO/PDFKit in a
      smoke test (no crash — graceful nil/throw path).
- [ ] Corpus total size < 5 MB.

## Validation

`swift test --filter FixtureSmokeTests`; corpus inventory README rendered in PR.

## Dependencies

Issue 01.

## Non-goals

Using the real Vision recognizer (Issue 08 wires it into the recall helper); e2e CLI
golden runs (Issue 18).

## Design References

DESIGN §11 (test strategy, determinism, recall), §10.1-B1 (malformed corpus); ISSUE_PLAN
KU-3.
