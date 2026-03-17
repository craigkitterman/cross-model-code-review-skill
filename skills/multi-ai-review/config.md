# Multi-AI Review Configuration

## Model Registry

Define which AI CLIs are available for reviews. Claude always participates as the orchestrator and a reviewer.

### Built-in Model Definitions

#### Codex CLI (OpenAI)

Install: `npm install -g @openai/codex` (requires Node.js 22+)
Auth: `codex auth login` (opens browser for OAuth)

```
Name: codex
Verify: codex --version
Invoke: codex exec --full-auto "{PROMPT}"
Large prompts: cat {PROMPT_FILE} | codex exec --full-auto -
Built-in review: codex exec review (auto-reviews current repo changes)
Workspace access: Yes (sandboxed to current git repo root)
Timeout: 300s
Strengths: Strong at security analysis, API design, performance patterns
Notes:
  - Must be run from within a git repository
  - Sandbox restricts file access to the repo root only
  - Does NOT support --quiet flag (removed in recent versions)
  - Use stdin (-) for prompts >32KB to avoid ARG_MAX limits on Windows
```

#### Gemini CLI (Google)

Install: `npm install -g @google/gemini-cli`
Auth: Run `gemini` (first run triggers browser-based Google OAuth automatically)

```
Name: gemini
Verify: gemini --version
Invoke: gemini -p "{PROMPT}"
Large prompts: cat {PROMPT_FILE} | gemini -p -
Workspace access: Yes (reads local files in current directory)
Timeout: 300s
Strengths: Strong at architecture analysis, scalability, UX thinking
Notes:
  - The -p flag runs in non-interactive (headless/piped) mode
  - Can read files in the current workspace directory
  - Use stdin piping for prompts >32KB to avoid ARG_MAX limits
```

### Adding Custom Models

Any CLI that accepts a text prompt and returns text output can be added. Pattern:

```
Name: {unique_identifier}
Verify: {command} --version (or equivalent)
Invoke: {command} {flags} "{PROMPT}"
Large prompts: cat {PROMPT_FILE} | {command} {flags} -
Workspace access: Yes/No (can it read files in the current directory?)
Timeout: {seconds}
Strengths: {what this model is particularly good at}
```

#### Example: Aider

```
Name: aider
Verify: aider --version
Invoke: aider --message "{PROMPT}" --no-auto-commits --yes
Large prompts: aider --message-file {PROMPT_FILE} --no-auto-commits --yes
Workspace access: Yes
Timeout: 300s
Strengths: Deep codebase understanding, refactoring suggestions
```

#### Example: Ollama (Local Models)

```
Name: ollama-qwen
Verify: ollama list | grep qwen2.5-coder
Invoke: echo "{PROMPT}" | ollama run qwen2.5-coder:32b
Large prompts: cat {PROMPT_FILE} | ollama run qwen2.5-coder:32b
Workspace access: No (must include file contents in prompt)
Timeout: 600s
Strengths: Privacy-sensitive reviews, no data leaves machine
```

#### Example: sgpt (Shell GPT)

```
Name: sgpt
Verify: sgpt --version
Invoke: sgpt "{PROMPT}"
Large prompts: cat {PROMPT_FILE} | sgpt
Workspace access: No
Timeout: 120s
Strengths: Quick lightweight reviews
```

#### Example: Custom Script Wrapper

```
Name: internal-reviewer
Verify: ./scripts/review-cli.sh --health
Invoke: ./scripts/review-cli.sh --review "{PROMPT}"
Large prompts: ./scripts/review-cli.sh --review-file {PROMPT_FILE}
Workspace access: Depends on script
Timeout: 300s
Strengths: Company-specific rules baked in
```

### Model Selection Strategy

| Scenario | Recommended Models |
|----------|--------------------|
| Quick feedback | 1 external (fastest available) + Claude |
| Standard review | Codex + Gemini + Claude |
| Maximum coverage | All available + Claude |
| Privacy-sensitive code | Ollama (local) + Claude |
| Company-specific rules | Custom script + 1 external + Claude |

### Workspace Access Implications

**Models WITH workspace access** (Codex, Gemini, Aider):
- Can read files directly from the working directory
- Prompt can reference file paths instead of including full contents
- Better for large codebases (avoids prompt size limits)
- Include instruction: "Read the files at these paths and review them: {FILE_LIST}"

**Models WITHOUT workspace access** (Ollama, sgpt):
- Must include full file contents inline in the prompt
- Subject to model's context window limits
- For large diffs, may need to chunk and review in parts
- Include the diff/file contents directly in the prompt text

---

## Review Profiles

### `quick` — Fast Feedback

**Purpose:** Rapid quality check for small changes. Single external model.

```
Domains: Bugs & Logic, Maintainability (2 of 8)
Models: 1 external (prefer fastest) + Claude
Passes: 1 (no re-review loop)
Severity threshold: Only report Critical and Important
Max files: 10
```

Best for: typo fixes, config changes, small refactors, dependency bumps.

### `standard` — Default Review

**Purpose:** Comprehensive review for typical development work.

```
Domains: All 8 code domains
Models: 2 external + Claude
Passes: 1-2 (re-review if Critical findings fixed)
Severity threshold: All findings reported
Max files: 50
```

Best for: feature branches, bug fixes, most pull requests.

### `deep` — Thorough Analysis

**Purpose:** Maximum coverage for critical or complex changes.

```
Domains: All 8 code domains
Models: All available + Claude
Passes: 2-3 (iterate until consensus APPROVED)
Severity threshold: All findings including style suggestions
Max files: No limit (chunk if needed)
Note: All models review the same code independently — no second-order peer review
```

Best for: security-sensitive features, major refactors, pre-release reviews.

### `security` — Security Audit

**Purpose:** Focused security analysis.

```
Domains: Security, Robustness, Bugs & Logic (3 of 8, security-weighted)
Models: All available + Claude
Passes: 2
Severity threshold: All security-adjacent findings
Focus areas:
  - Authentication and authorization
  - Input validation and sanitization
  - Data exposure and leakage
  - Injection vulnerabilities (SQL, XSS, command, path traversal)
  - Secrets and credential handling
  - CSRF, CORS, and header security
  - Dependency vulnerabilities
  - Cryptographic usage
```

Best for: auth changes, API endpoints, data handling, payment flows.

### `ux` — UX & Accessibility Review

**Purpose:** User experience and accessibility analysis.

```
Domains: UX Impact, Performance, Architecture (3 of 8, UX-weighted)
Focus areas: Accessibility checks are part of the UX Impact domain
Models: All available + Claude
Passes: 1-2
Focus areas:
  - Accessibility (WCAG 2.1 AA compliance)
  - Loading states and skeleton screens
  - Error message clarity and recovery paths
  - Responsive design and mobile experience
  - Keyboard navigation and focus management
  - Color contrast and visual hierarchy
  - Animation and reduced motion support
  - Touch target sizes (>= 44px)
  - Component library consistency (shadcn/ui, Radix, etc.)
  - Perceived performance and optimistic UI
```

Best for: UI components, user flows, design system work.

### `doc` — Document Review

**Purpose:** Review specs, plans, and documentation for quality and completeness.

```
Domains: 8 document-specific domains (different from code domains):
  1. Completeness
  2. Technical Accuracy
  3. Consistency
  4. Clarity
  5. User Experience
  6. Simplicity
  7. Product Thinking
  8. Polish
Models: 2 external + Claude
Passes: 1-2
```

Best for: feature specs, implementation plans, PRDs, API documentation.

### `pre-merge` — Merge Gate

**Purpose:** Final quality gate before merging to main/production branch.

```
Domains: All 8 code domains
Models: All available + Claude
Passes: 1 (no iteration — this is a go/no-go gate)
Additional focus: Claude also checks for common merge-readiness issues:
  - TODO/FIXME/HACK comments in changed code (grep for them)
  - console.log/debug statements left in production code
  - Secrets or credentials accidentally included in diff
  - Breaking changes that should be documented
```

Best for: final review before merge to main, release branches.

Note: These additional checks are performed by Claude using Grep/Read tools during Phase 3 (self-review). External models focus on the standard 8 code domains.

---

## Customization

### Adjusting Profiles via Natural Language

Tell Claude what you want in plain English:

```
/review but focus only on security and performance
/review quick — just bugs, nothing else
/review deep — I want all models and at least 2 passes
/review but skip scalability checks, this is a prototype
```

Claude interprets these instructions and adjusts the profile parameters accordingly.

### Project-Specific Defaults

Add to your project's `CLAUDE.md`:

```markdown
## Multi-AI Review Defaults
- Default profile: standard
- Default models: codex, gemini
- Always include domains: security
- Extra review criteria: Must follow our API versioning pattern
- Custom severity rules: Treat any console.log in production code as Important
```

These defaults are read during Phase 1 (context gathering) and included in every review prompt.
