# Multi-AI Review

**Consensus-driven code and document review using multiple AI models.**

A Claude Code skill that orchestrates parallel reviews from any combination of AI CLIs (Codex, Gemini, Aider, Ollama, etc.), synthesizes findings with confidence scoring, and iterates until consensus approval.

## Version

v1.0.0

## Why Multi-Model Review?

Every AI model has blind spots. A single model might miss a security vulnerability that another catches instantly. Multi-AI Review solves this by:

- **Redundant coverage** — Every model reviews every domain independently
- **Consensus scoring** — Findings agreed upon by 2+ models are high-confidence signals
- **Conflict resolution** — When models disagree, Claude arbitrates with full codebase context
- **Iterative improvement** — Fix-and-re-review loops continue until consensus approval

## What It Reviews

| Review Type | Domains | How to Trigger |
|-------------|---------|----------------|
| **Code Review** | Security, Bugs, Robustness, Performance, Architecture, Scalability, UX, Maintainability | `review this code` or `/review` |
| **Security Audit** | Deep security analysis with OWASP checklist | `review with a security focus` |
| **UX Review** | Accessibility (WCAG 2.1 AA), responsive design, component compliance | `review for ux and accessibility` |
| **Document Review** | Completeness, accuracy, clarity, consistency, product thinking | `review the spec at specs/029/plan.md` |
| **Pre-Merge Gate** | All code domains + regression checks | `review as a pre-merge gate` |

## Quick Start

### 1. Install the Skill

Copy the `multi-ai-review` folder to your Claude Code skills directory:

```bash
# Option 1: Download and copy
# Download the multi-ai-review folder and copy it to your Claude Code skills directory:
cp -r multi-ai-review ~/.claude/skills/

# Option 2: Clone from GitHub
# git clone <repo-url>
# cp -r multi-ai-review ~/.claude/skills/
```

The skill registers as `/review`. The folder name stays `multi-ai-review`.

File structure:
```
~/.claude/skills/multi-ai-review/
  SKILL.md
  config.md
  references/
    domains.md
    synthesis.md
    prompts.md
```

### 2. Verify External CLIs

At least one external AI CLI is needed. Claude always participates as a reviewer.

```bash
# Check what's available
codex --version    # OpenAI Codex CLI
gemini --version   # Google Gemini CLI
```

### 3. Run a Review

Natural language works best. All of these trigger the skill:

```
/review                                          — Review uncommitted changes (standard profile)
review this code                                 — Same as above, natural language
review src/auth/login.ts for security issues     — Specific file, security focus
review the spec at docs/plan.md                  — Document review
do a deep review of the authentication changes   — Deep profile, scoped to auth
run a quick review                               — Fast single-model pass
```

Claude interprets your intent from natural language — no special syntax needed. If the intent is ambiguous, Claude will ask which profile and scope you want.

## Review Profiles

| Profile | Speed | Coverage | Best For |
|---------|-------|----------|----------|
| `quick` | Fast | 2 domains, 1 model | Small changes |
| `standard` | Medium | All 8 domains, 2 models | Default for most work |
| `deep` | Slow | All domains, all models, 2-3 passes | Critical features |
| `security` | Medium | Security-focused, all models | Auth, APIs, data handling |
| `ux` | Medium | UX + accessibility, all models | UI components |
| `doc` | Medium | 8 doc domains, 2 models | Specs, plans, PRDs |
| `pre-merge` | Medium | All domains + regression checks | Final merge gate |

## How It Works

```
Phase 1: Scope     → Detect changes, build file list, gather context
Phase 2: Review    → Run external models IN PARALLEL (Codex, Gemini, etc.)
Phase 3: Self-Review → Claude reviews independently with full codebase context
Phase 4: Synthesize → Cross-reference findings, assign confidence levels
Phase 5: Report    → Generate consensus report with matrix and scores
Phase 6: Iterate   → Fix issues, re-review, repeat until consensus
Phase 7: Certify   → Final certification when all models approve
```

### Consensus Matrix Example

```
| Domain         | Claude | Codex | Gemini | Consensus | Spread |
|----------------|--------|-------|--------|-----------|--------|
| Security       | 9/10   | 8/10  | 9/10   | 8.7/10    | 1      |
| Bugs & Logic   | 8/10   | 9/10  | 8/10   | 8.3/10    | 1      |
| Performance    | 7/10   | 7/10  | 9/10   | 7.7/10    | 2      |
| Architecture   | 9/10   | 9/10  | 8/10   | 8.7/10    | 1      |
```

Spread > 2 is flagged — it means one model saw something others missed.

### Confidence Levels

| Level | Meaning | Action |
|-------|---------|--------|
| **HIGH** | 2+ models found the same issue | Fix immediately |
| **MEDIUM** | 1 model found it | Verify, likely valid |
| **REVIEW** | Models disagree | Claude arbitrates |

## Adding Custom Models

Any CLI that accepts a prompt and returns text works. See [config.md](config.md) for full details.

```
Name: ollama-qwen
Verify: ollama list | grep qwen2.5-coder
Invoke: echo "{PROMPT}" | ollama run qwen2.5-coder:32b
Workspace access: No
Timeout: 600s
```

Supported out of the box: **Codex CLI**, **Gemini CLI**

Community examples: **Aider**, **Ollama**, **sgpt**, **custom scripts**

## File Structure

```
multi-ai-review/
├── SKILL.md                    # Main orchestration (loaded on trigger)
├── config.md                   # Model registry + profile definitions
├── references/
│   ├── domains.md              # 16 review domains (8 code + 8 doc)
│   ├── synthesis.md            # Consensus algorithm + report template
│   └── prompts.md              # Prompt templates for external models
├── README.md                   # This file
└── CHANGELOG.md                # Version history
```

**Progressive disclosure:** Only `SKILL.md` loads initially. Reference files load on-demand
based on the current review phase, keeping context window usage efficient.

## Design Principles

1. **Redundancy over division** — Every model reviews everything. No splitting work.
2. **Confidence from consensus** — Multi-model agreement is the strongest signal.
3. **Structured output** — Same format from every model enables automated synthesis.
4. **Progressive depth** — Quick for small changes, deep for critical code.
5. **Project-aware** — Reads your CLAUDE.md for project-specific standards.
6. **Graceful degradation** — Works with 1 model, better with 2, best with 3+.
7. **Iterative** — Fix-and-re-review until consensus, max 3 iterations.

## Customization

### Project-Specific Standards

Add to your `CLAUDE.md`:

```markdown
## Multi-AI Review Defaults
- Default profile: standard
- Default models: codex, gemini
- Always include domains: security
- Extra criteria: Must follow our API versioning pattern
```

### Override on Invocation

```
review with the deep profile using codex and gemini, run 3 passes
review only the security and performance domains, critical and important only
review but skip the scalability and maintainability domains
```

## Requirements

- **Claude Code** (any version with skills support)
- **At least one external AI CLI** (Codex, Gemini, Aider, Ollama, etc.)
- External CLIs must be authenticated and working independently

## License

MIT — use freely, contribute improvements.

## Contributing

1. Fork this repository
2. Add your improvement (new domain criteria, model support, profiles)
3. Test with the skill itself: `review your changes as a doc review` on your changes
4. Submit a PR

The best contributions are battle-tested improvements from real review sessions.
