# Review Domains — Detailed Criteria

Each domain defines what to look for during a review. Domains are grouped into **Code Domains** (for source code review) and **Document Domains** (for spec/plan/doc review).

Every configured model reviews ALL active domains independently. No division of labor.

---

## Code Domains

### 1. Security

**Goal:** Identify vulnerabilities that could be exploited in production.

| Check | What to Look For |
|-------|-----------------|
| Injection | SQL injection, NoSQL injection, command injection, LDAP injection, template injection |
| XSS | Reflected, stored, and DOM-based cross-site scripting; unsanitized user input in HTML/JSX |
| Authentication | Missing auth checks on protected routes, weak password policies, session fixation |
| Authorization | Broken access control, privilege escalation, IDOR (insecure direct object references) |
| Data exposure | PII in logs, API keys in source, sensitive data in error messages, overly permissive CORS |
| CSRF | Missing CSRF tokens on state-changing operations, SameSite cookie attributes |
| Path traversal | Unsanitized file paths, directory traversal in file upload/download |
| Cryptography | Weak algorithms (MD5, SHA1 for security), hardcoded secrets, insufficient entropy |
| Dependencies | Known vulnerable packages, outdated dependencies with CVEs |
| Headers | Missing security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options) |
| Multi-tenancy | Tenant isolation failures, cross-tenant data leakage, missing tenant_id filters |

**Severity guide:**
- Critical: Exploitable vulnerability with production impact (injection, auth bypass, data leak)
- Important: Weakness that needs defense-in-depth (missing headers, weak validation)
- Suggestion: Hardening opportunity (additional logging, rate limiting)

### 2. Bugs & Logic

**Goal:** Find correctness errors that cause wrong behavior.

| Check | What to Look For |
|-------|-----------------|
| Logic errors | Wrong boolean conditions, inverted comparisons, incorrect operator precedence |
| Edge cases | Empty arrays, null/undefined, zero values, negative numbers, empty strings |
| Off-by-one | Loop bounds, array indexing, pagination (page 0 vs page 1), range boundaries |
| Race conditions | Concurrent access without locks, TOCTOU bugs, async operation ordering |
| Type safety | Implicit type coercion, unchecked casts, missing type narrowing |
| State management | Stale state, missing state transitions, inconsistent state updates |
| Error propagation | Swallowed errors, wrong error types thrown, missing error context |
| Boundary conditions | Integer overflow, string length limits, file size limits, timeout handling |

**Severity guide:**
- Critical: Bug that causes data corruption, crashes, or wrong business logic
- Important: Edge case that fails silently or produces confusing behavior
- Suggestion: Defensive check that would prevent future bugs

### 3. Robustness

**Goal:** Ensure the code handles failure gracefully.

| Check | What to Look For |
|-------|-----------------|
| Error handling | Try/catch at appropriate boundaries, meaningful error messages, no bare `catch {}` |
| Input validation | Validation at system boundaries (API routes, form handlers), schema validation (zod, etc.) |
| Defensive coding | Null checks before access, optional chaining where appropriate, default values |
| Timeout/retry | Network call timeouts, retry with backoff, circuit breaker patterns |
| Resource cleanup | Closing connections, clearing intervals/timeouts, releasing locks |
| Idempotency | Safe to retry operations, duplicate request handling |
| Graceful degradation | Fallback behavior when external services are down, partial failure handling |

### 4. Performance

**Goal:** Identify code that will be slow or resource-intensive.

| Check | What to Look For |
|-------|-----------------|
| N+1 queries | Querying inside loops, missing eager loading, batching opportunities |
| Memory | Memory leaks (event listeners, closures, caches), large object copies |
| Rendering | Unnecessary re-renders, missing memoization (useMemo, useCallback, React.memo) |
| Algorithms | O(n^2) where O(n) is possible, unnecessary sorting, redundant iterations |
| Caching | Missing cache for expensive operations, stale cache invalidation |
| Bundle size | Large imports (import entire lodash vs specific function), unused dependencies |
| Database | Missing indexes on queried columns, SELECT * when specific columns suffice |
| Network | Redundant API calls, missing request deduplication, large payload sizes |
| Lazy loading | Components/routes/images not lazy loaded when appropriate |

### 5. Architecture

**Goal:** Evaluate structural quality and design decisions.

| Check | What to Look For |
|-------|-----------------|
| Separation of concerns | Business logic in UI components, database calls in route handlers |
| Single responsibility | Functions/classes doing too many things, god objects |
| Coupling | Tight coupling between modules, implicit dependencies, global state abuse |
| Cohesion | Related functionality scattered across files, unrelated code grouped together |
| Patterns | Inconsistent patterns across similar code, anti-patterns (e.g., prop drilling) |
| Abstraction | Leaky abstractions, wrong abstraction level, premature abstraction |
| API design | REST conventions followed, consistent naming, proper HTTP status codes |
| Dependency direction | Inner layers depending on outer layers, circular dependencies |
| Extensibility | Hardcoded values that should be configurable, switch statements over polymorphism |

### 6. Scalability

**Goal:** Will this code work under 10x/100x load?

| Check | What to Look For |
|-------|-----------------|
| Bottlenecks | Single points of contention, unbounded queues, synchronous bottlenecks |
| Database | Missing connection pooling, long-running transactions, table locks |
| Concurrency | Lock contention, deadlock potential, thread safety |
| Statelessness | Server-side session state, in-memory caches without eviction |
| Horizontal scaling | Code that assumes single instance, file system dependencies |
| Rate limiting | Missing rate limits on public endpoints, no backpressure handling |
| Pagination | Unbounded queries, missing LIMIT/OFFSET, cursor-based pagination |
| Async processing | Long-running operations blocking request threads, missing job queues |

### 7. UX Impact

**Goal:** Evaluate user-facing quality of code changes.

| Check | What to Look For |
|-------|-----------------|
| Loading states | Missing loading indicators, no skeleton screens, jarring content shifts |
| Error messages | Generic "Something went wrong", missing actionable guidance, technical jargon |
| Accessibility | Missing ARIA labels, no keyboard navigation, insufficient color contrast (4.5:1 min) |
| Responsive design | Hardcoded widths, missing mobile breakpoints, overflow on small screens |
| Focus management | Focus not moved after modal open/close, missing focus trap in dialogs |
| Animation | No reduced motion support (prefers-reduced-motion), janky transitions |
| Touch targets | Buttons/links < 44px touch target, insufficient spacing between targets |
| Empty states | No messaging for empty lists/results, missing calls to action |
| Optimistic UI | Operations that could use optimistic updates for perceived performance |
| Consistency | Components that look different from established design system patterns |

### 8. Maintainability

**Goal:** Will future developers understand and safely modify this code?

| Check | What to Look For |
|-------|-----------------|
| Readability | Complex expressions without explanation, clever code over clear code |
| Naming | Misleading names, abbreviations, inconsistent naming conventions |
| Code duplication | Copy-pasted logic that should be extracted, similar components not unified |
| Documentation | Missing comments on non-obvious logic, outdated comments, missing JSDoc on exports |
| Test coverage | Untested critical paths, tests that test implementation rather than behavior |
| Configuration | Magic numbers, hardcoded URLs/IDs, missing environment variables |
| Technical debt | TODO comments without tracking, known workarounds, deprecated API usage |
| Observability | Missing logging at key decision points, no metrics for business operations |
| Error boundaries | Missing React error boundaries, unhandled promise rejections |
| Failure modes | What happens when this code fails? Is it obvious? Recoverable? |

---

## Document Domains

Used when the review profile is `doc`. These replace code domains entirely.

### 1. Completeness

**Goal:** Ensure nothing critical is missing that would block or derail implementation.

| Check | What to Look For |
|-------|-----------------|
| User personas | All target users identified with their needs and constraints |
| User flows | Step-by-step flows for all major paths, including unhappy paths |
| Edge cases | Boundary conditions enumerated, unusual scenarios addressed |
| Error scenarios | What happens when things fail? Documented for each flow |
| Success metrics | Measurable outcomes defined, baseline and target values |
| Acceptance criteria | Testable criteria for every requirement |
| Dependencies | External and internal dependencies identified |
| Constraints | Technical, business, and time constraints documented |
| Out of scope | Explicitly stated what this does NOT cover |
| Migration | Data migration, backward compatibility, rollback plan |

**Severity guide:**
- Critical: Missing requirement that would cause a failed launch or user harm (no error handling, missing auth flow)
- Important: Gap that would require a follow-up release to address (missing edge case, incomplete migration plan)
- Suggestion: Nice-to-have coverage that would make the spec more thorough (additional persona, edge case)

### 2. Technical Accuracy

**Goal:** Ensure the document is grounded in reality and technically sound.

| Check | What to Look For |
|-------|-----------------|
| Feasibility | All requirements are technically achievable with the stated stack and timeline |
| Performance | Performance implications considered and addressed with realistic expectations |
| Security | Security implications addressed proactively, threat model appropriate for the domain |
| Scalability | Scale requirements and approach documented with concrete growth assumptions |
| Technical debt | Acknowledged and planned for, with explicit payback timeline |
| Integration | Integration points with existing systems identified, including data flow direction |
| API contracts | API contracts match described behavior; request/response shapes, methods, and URLs are accurate |
| Error codes | Error codes and HTTP status codes documented correctly and consistently across endpoints |
| Data types | Data types and formats specified precisely (ISO 8601 dates, currency precision, ID formats) |
| Third-party limits | Third-party service limitations acknowledged (rate limits, quotas, SLAs, deprecation timelines) |
| Benchmarks | Performance benchmarks are realistic, evidence-based, and specify measurement methodology |

**Severity guide:**
- Critical: Technical claim that would cause implementation failure (impossible requirement, wrong API contract)
- Important: Gap that would cause rework or debugging (missing error codes, unspecified data formats)
- Suggestion: Enhancement that would improve implementation speed (better examples, clearer constraints)

### 3. Consistency

**Goal:** Ensure the document does not contradict itself, other docs, or the actual system.

| Check | What to Look For |
|-------|-----------------|
| Internal | No contradictions within the document (e.g., requirement A conflicts with requirement B) |
| Terminology | Same terms used consistently throughout; no synonyms for the same concept |
| Cross-document | Aligned with related specs, plans, and existing system behavior |
| Versioning | Migration and versioning strategy consistent with project norms |
| Formatting | Date formats, number formats, and units consistent throughout the document |
| Auth model | Authorization model described is consistent with the actual system architecture |
| Error handling | Error handling approach consistent across all flows and endpoints described |
| Priorities | Priority/severity definitions match those used in other project docs and tooling |
| Feature references | Referenced features, screens, or endpoints actually exist in the codebase or are planned |
| Naming | Field names, table names, and API paths match what exists in the schema and routes |

**Severity guide:**
- Critical: Contradiction that would cause conflicting implementations across teams
- Important: Inconsistency that would cause confusion or rework during implementation
- Suggestion: Minor formatting or naming inconsistency that reduces polish

### 4. Clarity

**Goal:** Ensure every requirement can be understood and implemented without guessing.

| Check | What to Look For |
|-------|-----------------|
| Ambiguity | No "should", "might", "could" without commitment; every requirement uses "must" or "will" |
| Specificity | Concrete numbers where applicable (not "fast" but "< 200ms p95 latency") |
| Definitions | Domain terms defined, acronyms expanded on first use, glossary for complex domains |
| Examples | Concrete examples for complex requirements, including sample inputs and expected outputs |
| Measurability | Success criteria are objectively measurable, not subjective ("intuitive", "clean") |
| Decision rationale | Why this approach was chosen over alternatives; trade-offs documented, not just the decision |
| Visual aids | Diagrams, flowcharts, or wireframes provided for complex flows and state transitions |
| Acceptance criteria | Acceptance criteria are binary (pass/fail), not subjective or open to interpretation |
| Edge case behavior | Edge cases have explicit expected behavior documented, not left to implementer judgment |

**Severity guide:**
- Critical: Ambiguity that would cause two developers to implement different behavior
- Important: Missing context that would require a follow-up conversation to resolve
- Suggestion: Additional example or diagram that would speed up implementation

### 5. User Experience

**Goal:** Ensure the spec considers the full range of real user experiences.

| Check | What to Look For |
|-------|-----------------|
| Intuitiveness | User flow is natural and discoverable without training or documentation |
| Efficiency | Minimal steps to complete tasks; no unnecessary friction or confirmation dialogs |
| Feedback | Clear feedback at each step (loading, success, error) with appropriate timing |
| Error recovery | Graceful handling of mistakes, easy to undo, clear path back to a good state |
| Progressive disclosure | Complexity revealed incrementally; advanced features don't overwhelm new users |
| Offline/degraded | Behavior specified for offline, slow connection, or degraded service scenarios |
| First-time vs returning | First-time user onboarding differentiated from returning user experience |
| Accessibility | Accessibility requirements stated explicitly (WCAG level, screen reader support, keyboard-only) |
| Internationalization | Localization and internationalization requirements addressed (RTL, date formats, currencies) |
| Performance budgets | Performance budgets defined (time to interactive, API response times, animation frame rates) |

**Severity guide:**
- Critical: Gap that would make the feature unusable for a significant user segment (no error recovery, inaccessible)
- Important: Gap that would cause user frustration or support tickets (missing loading state, no offline handling)
- Suggestion: Enhancement that would elevate the experience (better onboarding, smoother transitions)

### 6. Simplicity

**Goal:** Ensure the spec does not over-engineer, over-scope, or introduce unnecessary complexity.

| Check | What to Look For |
|-------|-----------------|
| Over-engineering | Is the simplest solution chosen? Are there simpler alternatives not considered? |
| Feature creep | Are all features necessary for v1? Can anything be deferred to v2 without losing core value? |
| Platform conventions | Consistent with platform norms (users don't need to learn new patterns) |
| Learnability | Can users figure it out without documentation or training? |
| Forgiveness | System tolerates and recovers from user mistakes without data loss |
| Gold-plating | No features that sound impressive but won't actually be used by real users |
| Configuration | Configuration options are minimal; sensible defaults preferred over exposing options |
| API surface | API surface area is minimal; fewer well-designed endpoints over many narrow ones |
| Upgrade path | Upgrade and migration path from current state is straightforward and low-risk |
| Rollback | Rollback procedure documented and tested; changes are reversible if something goes wrong |

**Severity guide:**
- Critical: Complexity that would derail the timeline or make the feature unmaintainable
- Important: Scope that should be deferred to keep v1 focused and shippable
- Suggestion: Simplification that would reduce implementation effort without losing value

### 7. Product Thinking

**Goal:** Evaluate whether the spec solves a real problem and is set up for measurable success.

| Check | What to Look For |
|-------|-----------------|
| Problem fit | Does this solve the right problem? Is the problem validated with evidence (user interviews, data)? |
| Use cases | Are obvious use cases missing? Are there adjacent scenarios that should be considered? |
| Competitive context | How does this compare to alternatives the user could choose instead? |
| Differentiation | What makes this better than existing solutions? Is the advantage defensible? |
| User value | Would users choose this? Pay for this? Recommend this? Is the value proposition clear? |
| Success metrics | Success metrics tied to business outcomes (revenue, retention, efficiency), not vanity metrics (page views) |
| Failure modes | Failure modes and their business impact documented; what happens if this feature fails silently? |
| Rollout strategy | Rollout strategy defined (feature flags, percentage rollout, canary, beta users) |
| Operational burden | Support and operational burden considered; who monitors this? Who gets paged? |
| Observability | Analytics and observability requirements defined; how will you know if this is working? |

**Severity guide:**
- Critical: Spec solves the wrong problem or has no way to measure success
- Important: Missing rollout strategy or failure mode analysis that increases launch risk
- Suggestion: Additional competitive context or success metric that would strengthen the case

### 8. Polish

**Goal:** Identify opportunities to elevate the experience from functional to memorable.

| Check | What to Look For |
|-------|-----------------|
| Micro-interactions | Thoughtful animations and transitions planned for state changes and user actions |
| Empty states | Engaging empty states with clear next actions, not just "No data" |
| Smart defaults | Auto-fill, remembered preferences, intelligent suggestions that reduce work |
| Contextual help | Tooltips, hints, and guidance where users might get stuck or make mistakes |
| Keyboard shortcuts | Power user efficiency considered for frequent actions |
| Undo/redo | Reversible actions where applicable; destructive actions require confirmation |
| Optimistic UI | Instant feedback with background processing for perceived speed |
| Skeleton loading | Content-shaped loading indicators that reduce perceived wait time |
| Error messages | Error messages are human-readable, actionable, and suggest a next step |
| Success celebration | Success states are celebrated appropriately (not just acknowledged with a toast) |
| Self-service onboarding | Onboarding flow is self-service; users can get started without contacting support |

**Severity guide:**
- Critical: Missing polish that would make the feature feel broken (no loading states, cryptic errors)
- Important: Missing delight that would reduce adoption or satisfaction (poor empty states, no undo)
- Suggestion: Opportunity to surprise and delight that would differentiate the product

---

## Approval Criteria

### Per-Model Approval

A model should mark its review as **APPROVED** when:
- Zero Critical findings remaining
- All Important findings either resolved or explicitly accepted with justification
- Overall quality score >= 7/10

A model should mark its review as **CHANGES_REQUESTED** when:
- 1+ Important findings that should be fixed
- Quality score 5-6/10

A model should mark its review as **BLOCKED** when:
- 1+ Critical findings remaining
- Quality score < 5/10
- Security vulnerability is exploitable

### Consensus Approval

Overall review is **APPROVED** when ALL models approve.
Overall review is **CHANGES_REQUESTED** when ANY model requests changes (but none block).
Overall review is **BLOCKED** when ANY model blocks.
