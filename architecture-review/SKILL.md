---
name: architecture-review
description: Analyse technical, code, and implementation design decisions before building. Runs 7 architectural frameworks (SWOT, GAP, SEARCH, STRIDE, ATAM, C4, ADR) against a proposed change to surface risks, tradeoffs, and gaps before any code is written. Pass a description of the change as argument (e.g., "add marketplace skill routing", "refactor queue to use Redis streams").
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Agent
  - Task
---

# Architecture Review — Pre-Implementation Analysis Skill

Run a structured architectural analysis on a proposed technical change **before** writing any code. The goal is to catch design flaws, security holes, scaling limits, and migration gaps upfront.

## Arguments

`$ARGUMENTS` — Description of the proposed change, feature, or design decision to analyse.

Examples:
- `/architecture-review add marketplace skill execution to V6ToolRegistry`
- `/architecture-review migrate queue backend from database to Redis streams`
- `/architecture-review add multi-tenant secret isolation for installed workflows`
- `/architecture-review refactor ReactLoopService checkpointing to be async`

---

## How This Skill Works

When invoked, run **all 7 frameworks** against the proposed change. For each framework, read the relevant source files to ground the analysis in actual code — never speculate about implementation details without reading them first.

Output a single structured report with all 7 sections, then a final **Go / No-Go / Conditional Go** recommendation.

---

## Framework 1: SWOT Analysis — Strategic Viability

Evaluate the proposed change from a strategic perspective.

| Category | What to assess |
|----------|---------------|
| **Strengths** | What existing code/patterns does this leverage? How much reuse vs new code? What safety mechanisms does it inherit? |
| **Weaknesses** | What's brittle, hardcoded, or fragile in the approach? What coupling does it introduce? |
| **Opportunities** | What future capabilities does this unlock? Revenue, scale, or ecosystem benefits? |
| **Threats** | What could go wrong in production? Data leaks, race conditions, sync drift, breaking changes? |

**Source check**: Read the files that will be modified. Identify the exact functions/classes affected.

---

## Framework 2: GAP Analysis — Transition Planning

Map the journey from current state to target state.

1. **Current State**: What exists today? Read the actual code. What does it do, what doesn't it do?
2. **Target State**: What should exist after this change? Be specific about behaviour, not just structure.
3. **The Gap**: What's missing? List each discrete piece of work.
4. **Bridge (Action Plan)**: Ordered steps to close the gap. Flag any steps that require migrations, env var changes, or cross-service coordination.

**Source check**: Read the current implementation files. Identify what already exists vs what needs building.

---

## Framework 3: SEARCH — System Traits Assessment

Evaluate 6 non-functional requirements. Rate each as Low / Medium / High / Exceptional with a one-line justification.

| Trait | Question |
|-------|----------|
| **S — Scalability** | Does this change scale horizontally? What's the bottleneck (DB writes, memory, API calls)? |
| **E — Extensibility** | Can future developers extend this without modifying the core? Is it pluggable? |
| **A — Availability** | What happens when a dependency fails? Is there a fallback? Graceful degradation? |
| **R — Reliability** | Can this produce incorrect results silently? What invariants could be violated? |
| **C — Consistency** | In concurrent/async scenarios, can state become inconsistent? Race conditions? |
| **H — Health / Observability** | Can we tell if this is working? Logs, metrics, health checks, alerts? |

---

## Framework 4: STRIDE — Threat Modelling

For each STRIDE category, assess whether the proposed change introduces or mitigates the threat. Only flag categories that are **actually relevant** — skip categories with no meaningful exposure.

| Threat | Definition | Assessment approach |
|--------|-----------|-------------------|
| **Spoofing** | Faking identity | Can an unauthorized actor trigger this? Are auth checks in place? |
| **Tampering** | Modifying data | Can inputs/state be manipulated to produce unintended behaviour? |
| **Repudiation** | Denying actions | Is there an audit trail? Can we prove what happened? |
| **Information Disclosure** | Data leaks | Could secrets, PII, or internal state leak across tenant boundaries? |
| **Denial of Service** | Resource exhaustion | Can this be abused to exhaust CPU, memory, DB connections, API quotas? |
| **Elevation of Privilege** | Gaining access | Could this bypass permission checks or escalate user capabilities? |

**Source check**: Read auth middleware, secret management, and permission checks in the affected code paths.

---

## Framework 5: ATAM — Architecture Tradeoff Analysis

Identify the **top 3 architectural decisions** in the proposed change. For each:

1. **Decision**: What architectural choice is being made?
2. **Optimises for**: Which quality attribute does this favour? (performance, reliability, extensibility, etc.)
3. **Trades off**: Which quality attribute suffers? Be specific about the cost.
4. **Sensitivity point**: Under what conditions does this tradeoff become painful?
5. **Mitigation**: What could reduce the downside without losing the upside?

---

## Framework 6: C4 Model — Structural Impact

Zoom through 4 levels to show where this change fits architecturally. Only go as deep as needed — skip levels that aren't affected.

- **Level 1 — Context**: How does this change affect system boundaries? New external dependencies? Changed data flows between systems?
- **Level 2 — Containers**: Which services/containers are affected? (iris-api, fl-api, fl-elon-web-ui, n8n, daemon, etc.)
- **Level 3 — Components**: Which modules/services/controllers within those containers are modified?
- **Level 4 — Code**: Which specific classes, methods, or functions change? What's the call chain?

Present as a concise list, not a diagram. Reference actual file paths.

---

## Framework 7: ADR — Architecture Decision Record

Write a draft ADR for the most significant decision in this change.

```
## ADR: [Title]

### Status
Proposed

### Context
[Why is this decision needed? What problem does it solve?]

### Decision
[What are we doing? Be specific about the implementation approach.]

### Consequences
**Positive:**
- [benefit 1]
- [benefit 2]

**Negative:**
- [cost 1]
- [cost 2]

**Risks:**
- [risk + mitigation]
```

---

## Final Verdict

After all 7 frameworks, deliver a clear recommendation:

### Recommendation: **[Go / No-Go / Conditional Go]**

**If Go**: List the implementation order (what to build first, what to defer).

**If Conditional Go**: List the specific conditions that must be met before proceeding (e.g., "add migration for X first", "verify Y doesn't break Z").

**If No-Go**: Explain why and propose an alternative approach.

### Critical Risks (if any)
- Numbered list of risks that could cause production incidents if ignored.

### Suggested Implementation Order
1. Step 1 — [what + why first]
2. Step 2 — [what + why second]
3. ...

---

## Research Protocol

Before writing the analysis, you MUST:

1. **Read the affected files** — Use Glob/Grep/Read to find and examine the actual code that will change.
2. **Check for existing patterns** — Search for how similar problems are already solved in the codebase.
3. **Verify assumptions** — If the proposal mentions a table, class, or config, confirm it exists and check its current state.
4. **Check recent git history** — Look at recent commits to the affected files to understand momentum and context.

Do NOT speculate about code you haven't read. If you can't find a file or class mentioned in the proposal, flag it explicitly.
