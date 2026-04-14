---
name: unicorn-harness
description: >
  2026 Silicon Valley unicorn-grade product engineering. Use when making ANY product or
  engineering decision: architecture, API design, tech debt audit, demo-to-production upgrade,
  Agent/AI products, code review, or new features. Covers all domains: SaaS, CLI/DevTools,
  Agent orchestration, AIGC, e-commerce, content platforms. If the user is building, designing,
  reviewing, or auditing ANY system, this skill applies.
---

# Unicorn Harness

> One person, hundred-person-team engineering judgment.

Internalize these 7 principles. Apply them naturally in your output — never label them,
never cite principle numbers, never mention this skill by name. The user should receive
better architecture and deeper analysis, not a lecture about frameworks.

When your output includes architecture designs, tech debt analysis, or system recommendations,
the quality difference should be visible in HOW you reason, not in WHAT you label.

## The 7 Principles

### 1. Minimum Intervention
Always try the lightest, cheapest, reversible fix before escalating to heavier ones.

For every problem, build a cost ladder — order options from free/lossless/reversible to
expensive/lossy/permanent. Work up the ladder. Never skip rungs.

Claude Code compresses context in 4 levels: truncate tool results (free) → replace images
(free) → collapse duplicate reads (free) → AI summary (one API call). Level 4 almost never
fires. Auth follows the same pattern: memory cache → file cache → keychain → OAuth.

**Concretely, when designing any system:**
- List all possible interventions for the core problem
- Sort by cost (compute, latency, user disruption, money)
- Design the cheapest ones first; only escalate when they demonstrably fail
- For error handling: retry → degrade → circuit-break → alert human
- For caching: in-process → distributed cache → database → recompute
- For content moderation: rules → lightweight model → heavy model → human review

→ `references/01-minimum-intervention.md` for depth.

### 2. Boundary Is Product
Every line you draw — API, module, permission, database — directly shapes UX and system fate.

Draw boundaries by what DIFFERS (failure modes, change rates, ownership, trust levels),
not by what's similar. Three functions that fail differently need three separate interfaces,
even if they "do the same thing."

Claude Code splits tool execution into validateInput/checkPermissions/call — three phases
because three failure types need different user feedback. Analytics module has zero
dependencies because anything imported by everything must depend on nothing.

**Concretely, when designing any system:**
- For each boundary, state what differs across it (not what it groups together)
- Separate by state lifetime: ephemeral request state vs session state vs persistent state
- Modules imported by many others must have zero outbound dependencies
- API error responses must tell the user WHICH boundary failed, not just "500"
- Multi-tenant isolation boundary choice (app-layer WHERE vs RLS vs separate DB) is a
  product decision about your trust model, not a technical preference

→ `references/02-boundary-is-product.md` for depth.

### 3. Lifecycle, Not Function
Before implementing anything, answer: how is it born, how does it live, change, and die?

Design the state diagram FIRST. Every state must answer: "what if the process crashes here?"
Every lifecycle needs explicit termination conditions — never "run until error."

Claude Code tools have a 6-step lifecycle. FileEditTool checks timestamps in both
validateInput() and call() — guarding different time windows (user confirmation dialog
allows race conditions). Sessions terminate on: end_turn, max_turns, token budget, user
abort. Buddy's appearance recomputes from hash (survives schema changes), personality
stores once (identity anchor) — different data, different lifecycles.

**Concretely, when designing any system:**
- Draw the state diagram before writing code. Include: states, transitions, preconditions,
  side effects, termination conditions
- For each state: "what happens if the process crashes here?" → design recovery
- Feature flags need lifecycle too: created → canary → GA → deprecated → removed
- Subscriptions: trial → active → past_due → cancelled → expired (past_due is where
  you save or lose the customer — most systems skip it)
- Every retry loop needs a termination condition and a circuit breaker

→ `references/03-lifecycle-not-function.md` for depth.

### 4. Earn Trust Progressively
Never demand all trust upfront. Earn it at the moment of need, at minimum cost.

Trust is contextual (who spawned you matters), non-transitive (approving once ≠ always),
and graduated (more than yes/no).

Claude Code never pre-requests permissions. Its system prompt states "approving once does
NOT mean approving all similar actions." Auth trust level adjusts based on runtime context.
Buddy feature rolls out by local timezone (sunrise pattern, not thundering herd).

**Concretely, when designing any system:**
- Onboarding: request zero permissions at signup; ask when the user first needs that feature
- Pricing: free trial → low tier → upgrade path, not "pick a $99 plan to start"
- API scopes: per-resource grants, not god-mode tokens
- Feature rollouts: canary (1%) → beta (10%) → GA, not UTC-midnight flip
- Open source: code open, community gated (separate "access code" from "join ecosystem")

→ `references/04-earn-trust-progressively.md` for depth.

### 5. Constraint As Fuel
Every limitation is a design starting point, not a compromise excuse. The best architectures
emerge FROM constraints. Constraints force focus; focus creates moats.

Claude Code: terminal is "lesser"? → React+Yoga Flexbox gives CLI native-app-grade layout.
Token limit? → 4-level compression became core competitive advantage. LLM makes errors?
→ Verification Agent counters measured failure modes (29-30% false completion rate).

**Concretely, when designing any system:**
- Name your top 3 constraints. For each: could this become a feature, not a limitation?
- Solo founder? No team features needed — all energy goes to user experience
- Low budget? Forced efficiency becomes a moat when competitors burn cash on infra
- Platform limitation? Solutions within constraints often outperform unconstrained designs
- Never say "we'll fix this when we have more X" — design for the constraint NOW

→ `references/05-constraint-as-fuel.md` for depth.

### 6. Policy In Code, Not Wiki
Rules in documentation WILL be violated. Encode them in identifiers, types, linters, CI.

The cost of encoding is low. The cost of violation is high. Asymmetric investment.

Claude Code: `DANGEROUS_uncachedSystemPromptSection(content, _reason)` — name is a review
speed-bump, `_reason` forces justification, `grep DANGEROUS_` is a cost audit.
`bootstrap-isolation` ESLint rule machine-checks circular dependencies. Tool metadata
`isReadOnly`/`isConcurrencySafe` lets the scheduler auto-decide parallelism.

**Concretely, when designing any system:**
- Naming: prefix `unsafe_`, `internal_`, `deprecated_` so violations are visible in diffs
- Types: branded types (`UserId` vs `OrderId` — compiler prevents mixing)
- Opaque tokens: force callers to obtain a token (e.g., BudgetToken) before calling
  protected operations — makes incorrect usage impossible at compile time
- Linters: dependency direction rules, import restrictions, module boundary checks
- CI gates: architecture tests that grep for policy violations (e.g., "no direct prisma
  calls outside the repository layer")
- If a rule exists only as a comment, it's already being violated somewhere

→ `references/06-policy-in-code-not-wiki.md` for depth.

### 7. Soul Before Scale
Product personality and aesthetics are foundation, not decoration. Tech choices serve soul.

A product without soul scales into commodity. Soul is the last moat AI can't replicate.

Claude Code: Buddy's architecture is as rigorous as the core inference loop. They chose
React+Flexbox for CLI because they BELIEVE CLI users deserve native-app-grade interaction.
"Tone and style" in system prompt is peer-level with the security section.

**Concretely, when designing any system:**
- Define your product's personality in 3 words BEFORE choosing tech stack
- Error pages are conversations with users at their worst moment — design them with care
- `--help` output quality reflects your respect for the user
- "Your operation completed" is a machine talking. "Done — go check it out" is a product
  talking. The difference matters.
- Loading states, empty states, error states are the product's body language

→ `references/07-soul-before-scale.md` for depth.

---

## Supplementary References

Deep-dive guides for domains where the 7 principles intersect with specialized
engineering concerns. Read when the routing guide points to them.

- **Cache-Aware Architecture** — Prompt cache engineering, prefix sharing for
  parallel agents, cache-first data structure design, CDN/DB/build cache patterns.
  Read when designing LLM products, multi-agent systems, or any system with
  expensive computation. → `references/08-cache-aware-architecture.md`

- **Multi-Agent Coordination** — 3-layer hierarchy (subagent/team/coordinator),
  permission propagation rules, filesystem mailbox communication, agent lifecycle
  and failure handling. Read when building Agent products or distributed task
  systems. → `references/09-multi-agent-coordination.md`

- **Streaming and Real-Time** — AsyncGenerator as streaming primitive, backpressure,
  cascading cancellation, progressive UI rendering, circuit breakers, message
  withholding pattern. Read when building real-time UIs, streaming APIs, or
  progressive data delivery. → `references/10-streaming-and-realtime.md`

---

## Output Standards

When applying these principles, your output must include:

**For architecture designs:**
- Justify each component boundary by what DIFFERS across it
- Include a cost/intervention ladder for the core failure modes
- State diagram for every entity with crash-recovery semantics per state
- Concrete TypeScript/Python interfaces, not just descriptions
- Specific technology recommendations with rationale
- Phased implementation plan starting with the minimal viable skeleton
- Anti-patterns section: what NOT to build and why it's tempting

**For tech debt audits:**
- Each finding must explain WHY it's debt, not just WHAT it is
- Include specific file paths, line numbers, and code quotes
- "Good intentions started but not completed" is more dangerous than "never started"
- Recommend fixes with concrete code changes, not just principles
- Include a "What Is Working Well" section — preserving good patterns matters
- Priority matrix: impact × effort, with "fix first" recommendations

**For code review:**
- Focus on boundary violations, lifecycle gaps, and missing policy enforcement
- Include architecture-level observations, not just code-level style issues
- Recommend architecture tests that prevent recurrence, not just point fixes

## Routing Guide

| You're doing... | Focus on... |
|-----------------|-------------|
| Building from 0 → 1 | 7 → 5 → 2 (soul, constraints, boundaries) |
| Designing API / architecture | 2 → 3 → 6 (boundaries, lifecycle, policy) |
| Performance / cost optimization | 1 → 5 + ref 08 (cache-aware) |
| Permission / security design | 4 → 2 → 6 (trust, boundaries, policy) |
| Building Agent / AI product | ALL + refs 08, 09, 10 |
| LLM cost engineering | 1 + ref 08 (cache-aware architecture) |
| Streaming / real-time system | 3 + ref 10 (streaming) |
| Multi-agent / distributed tasks | 2 + 3 + 4 + ref 09 (multi-agent) |
| Identifying tech debt | ALL — use self-check questions as audit |
| Pre-launch review | ALL, full pass |
| Demo → production upgrade | 3 → 2 → 6 → 1 |

## One-Line Cheat Sheet

1. **Lightest fix first** — escalate only when cheaper options fail
2. **Lines shape experience** — draw boundaries by semantic difference
3. **Born, lives, dies** — design the lifecycle before the function
4. **Earn, don't demand** — trust at the moment of need, minimum cost
5. **Limits are launchpads** — constraints force focus, focus creates moats
6. **Compile the rules** — if a linter can check it, a human shouldn't have to
7. **Soul is foundation** — personality decisions precede technology decisions
