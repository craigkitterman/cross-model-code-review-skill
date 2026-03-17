---
name: review
description: |
  Installed as `/review`. Also triggers on natural language: 'review this code', 'code review', etc.
  Multi-model consensus-driven review for code, documents, UX, and security.
  Orchestrates parallel reviews from any combination of AI CLIs (Codex, Gemini, Aider, Ollama, etc.)
  with structured output, cross-model synthesis, and iterative improvement loops.
  Supports 8 review domains with configurable profiles for different review depths.
  Use when: (1) Reviewing code changes or PRs, (2) Security auditing, (3) UX/accessibility review,
  (4) Document/spec review, (5) Pre-merge quality gates, (6) Architecture review.
  Triggers: "review", "code review", "multi-ai review", "consensus review", "/review",
  "/code-review", "/doc-review", "/security-review", "/ux-review", "review this",
  "review changes", "multi-model review", "ai review"
---

# Multi-AI Review

Consensus-driven quality verification using multiple AI models. Claude orchestrates external model CLIs,
each performing independent reviews. Claude then synthesizes findings into a confidence-scored consensus
report with cross-references and actionable recommendations.

**Why multi-model?** Different models have different blind spots. Redundant independent review catches
issues any single model would miss. Consensus findings (found by 2+ models) are high-confidence signals.

## Quick Start

Natural language works best. All of these trigger the skill:

| What You Say | What Happens |
|------------|--------|
| `/review` | Review uncommitted changes (standard profile) |
| `/review security focus` | Security-focused deep review |
| `/review ux focus` | UX and accessibility review |
| `/review quick` | Fast single-model review |
| `/review deep` | All domains, all models, multiple passes |
| `review the spec at specs/029/plan.md` | Document/spec review |
| `review src/auth/login.ts` | Review a specific file |
| `review this code for security issues` | Natural language trigger |

Claude interprets your intent from natural language — no special flag syntax needed.
If the intent is ambiguous, Claude will ask which profile and scope you want.

## Prerequisites

At least one external model CLI is recommended for multi-model consensus. Without external CLIs, Claude performs a solo review with reduced confidence.

```bash
# Verify available models (run all, use what's available):
codex --version
gemini --version
```

Works without external CLIs (solo mode with reduced confidence), but multi-model consensus requires at least one.
See `config.md` for adding models and full profile definitions.

## Review Profiles

| Profile | Domains Reviewed | Min Models | Passes | Best For |
|---------|-----------------|------------|--------|----------|
| `quick` | Bugs & Logic, Maintainability | 1 + Claude | 1 | Small changes, typo fixes |
| `standard` | All 8 code domains | 2 + Claude | 1-2 | Default for most work |
| `deep` | All 8 code domains | All available | 2-3 | Critical features, releases |
| `security` | Security, Robustness, Bugs & Logic | All available | 2 | Auth, APIs, data handling |
| `ux` | UX Impact, Performance, Architecture | All available | 1-2 | UI components, user flows |
| `doc` | 8 document domains (different set) | 2 + Claude | 1-2 | Specs, plans, PRDs |
| `pre-merge` | All 8 code domains | All available | 1 | Final gate before merge |

**Code domains** (profiles: quick, standard, deep, security, ux, pre-merge):
Security, Bugs & Logic, Robustness, Performance, Architecture, Scalability, UX Impact, Maintainability

**Document domains** (profile: doc):
Completeness, Technical Accuracy, Consistency, Clarity, User Experience, Simplicity, Product Thinking, Polish

See `references/domains.md` for detailed criteria per domain.

---

## Workflow Phases

### Phase 1: Scope & Context

**1a. Determine what to review:**

```bash
# Auto-detect: staged > unstaged > recent commits
git diff --cached --stat                    # Staged changes
git diff --stat                             # Unstaged changes
DEFAULT_BRANCH=$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null | sed 's@origin/@@' || echo main)
git log --oneline $DEFAULT_BRANCH..HEAD 2>/dev/null    # Commits not on default branch
```

If the user specified files, a PR number, or a document path, use that instead.

**1b. Build the reviewable file list:**
- Include: source code (`.ts`, `.tsx`, `.js`, `.jsx`, `.py`, `.go`, `.rs`, `.java`, `.rb`, `.sql`,
  `.c`, `.cpp`, `.h`, `.cs`, `.php`, `.swift`, `.kt`, `.vue`, `.svelte`)
- Include for doc profile: `.md`, `.mdx`, `.txt`, `.rst`
- Exclude: lockfiles, `node_modules/`, `dist/`, generated files, binary files
- For large diffs (>3000 lines): split into logical chunks and review sequentially

**1c. Gather project context:**
- Read `CLAUDE.md` / `AGENTS.md` / `.cursorrules` / `.windsurfrules` for project-specific standards
- Read `.eslintrc` / `tsconfig.json` for configured linting rules
- Note the active profile and which domains are enabled

**1d. Confirm scope** (if >20 files or ambiguous):
> "Found {N} changed files. Reviewing with **{profile}** profile using {models}. Proceed?"

### Phase 2: Parallel External Reviews

Run each external model review simultaneously using `run_in_background: true` on the Bash tool.

For each model, construct a review prompt from `references/prompts.md` containing:
1. The diff or file contents (inline — external models may lack workspace access)
2. The active domain criteria from `references/domains.md`
3. The structured output format specification
4. Project-specific context (from CLAUDE.md, linting rules, etc.)

**Invocation commands:**

```bash
# Codex CLI (non-interactive, reads workspace files in its sandbox)
codex exec --full-auto "PROMPT_HERE"

# Codex built-in review (alternative — auto-generates review of current repo)
codex exec review

# Gemini CLI (headless/non-interactive)
gemini -p "PROMPT_HERE"
```

**For large prompts** (>32KB — hits OS ARG_MAX limits, especially on Windows):
Write the prompt to a temp file and pipe via stdin to avoid shell argument limits:
```bash
# Write prompt to temp file
cat > /tmp/review-prompt.txt << 'REVIEW_EOF'
{COMPLETE_PROMPT}
REVIEW_EOF

# Codex via stdin (avoids ARG_MAX)
cat /tmp/review-prompt.txt | codex exec --full-auto -

# Gemini via stdin
cat /tmp/review-prompt.txt | gemini -p -

# Clean up
rm /tmp/review-prompt.txt
```

**Sandbox note:** Codex CLI runs in a sandbox scoped to the current git repository.
It CANNOT read files outside the workspace root. If the files to review are outside
the repo (e.g., `~/.claude/skills/`), you must either:
1. Copy the files into the workspace temporarily, or
2. Include the full file contents inline in the prompt text

**Timeout:** 5 minutes per model. If a model times out, log it and continue with remaining models.

### Phase 3: Claude Self-Review

While external models run, Claude performs its own independent review:

1. Read every changed file **completely** (not just the diff — full file context matters)
2. Review against all active domains using criteria from `references/domains.md`
3. Leverage advantages external models lack:
   - Full codebase context (imports, callers, related files)
   - Project-specific patterns from CLAUDE.md and project rules
   - Knowledge of recent changes and architectural decisions
4. Document findings in the same structured format as external models

### Phase 4: Consensus Synthesis

After all reviews complete, synthesize using the algorithm from `references/synthesis.md`:

**4a. Parse and normalize findings:**
- Extract findings from each model's output
- Normalize to common format: `{severity, domain, file, line, title, description, fix}`

**4b. Cross-reference findings:**
- Match findings by file + line range (within 5 lines = same finding)
- Match findings by semantic similarity (same issue, different wording)

**4c. Assign confidence levels:**

| Found By | Confidence | Action |
|----------|------------|--------|
| 2+ models | **HIGH** | Address immediately — validated by independent reviewers |
| 1 model only | **MEDIUM** | Likely valid, verify context before acting |
| Models conflict | **REVIEW** | Document both perspectives, Claude arbitrates |

**4d. Severity escalation:** If ANY model rates a finding as Critical, it stays Critical.

**4e. Calculate domain scores:** Average scores across models, flag outliers (>2 point spread).

### Phase 5: Generate Consensus Report

Output a structured report (template in `references/synthesis.md`):

```markdown
# Multi-AI Review Report

**Profile:** {profile} | **Models:** {model_list} | **Files:** {count} | **Date:** {date}

## Verdict: {APPROVED | CHANGES_REQUESTED | BLOCKED}
## Overall Score: {X}/10

## Executive Summary
{2-3 sentences: overall quality, key concerns, standout strengths}

## Consensus Matrix
| Domain | Claude | {Model2} | {Model3} | Consensus |
|--------|--------|----------|----------|-----------|
| Security | 9/10 | 8/10 | 9/10 | 8.7 |
| ... | | | | |

## Findings

### Critical — HIGH Confidence (multi-model consensus)
{findings agreed upon by 2+ models}

### Critical — MEDIUM Confidence (single model)
{critical findings from one model, verified by Claude}

### Important
{important findings, grouped by domain}

### Suggestions
{nice-to-have improvements}

## Model Agreement
- Unanimous: {X} findings
- Majority: {Y} findings
- Single-model: {Z} findings
- Conflicts resolved: {W}
```

### Phase 6: Fix & Re-review Loop

If verdict is not APPROVED:

1. Present findings to user with prioritized fix recommendations
2. After user approves, implement fixes starting with Critical findings
3. Re-run external reviews on **changed files only** (incremental)
4. Synthesize new findings, compare with previous iteration
5. Track: fixes confirmed, new issues introduced, remaining issues
6. Repeat until consensus APPROVED or user accepts current state

**Max iterations:** 3 (configurable). After 3, present final state and let user decide.

### Phase 7: Final Certification

When all models approve:

```markdown
## Multi-AI Quality Certification

APPROVED by consensus ({N} models)

| Model | Verdict | Score |
|-------|---------|-------|
| Claude | APPROVED | 9/10 |
| {Model2} | APPROVED | 8/10 |

All {profile} domains verified. Review confidence: HIGH.
Iterations: {N}. Total findings resolved: {X}.
```

---

## Error Handling

| Scenario | Action |
|----------|--------|
| CLI not installed | Skip model, warn user, continue with available |
| Model times out (>5 min) | Log timeout, continue with remaining models |
| Unparseable output | Extract what's usable, note reduced confidence for that model |
| All external models fail | Claude solo review, mark overall confidence REDUCED |
| No changes detected | Inform user, offer to review specific files |
| Diff too large (>5000 lines) | Split into chunks, review sequentially, merge findings |
| Model returns only praise | Flag as potentially shallow review, weight findings lower |
| Solo review (no external models) | Simplify report: single-column matrix, note REDUCED confidence, skip cross-referencing |

## Scope Guidance: Diff vs Pre-existing Code

When reviewing a small diff within a large file, focus findings on **the changed code**.
Pre-existing issues in surrounding code may be noted as "out of scope" suggestions, but
should NOT be counted in the verdict or domain scores. The review evaluates the change,
not the entire file history.

## Reference Files

Load these on-demand based on the current phase:

| File | Load During | Contains |
|------|-------------|----------|
| `config.md` | Setup / customization | Model registry, profile definitions, adding new models |
| `references/domains.md` | Phase 2-3 (reviewing) | Detailed criteria for all 16 review domains (8 code + 8 doc) |
| `references/synthesis.md` | Phase 4-5 (synthesizing) | Consensus algorithm, conflict resolution, report templates |
| `references/prompts.md` | Phase 2 (prompting models) | Exact prompt templates for code review, doc review, re-review |

## Adding New Models

Any CLI that accepts a text prompt and returns text output works. Add to `config.md`:
1. Model name and verify command
2. Invocation pattern (how to pass prompt)
3. Whether it reads the local workspace
4. Timeout preference

See `config.md` for examples including Aider, Ollama, sgpt, and custom scripts.

## Principles

1. **Redundancy over division** — Every model reviews everything. No splitting responsibilities.
2. **Confidence from consensus** — Multi-model agreement is the strongest quality signal.
3. **Structured output** — Same format from every model enables automated synthesis.
4. **Progressive depth** — Quick profile for small changes, deep profile for critical code.
5. **Project-aware** — Reads CLAUDE.md and project rules, not just generic best practices.
6. **Graceful degradation** — Works with 1 model, better with 2, best with 3+.
7. **Iterative improvement** — Fix-and-re-review loop continues until consensus approval.
