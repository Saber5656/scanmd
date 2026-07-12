# Prior art: on-device screen/image OCR tools on macOS

- Date: 2026-07-12
- Purpose: position scanmd, confirm the gap it fills, and steal proven UX decisions.
- Method: author knowledge as of early 2026. Feature claims below are directional, not
  audited from current releases; anything marked (verify) should be re-checked if it starts
  driving a design change. None of these claims is load-bearing for v1 correctness.

## Comparison

| Tool | Form | OCR | Output | Markdown structuring | Local-only | OSS |
|---|---|---|---|---|---|---|
| macOS built-in (⇧⌘4 + Live Text in Preview/Photos) | OS feature | Vision | select/copy text manually | none | yes | no |
| TRex (amebalabs) | menu bar app | Vision | plain text → clipboard, automation hooks | none | yes | yes (MIT) (verify current state) |
| TextSniper | menu bar app (commercial) | Vision | plain text → clipboard, TTS | none | yes | no |
| Shottr | menu bar app (freemium) | Vision (verify) | screenshot annotation + OCR text copy | none | yes | no |
| normcap | cross-platform app | Tesseract | plain text → clipboard | none | yes | yes |
| ocrmac / macOCR-style CLIs | CLI | Vision | plain text / JSON to stdout | none | yes | yes |
| Raycast "Capture Text" style extensions | launcher plugin | Vision | plain text | none | yes | mixed |
| scanmd (this project) | **CLI + menu bar app, one core** | Vision | **Markdown** (+txt/json) to stdout/clipboard/file | **headings/lists/paragraph heuristics, front matter** | yes (guaranteed, ADR-003) | yes |

## Takeaways that shaped the design

1. **The gap is the output format, not OCR.** Every mature tool stops at flat plain text.
   None reconstructs Markdown structure or emits front matter for note-taking pipelines.
   scanmd's differentiator = Markdown assembly (DESIGN §8.3, Issue 09/10) + multi-source
   intake (PDF/camera/clipboard/screen) behind one CLI/JSON surface for agent pipelines.
2. **Vision, not Tesseract, for Japanese.** The Tesseract-based tool in the table is the
   weakest on ja text; every serious macOS-only tool uses Vision. Confirms ADR-002.
3. **Menu-bar + hotkey is the proven daily-driver UX** (TRex/TextSniper). Confirms ADR-001.
4. **Selection UI via the system is acceptable.** Several small tools (and many scripts)
   ride `screencapture -i`; custom overlays are polish, not table stakes. Confirms ADR-004.
5. **Plain-text clipboard delivery is the expected default** for hotkey capture; scanmd
   keeps "result lands on clipboard" as the app's always-on behavior (DESIGN §9.5).

## Non-goals confirmed by the survey

- Competing with annotation/screenshot managers (Shottr, CleanShot) — out of scope.
- Translation/TTS features (TextSniper) — out of scope for v1.
