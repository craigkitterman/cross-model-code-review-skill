# Changelog

## v1.0.0 — 2026-03-16

Initial public release. Built through multi-model iterative review (Claude + Codex + Gemini).

### Features
- 7 review profiles: quick, standard, deep, security, ux, doc, pre-merge
- 16 review domains: 8 for code, 8 for documents
- Consensus synthesis engine with confidence scoring (HIGH/MEDIUM/REVIEW)
- Model-agnostic design: works with Codex, Gemini, Aider, Ollama, or any CLI
- Progressive disclosure: SKILL.md under 300 lines, references load on-demand
- Structured output format for cross-model comparison
- Re-review iteration loop (max 3 passes)
- Graceful degradation (works with 1 model, better with 2+)

### Built-in Model Support
- OpenAI Codex CLI (`codex exec --full-auto`)
- Google Gemini CLI (`gemini -p`)
- Community examples: Aider, Ollama, sgpt, custom scripts

### Review Process
- Phase 1: Scope & context gathering (auto-detect changes, read CLAUDE.md)
- Phase 2: Parallel external model reviews
- Phase 3: Claude self-review with full codebase context
- Phase 4: Consensus synthesis with cross-referencing
- Phase 5: Structured report generation
- Phase 6: Fix & re-review iteration loop
- Phase 7: Final certification

### Development Notes
- Self-reviewed using its own multi-model workflow
- Gemini iteration 1: 8.5/10 → fixed ARG_MAX, CLI syntax, file types
- Codex iteration 1: 5/10 → fixed skill name, verdict enums, domain taxonomy,
  vaporware profiles, flag syntax, zero-division, hardcoded branch
- Claude iteration 1: 8/10 → fixed context gathering, sandbox docs
- All Critical and Important findings resolved across 2 review iterations
