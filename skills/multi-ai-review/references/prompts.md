# Prompt Templates for External Models

These templates are used to construct the review prompt sent to each external model CLI.
Claude fills in the template variables and sends the complete prompt.

**Key principle:** Every model gets the SAME prompt structure so outputs are comparable.
Adjust only for model-specific capabilities (e.g., workspace access vs inline content).

---

## Template Variables

| Variable | Source | Description |
|----------|--------|-------------|
| `{REVIEW_TYPE}` | Profile | "Code Review" or "Document Review" |
| `{PROFILE}` | User selection | quick, standard, deep, security, ux, doc, pre-merge |
| `{DOMAINS}` | Profile config | Comma-separated list of active domains |
| `{DOMAIN_CRITERIA}` | `references/domains.md` | Detailed criteria for each active domain |
| `{FILE_LIST}` | Phase 1 | List of files to review |
| `{DIFF_CONTENT}` | git diff | The actual code diff or file contents |
| `{PROJECT_CONTEXT}` | CLAUDE.md etc. | Project-specific standards and conventions |
| `{FILE_COUNT}` | Phase 1 | Number of files being reviewed |
| `{DOC_TYPE}` | Profile | Document type: "Feature Spec", "Implementation Plan", "PRD", etc. |
| `{DOCUMENT_CONTENT}` | Read tool | The full content of the document being reviewed |
| `{RELATED_DOCS_CONTENT}` | Read tool | Content of related/adjacent documents for cross-reference |
| `{APP_TYPE}` | Project context | Application type: "web app", "API", "mobile app", etc. |
| `{DESIGN_SYSTEM}` | Project context | Design system in use: "shadcn/ui", "Material UI", "custom", etc. |
| `{ITERATION_NUMBER}` | Phase 6 | Current re-review iteration (1, 2, 3) |
| `{PREVIOUS_FINDINGS_LIST}` | Phase 6 | Formatted list of findings from previous iteration |
| `{NEW_DIFF_CONTENT}` | git diff | Changes made since the last review iteration |

---

## Code Review Prompt Template

Use this template for all code review profiles (quick, standard, deep, security, ux, pre-merge).

```
You are performing an independent senior code review. Be thorough and practical.
Prioritize production risk and correctness over style preferences.

REVIEW PROFILE: {PROFILE}
FILES TO REVIEW: {FILE_COUNT} files
ACTIVE DOMAINS: {DOMAINS}

=== PROJECT CONTEXT ===
{PROJECT_CONTEXT}

=== FILES AND CHANGES ===
{DIFF_CONTENT}

=== REVIEW CRITERIA ===
Review ALL of the following domains independently. For each domain, provide specific
file:line references and actionable fix recommendations.

{DOMAIN_CRITERIA}

=== OUTPUT FORMAT (follow exactly) ===

### VERDICT: [APPROVED | CHANGES_REQUESTED | BLOCKED]
### QUALITY_SCORE: [1-10]

### Domain Scores
- {domain1}: X/10
- {domain2}: X/10
(one line per active domain)

### Findings

#### Critical (must fix before merge)
1. **[Finding Title]** [{domain}]
   - File: path/to/file.ts:42
   - Issue: [What is wrong and why it matters]
   - Impact: [Production risk if not fixed]
   - Fix: [Specific code change or approach]

(If none, write "None")

#### Important (should fix)
1. **[Finding Title]** [{domain}]
   - File: path/to/file.ts:42
   - Issue: [Description]
   - Fix: [Recommendation]

(If none, write "None")

#### Suggestions (nice to have)
1. **[Finding Title]** [{domain}]
   - File: path/to/file.ts:42
   - Suggestion: [Improvement idea]

(If none, write "None")

### Strengths
[2-3 things done well, with specific file:line references]

### Summary
[2-3 sentence overall assessment]

=== RULES ===
- Be strict and practical. Prioritize correctness and production risk.
- Every finding MUST include a specific file path and line number.
- Every finding MUST include a concrete fix recommendation.
- Do not invent issues. If uncertain, state what to verify.
- Do not mark style preferences as Critical.
- Security issues are always at least Important severity.
- If the code is solid, say so. Don't manufacture findings.
```

### Adapting for Models WITHOUT Workspace Access

For models that can't read files (Ollama, sgpt), include the full diff inline:

```
=== FILES AND CHANGES ===
The following is the complete diff of changes to review:

--- a/src/auth/login.ts
+++ b/src/auth/login.ts
@@ -15,6 +15,10 @@
 [full diff content here]
```

### Adapting for Models WITH Workspace Access

For models that can read files (Codex, Gemini, Aider), reference file paths:

```
=== FILES AND CHANGES ===
Read and review the following files from the workspace:

Changed files:
- src/auth/login.ts (lines 15-45 modified)
- src/api/users/route.ts (new file)
- src/middleware/validate.ts (lines 8-12 deleted)

The git diff summary:
[include git diff --stat output]

Read each file completely for full context, then review the changes.
```

---

## Document Review Prompt Template

Use this template when the review profile is `doc`.

```
You are performing an independent review of technical documentation.
Evaluate completeness, accuracy, and quality. Be constructive and specific.

DOCUMENT TYPE: {DOC_TYPE}
REVIEW PROFILE: doc
ACTIVE DOMAINS: {DOMAINS}

=== DOCUMENT CONTENT ===
{DOCUMENT_CONTENT}

=== RELATED DOCUMENTS ===
{RELATED_DOCS_CONTENT}

=== REVIEW CRITERIA ===
Review ALL of the following domains independently:

{DOMAIN_CRITERIA}

=== OUTPUT FORMAT (follow exactly) ===

### VERDICT: [APPROVED | CHANGES_REQUESTED | BLOCKED]
### QUALITY_SCORE: [1-10]

### Domain Scores
- Completeness: X/10
- Technical Accuracy: X/10
- Consistency: X/10
- Clarity: X/10
- User Experience: X/10
- Simplicity: X/10
- Product Thinking: X/10
- Polish: X/10

### Gaps Found

#### Critical (must address before implementation)
1. **[Gap Title]** [{domain}]
   - Section: [Which section has the gap]
   - Gap: [What is missing or wrong]
   - Impact: [Why this matters]
   - Recommendation: [How to fix]

#### Important (should address)
1. **[Gap Title]** [{domain}]
   - Section: [Section reference]
   - Gap: [Description]
   - Recommendation: [Fix]

#### Enhancement Ideas
1. **[Idea Title]** [{domain}]
   - Benefit: [How it improves the document]
   - Effort: [Low/Medium/High]

### Strengths
[2-3 things done well]

### Summary
[2-3 sentence overall assessment of document quality and readiness]

=== RULES ===
- Focus on gaps that could cause implementation problems.
- Distinguish between "missing" (not covered) and "unclear" (covered but ambiguous).
- Don't suggest adding unnecessary complexity.
- Check for internal contradictions.
- Verify that acceptance criteria are testable.
- If the document is thorough, say so. Don't manufacture gaps.
```

---

## Re-Review Prompt Template

Use this template when re-reviewing after fixes have been applied.

```
You previously reviewed this code and found issues. The developer has implemented fixes.
Re-review to verify the fixes are correct and no new issues were introduced.

ITERATION: {ITERATION_NUMBER}
PREVIOUS FINDINGS TO VERIFY:
{PREVIOUS_FINDINGS_LIST}

=== CHANGES SINCE LAST REVIEW ===
{NEW_DIFF_CONTENT}

=== OUTPUT FORMAT (follow exactly) ===

### VERDICT: [APPROVED | CHANGES_REQUESTED | BLOCKED]
### QUALITY_SCORE: [1-10]

### Previous Findings Status

| # | Finding | Status | Notes |
|---|---------|--------|-------|
| 1 | {finding_title} | FIXED / NOT_FIXED / PARTIALLY_FIXED | {detail} |
| 2 | {finding_title} | FIXED / NOT_FIXED / PARTIALLY_FIXED | {detail} |

### New Issues Found
{Any NEW issues introduced by the fixes, using same format as initial review}
(If none, write "None — fixes look clean")

### Summary
[Was the fix iteration successful? What remains?]
```

---

## Security Audit Prompt Template

Extended prompt for the `security` profile with deeper security focus.

```
You are performing a SECURITY AUDIT. This is not a general code review.
Focus exclusively on security vulnerabilities, attack vectors, and defensive gaps.
Assume an adversarial threat model: a skilled attacker with knowledge of common web vulnerabilities.

AUDIT SCOPE: {FILE_COUNT} files
APPLICATION TYPE: {APP_TYPE}

=== FILES TO AUDIT ===
{DIFF_CONTENT}

=== SECURITY CHECKLIST ===
For each file, systematically check:

INJECTION ATTACKS:
- [ ] SQL injection (parameterized queries used?)
- [ ] NoSQL injection (MongoDB operators in user input?)
- [ ] Command injection (child_process, exec with user input?)
- [ ] XSS (user content rendered without sanitization?)
- [ ] Template injection (user input in template strings?)
- [ ] Path traversal (user input in file paths?)
- [ ] LDAP/XML injection (if applicable)

AUTHENTICATION & AUTHORIZATION:
- [ ] Auth check on every protected route
- [ ] Session management secure (httpOnly, secure, SameSite cookies)
- [ ] Password handling (hashed with bcrypt/argon2, never logged)
- [ ] Token validation (JWT expiry, signature verification)
- [ ] Privilege escalation (can user access other users' data?)
- [ ] IDOR (direct object references without ownership check)

DATA PROTECTION:
- [ ] PII not logged or exposed in error messages
- [ ] API keys/secrets not in source code
- [ ] Sensitive data encrypted at rest and in transit
- [ ] Minimum necessary data returned in API responses
- [ ] CORS configured restrictively (not *)

INFRASTRUCTURE:
- [ ] Security headers set (CSP, HSTS, X-Frame-Options)
- [ ] Rate limiting on auth and public endpoints
- [ ] CSRF protection on state-changing operations
- [ ] Dependencies free of known CVEs

MULTI-TENANT (if applicable):
- [ ] Tenant isolation in database queries (WHERE tenant_id = ?)
- [ ] RLS policies active and correct
- [ ] Cross-tenant data leakage impossible
- [ ] Tenant context validated on every request

=== OUTPUT FORMAT (follow exactly) ===

### VERDICT: [APPROVED | CHANGES_REQUESTED | BLOCKED]
### QUALITY_SCORE: [1-10]

### Domain Scores
- Security: X/10
- Robustness: X/10
- Bugs & Logic: X/10

### Findings

#### Critical (must fix before merge)
{Using standard finding format: title, domain, file:line, issue, impact, fix}

(If none, write "None")

#### Important (should fix)
{Same format}

(If none, write "None")

#### Suggestions (nice to have)
{Same format}

(If none, write "None")

### Security Checklist Results
{Mark each checklist item as PASS, FAIL, or N/A with brief notes}

### Attack Surface Summary
{Brief description of the attack surface and highest-risk areas}

### Strengths
[2-3 security strengths with specific file:line references]

### Summary
[2-3 sentence security assessment]
```

---

## UX Review Prompt Template

Extended prompt for the `ux` profile.

```
You are performing a UX AND ACCESSIBILITY REVIEW of user-facing code changes.
Focus on the end-user experience: is this usable, accessible, and delightful?

SCOPE: {FILE_COUNT} UI files
DESIGN SYSTEM: {DESIGN_SYSTEM}

=== FILES TO REVIEW ===
{DIFF_CONTENT}

=== UX CHECKLIST ===

ACCESSIBILITY (WCAG 2.1 AA):
- [ ] All interactive elements have accessible names (aria-label, aria-labelledby)
- [ ] Color contrast ratio >= 4.5:1 for text, >= 3:1 for large text
- [ ] All functionality available via keyboard
- [ ] Focus visible and logical tab order
- [ ] Images have meaningful alt text (or alt="" for decorative)
- [ ] Form inputs have associated labels
- [ ] Error messages announced to screen readers (aria-live or role="alert")
- [ ] prefers-reduced-motion respected for animations

LOADING & STATES:
- [ ] Loading state shown during async operations
- [ ] Skeleton screens for content-heavy areas (not just spinners)
- [ ] Empty state with clear call to action
- [ ] Error state with recovery guidance
- [ ] Success feedback (toast, animation, etc.)
- [ ] Optimistic UI where appropriate

RESPONSIVE & MOBILE:
- [ ] Works at 320px viewport width minimum
- [ ] Touch targets >= 44x44px
- [ ] No horizontal scroll on mobile
- [ ] Adequate spacing between touch targets (>= 8px gap)
- [ ] Content readable without zooming

DESIGN SYSTEM COMPLIANCE:
- [ ] Uses design system components (shadcn/ui, Radix, etc.)
- [ ] Consistent with existing UI patterns in the app
- [ ] Follows spacing system (4px/8px grid)
- [ ] Typography consistent with design tokens
- [ ] Colors from design token palette (no hardcoded hex)

=== OUTPUT FORMAT ===
### VERDICT: [APPROVED | CHANGES_REQUESTED | BLOCKED]
### QUALITY_SCORE: [1-10]

### Domain Scores
- UX Impact: X/10
- Performance: X/10
- Architecture: X/10

### Issues Found
{Using standard finding format with severity, file:line, and fix}

### Checklist Results
{Mark each item as PASS, FAIL, or N/A}

### UX Strengths
{What's done well from a user experience perspective}
```

---

## Prompt Size Management

When the prompt exceeds ~30,000 characters (rough limit for most CLIs):

1. **Summarize project context** — Include only the most relevant CLAUDE.md rules
2. **Chunk files** — Split into batches of 5-10 files, review each batch separately
3. **Diff only** — Send only the diff, not full file contents
4. **Use temp files** — Write prompt to `/tmp/review-prompt-{model}.txt` and pass as file input
5. **Prioritize** — If chunking, review security-sensitive files first

### Writing Prompt to Temp File

```bash
# Write prompt to temp file
cat > /tmp/review-prompt.txt << 'REVIEW_EOF'
{COMPLETE_PROMPT}
REVIEW_EOF

# Invoke models via stdin piping (avoids ARG_MAX limits on Windows)
cat /tmp/review-prompt.txt | codex exec --full-auto -
cat /tmp/review-prompt.txt | gemini -p -

# Clean up
rm /tmp/review-prompt.txt
```

**IMPORTANT:** Do NOT use `$(cat file)` for large prompts — this still passes the content
as a shell argument and will hit OS ARG_MAX limits (~32KB on Windows). Always use stdin piping.
