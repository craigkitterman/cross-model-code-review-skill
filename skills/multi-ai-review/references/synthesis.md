# Consensus Synthesis Engine

How to cross-reference findings from multiple AI reviewers into a single confidence-scored report.

---

## Step 1: Parse and Normalize

Convert each model's output into a common structure. For each finding:

```
{
  model: "codex" | "gemini" | "claude" | "{custom}",
  severity: "critical" | "important" | "suggestion",
  domain: "security" | "bugs" | "robustness" | "performance" | "architecture" | "scalability" | "ux" | "maintainability",
  file: "path/to/file.ts",
  line: 42,
  line_end: 55,    // optional: end of affected range
  title: "SQL injection in user query",
  description: "User input is concatenated directly into SQL query without parameterization",
  fix: "Use parameterized query with $1 placeholder",
  code_before: "...",  // optional
  code_after: "..."    // optional
}
```

If a model's output doesn't follow the structured format exactly, extract what's available.
Map free-form text to the closest domain and severity.

## Step 2: Cross-Reference Findings

### Match Algorithm

Two findings from different models match when ANY of these conditions are true:

1. **Exact match:** Same file AND line within 5 lines AND same domain
2. **Semantic match:** Same file AND same domain AND title/description addresses the same issue (even with different wording)
3. **Pattern match:** Different files with same anti-pattern identified (e.g., "N+1 query" in multiple locations) — note as a recurring pattern in the report summary, but preserve each file's finding as a separate item for tracking and fixing

### Confidence Assignment

| Match Type | Confidence | Label |
|------------|------------|-------|
| Found by 3+ models | **CRITICAL CONSENSUS** | Nearly certain this is a real issue |
| Found by 2 models | **HIGH** | Very likely a real issue, prioritize |
| Found by 1 model only | **MEDIUM** | Probably valid, verify context |
| Models explicitly conflict | **REVIEW** | Requires Claude arbitration |

### Severity Escalation Rule

**The highest severity assigned by ANY model wins.**

If Codex says "Important" and Gemini says "Critical" → Final severity: **Critical**.

Rationale: It's safer to over-prioritize than to under-prioritize. A finding that one model considers critical deserves critical attention even if another model disagrees.

Exception: Claude can de-escalate if it can provide specific technical reasoning why the finding is not actually at the higher severity level (e.g., "This code path is unreachable because X").

## Step 3: Resolve Conflicts

When models explicitly disagree (one says "fix this", another says "this is fine"):

### Resolution Priority

1. **Security concerns always win.** If one model identifies a security risk and another doesn't, treat it as a finding.
2. **Check project context.** Does the project's CLAUDE.md, linting rules, or architectural patterns favor one interpretation?
3. **Prefer the simpler solution.** When both approaches are valid, prefer the one that's simpler and more maintainable.
4. **Claude arbitrates.** Claude has full codebase context that external models lack. Use this advantage.
5. **Document the disagreement.** Even if resolved, note that models disagreed so the user can review.

### Conflict Documentation Format

```markdown
#### Conflict: {title}
- **Codex says:** {codex_position}
- **Gemini says:** {gemini_position}
- **Claude resolution:** {resolution_with_reasoning}
- **Confidence:** {confidence in resolution}
```

## Step 4: Score Calculation

### Per-Domain Scoring

Each model assigns a 1-10 score per domain. Calculate consensus:

```
consensus_score = average(all_model_scores_for_domain)
score_spread = max(scores) - min(scores)
```

**Flag outliers:** If `score_spread > 2`, highlight the disagreement in the report.
This often reveals areas where one model saw something others missed.

### Overall Score

```
overall_score = average(all_domain_consensus_scores)
```

Adjust downward if:
- Any Critical findings remain unresolved: -2 points
- Any Important findings remain unresolved: -0.5 per finding (max -2)
- Model count is less than 2 (reduced confidence): -1 point

### Verdict Logic

```
IF any_model.verdict == "BLOCKED":
    overall_verdict = "BLOCKED"
ELIF any_model.verdict == "CHANGES_REQUESTED":
    overall_verdict = "CHANGES_REQUESTED"
ELIF all_models.verdict == "APPROVED":
    overall_verdict = "APPROVED"
ELSE:
    overall_verdict = "CHANGES_REQUESTED"  # default to cautious
```

## Step 5: Agreement Analysis

Calculate and report:

```
unanimous_findings = findings where all models agree
majority_findings = findings where 2+ of 3+ models agree
single_findings = findings from only one model
conflicts_resolved = findings where models disagreed and Claude arbitrated
total_findings = sum of all unique findings

agreement_rate = total_findings > 0 ? (unanimous_findings / total_findings * 100) : 100
# When total_findings == 0 (clean review), agreement_rate = 100 (unanimous that code is clean)
```

High agreement rate (>70%) = Models see similar things → high confidence in findings
Low agreement rate (<40%) = Models see different things → each brings unique value
Very low agreement (<20%) = Check if prompts were clear and models had sufficient context

---

## Consensus Report Template

Use this structure for the final report output:

```markdown
# Multi-AI Review Report

**Profile:** {profile}
**Models:** {comma-separated model list}
**Files reviewed:** {count}
**Date:** {YYYY-MM-DD}
**Iterations:** {N}

---

## Verdict: {APPROVED | CHANGES_REQUESTED | BLOCKED}
## Overall Score: {X.X}/10

---

## Executive Summary

{2-4 sentences: overall code quality assessment, most significant finding,
standout strengths, and recommendation. Be specific — not "code is good"
but "auth layer is solid, but input validation is missing on 3 API endpoints."}

---

## Consensus Matrix

| Domain | Claude | {Model2} | {Model3} | Consensus | Spread |
|--------|--------|----------|----------|-----------|--------|
| Security | X/10 | X/10 | X/10 | X.X/10 | X |
| Bugs & Logic | X/10 | X/10 | X/10 | X.X/10 | X |
| Robustness | X/10 | X/10 | X/10 | X.X/10 | X |
| Performance | X/10 | X/10 | X/10 | X.X/10 | X |
| Architecture | X/10 | X/10 | X/10 | X.X/10 | X |
| Scalability | X/10 | X/10 | X/10 | X.X/10 | X |
| UX Impact | X/10 | X/10 | X/10 | X.X/10 | X |
| Maintainability | X/10 | X/10 | X/10 | X.X/10 | X |

{Flag any row where Spread > 2 with a note explaining the disagreement}

---

## Findings

### Critical — HIGH Confidence (consensus)

{Findings agreed upon by 2+ models. These are near-certain issues.}

1. **{Title}** [{domain}]
   - **File:** `{path}:{line}`
   - **Found by:** {model1}, {model2}
   - **Issue:** {description}
   - **Impact:** {why this matters}
   - **Fix:** {specific recommendation}

### Critical — MEDIUM Confidence (single model)

{Critical findings from only one model, verified by Claude.}

### Important

{Important findings grouped by domain.}

### Suggestions

{Nice-to-have improvements, style suggestions.}

---

## Conflicts Resolved

{List any findings where models disagreed, with Claude's resolution and reasoning.}

---

## Model Agreement Analysis

| Metric | Value |
|--------|-------|
| Total unique findings | {N} |
| Unanimous (all models) | {N} ({X}%) |
| Majority (2+ models) | {N} ({X}%) |
| Single model only | {N} ({X}%) |
| Conflicts resolved | {N} |
| Agreement rate | {X}% |

---

## Per-Model Verdicts

| Model | Verdict | Score | Key Concern |
|-------|---------|-------|-------------|
| Claude | {verdict} | X/10 | {one-line summary} |
| {Model2} | {verdict} | X/10 | {one-line summary} |
| {Model3} | {verdict} | X/10 | {one-line summary} |

---

## Iteration History

{If multiple passes were done, show how findings evolved:}

| Iteration | Critical | Important | Suggestions | Verdict |
|-----------|----------|-----------|-------------|---------|
| 1 | X | X | X | CHANGES_REQUESTED |
| 2 | X | X | X | APPROVED |

**Findings resolved between iterations:** {list}
**New findings introduced:** {list or "None"}

---

## Certification

{Only include this section when verdict is APPROVED}

This code has been independently reviewed by {N} AI models with redundant coverage
across {M} quality domains. All critical and important findings have been resolved.

**Review confidence:** {HIGH | MEDIUM | REDUCED}
**Recommendation:** Ready for merge.
```

---

## Re-Review Strategy

When re-reviewing after fixes:

1. **Scope:** Only re-review files that changed since the last review
2. **Focus:** Verify previously-identified findings are resolved
3. **Watch for:** New issues introduced by the fixes
4. **Prompt:** Use the re-review prompt template from `prompts.md`
5. **Compare:** Show delta between iterations (what improved, what's new)
6. **Max iterations:** 3 (then present final state to user regardless)
