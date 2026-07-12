# ADR-003: v1 is strictly local-only; LLM formatting is a v2 opt-in behind a fixed seam

- Status: Accepted
- Date: 2026-07-12
- Decider: repository owner (explicit answer to design questionnaire)

## Context

Captured content is arbitrarily sensitive (anything visible on screen). Markdown
structuring could be improved by an LLM pass (tables, headings), but that requires sending
captured content to an external API, plus API-key handling in an OSS tool.

## Decision

1. v1 performs **zero network I/O**. No telemetry, no update checks, no cloud OCR, no LLM.
2. Markdown structuring in v1 is **rule-based** (DESIGN §8.3 rules r1–r7, layout heuristics
   in Issue 09).
3. The pipeline includes a `MarkdownFormatter` protocol between render and deliver;
   v1 registers only `PassthroughFormatter`. A future LLM formatter is v2 work: opt-in per
   invocation, key sourced from environment or Keychain (never config plaintext), with its
   own threat-model update.
4. Enforcement: CI fails if networking symbols (`URLSession`, `import Network`, `NWConnection`)
   appear anywhere in product targets (DESIGN §10.1 B6). Introducing any network capability
   requires a new ADR and explicit human approval.

## Consequences

- Privacy story is simple and verifiable; the README can promise "your captures never leave
  this Mac" (v1).
- v1 table reconstruction quality is limited — accepted; bbox data is preserved so v2 can
  improve structure without re-OCR.
- No API-key/secret handling exists anywhere in v1, shrinking the threat model (B6, B7).

## Alternatives considered

- Optional `--llm` in v1: rejected — drags key management, spend, and data-egress threat
  modeling into v1 for a secondary feature.
- LLM-required design: rejected — offline-incapable, recurring cost, and worst-case privacy.
