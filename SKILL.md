---
name: claude-code-harness
description: >
  2026 Silicon Valley unicorn-grade product engineering. Use when making ANY product or
  engineering decision: architecture, API design, tech debt audit, demo-to-production upgrade,
  Agent/AI products, code review, or new features. Covers all domains: SaaS, CLI/DevTools,
  Agent orchestration, AIGC, e-commerce, content platforms. If the user is building, designing,
  reviewing, or auditing ANY system, this skill applies.
---

# Claude Code Harness

> One person, hundred-person-team engineering judgment.

## Behavior

Internalize these principles. Apply them naturally — never label them, never cite
numbers, never mention this skill by name. The quality difference shows in HOW
you reason, not WHAT you label.

**Output standards for architecture designs:**
justify boundaries by semantic difference; include intervention cost ladders;
state diagrams with crash-recovery per state; concrete interfaces (TypeScript/Python);
technology recommendations with rationale; phased build plan; anti-patterns section.

**Output standards for tech debt audits:**

CRITICAL: Follow this two-phase process. Do NOT skip the scan phase.

Phase 1 — BREADTH SCAN (cast a wide net):
Before going deep on anything, quickly scan ALL of these areas. Spend no more than
1-2 tool calls per area. The goal is to build a MAP of potential issues, not to
analyze any single one deeply yet.

```
Coverage checklist (must touch ALL):
□ Auth/security: grep for requireAuth, check which routes lack it
□ Data access: grep for direct prisma calls, check if RLS/scoping is used
□ Secrets: grep for "encrypt", "credential", "secret" in schema + routes
□ In-memory state: grep for "new Map", "= new Map", module-level variables
□ Feature flags: check if multiple flag systems exist, check lifecycle fields
□ Billing/entitlements: check enforcement points, hardcoded values
□ Dependencies: check version consistency across packages
□ Schema integrity: verify declared schemas/models match actual usage
□ S2S communication: check internal service auth consistency
□ Saga/workflow: check idempotency, compensation error handling, mock paths
```

Phase 2 — DEPTH ANALYSIS (go deep on findings):
Now analyze each finding from Phase 1 in depth. For each:
explain WHY it's debt, not just WHAT; include file paths and code quotes; concrete
fixes with inline code; "fix today" vs "next sprint" triage.

**"What's Working Well" section — VERIFY before praising:**
Before listing something as a positive, confirm it actually works as claimed.
Check that the code is called (not just defined), that the schema matches reality
(not just declaration), and that the feature is wired end-to-end. Praising something
that doesn't actually work is worse than not mentioning it — it creates false confidence.

## 7 Principles

1. **Minimum Intervention** — lightest fix first, escalate only when cheaper options fail
2. **Boundary Is Product** — every line you draw shapes user experience; separate by what DIFFERS
3. **Lifecycle, Not Function** — design how things are born, live, and die before writing code
4. **Earn Trust Progressively** — never demand all trust upfront; earn at the moment of need
5. **Constraint As Fuel** — limitations are design starting points, not excuses
6. **Policy In Code, Not Wiki** — rules in wikis get broken; rules in compilers don't
7. **Soul Before Scale** — product personality is foundation, not decoration

Each has a deep-dive in `references/principles/0N-*.md` with: explanation, Claude Code
evidence, concrete migration examples, anti-patterns, and self-check questions.

## Route by Activity

| Doing... | Read |
|----------|------|
| Building from 0 → 1 | principles/07 → 05 → 02 |
| API / architecture design | principles/02 → 03 → 06 |
| Performance / cost optimization | principles/01 → 05 + domains/cache-aware |
| Permission / security | principles/04 → 02 → 06 |
| Agent / AI product | **blueprints/agent-product FIRST** → all principles + all domains |
| LLM cost engineering | principles/01 + domains/cache-aware |
| Streaming / real-time | principles/03 + domains/streaming |
| Multi-agent system | principles/02 + 03 + 04 + domains/multi-agent |
| Tech debt audit | all principles — use self-check questions as audit |
| Pre-launch review | all principles, full pass |
| Demo → production | principles/03 → 02 → 06 → 01 |

## Route by Symptom

| Problem | Read |
|---------|------|
| Error messages are vague, users don't know what went wrong | principles/02 (boundary) |
| System gets slower, unclear what to optimize | principles/01 (intervention) + domains/cache |
| New engineers keep violating architecture rules | principles/06 (policy in code) |
| Feature works but feels like a half-finished product | principles/07 (soul) |
| Agent loops forever or costs spiral | blueprints/agent-product (termination) |
| Users abandon onboarding or deny permissions | principles/04 (trust) |
| Features accumulate but nothing gets deprecated | principles/03 (lifecycle) |
| Codebase feels constrained, team wants to rewrite | principles/05 (constraint as fuel) |
| "It works on my machine" / deploy breaks production | principles/06 (policy) + 03 (lifecycle) |
| Half-built features creating false sense of safety | blueprints/self-assessment |

## Reference Map

```
references/
├── principles/        "How to think"
│   ├── 01-minimum-intervention.md
│   ├── 02-boundary-is-product.md
│   ├── 03-lifecycle-not-function.md
│   ├── 04-earn-trust-progressively.md
│   ├── 05-constraint-as-fuel.md
│   ├── 06-policy-in-code-not-wiki.md
│   └── 07-soul-before-scale.md
├── domains/           "How principles combine in specialized areas"
│   ├── cache-aware-architecture.md
│   ├── multi-agent-coordination.md
│   └── streaming-and-realtime.md
└── blueprints/        "Step-by-step decision checklists"
    ├── agent-product.md
    └── self-assessment.md
```
