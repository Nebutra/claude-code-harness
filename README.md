# Unicorn Harness

**One person, hundred-person-team engineering judgment.**

7 principles extracted from production-grade systems (Claude Code v2.1.88) and generalized into a thinking framework that applies to any product domain — SaaS, CLI/DevTools, Agent orchestration, AIGC, e-commerce, content platforms, and beyond.

> "We don't just incubate unicorns — we build the engine that continuously fissions new ones."
> — Nebutra Manifesto

## What This Is

A [Claude Code SKILL](https://docs.anthropic.com/en/docs/claude-code) that upgrades engineering decision-making. Not a tutorial. Not a checklist. A thinking operating system.

The gap between "can write code" and "can ship a unicorn-grade product" isn't technical — it's cognitive. This SKILL closes that gap by encoding the thinking patterns behind $1B products into 7 actionable principles.

## The 7 Principles

| # | Principle | One-liner |
|---|-----------|-----------|
| 1 | **Minimum Intervention** | Lightest fix first — escalate only when cheaper options fail |
| 2 | **Boundary Is Product** | Every line you draw shapes user experience |
| 3 | **Lifecycle, Not Function** | Design how things are born, live, and die — before writing code |
| 4 | **Earn Trust Progressively** | Never demand all trust upfront |
| 5 | **Constraint As Fuel** | Limitations are design starting points, not excuses |
| 6 | **Policy In Code, Not Wiki** | Rules in wikis get broken; rules in compilers don't |
| 7 | **Soul Before Scale** | Product personality is foundation, not decoration |

## Quick Start

### As a Claude Code SKILL

```bash
# Add as a git submodule to your project
git submodule add https://github.com/user/unicorn-harness.git .claude/skills/unicorn-harness

# Or copy the SKILL.md into your skills directory
cp unicorn-harness/SKILL.md ~/.claude/skills/unicorn-harness/SKILL.md
```

### Just Read the Principles

Each principle has a deep-dive reference document:

- [01 — Minimum Intervention](references/01-minimum-intervention.md)
- [02 — Boundary Is Product](references/02-boundary-is-product.md)
- [03 — Lifecycle, Not Function](references/03-lifecycle-not-function.md)
- [04 — Earn Trust Progressively](references/04-earn-trust-progressively.md)
- [05 — Constraint As Fuel](references/05-constraint-as-fuel.md)
- [06 — Policy In Code, Not Wiki](references/06-policy-in-code-not-wiki.md)
- [07 — Soul Before Scale](references/07-soul-before-scale.md)

Each reference covers: what the principle says, why it matters, how Claude Code implements it, how to migrate it to YOUR product, anti-patterns, and self-check questions.

**Supplementary deep-dives** (for specialized domains):

- [08 — Cache-Aware Architecture](references/08-cache-aware-architecture.md) — Prompt cache engineering, prefix sharing, cache-first design
- [09 — Multi-Agent Coordination](references/09-multi-agent-coordination.md) — 3-layer hierarchy, permission propagation, communication patterns
- [10 — Streaming and Real-Time](references/10-streaming-and-realtime.md) — AsyncGenerator, backpressure, progressive rendering

## When to Use

| You're doing... | Focus on... |
|-----------------|-------------|
| Building from 0 to 1 | Soul, Constraint, Boundary |
| Designing API / architecture | Boundary, Lifecycle, Policy |
| Performance / cost optimization | Minimum Intervention, Constraint |
| Permission / security design | Trust, Boundary, Policy |
| Building Agent / AI products | All, especially Intervention + Lifecycle |
| Identifying tech debt | All — use self-check questions as audit |
| Pre-launch review | Full pass through all 7 |

## TDD-Validated

This SKILL was tested with real prompts (Agent architecture design, production codebase tech debt audit) in A/B comparison: with SKILL vs. without SKILL vs. v1 SKILL vs. v2 SKILL.

Results:
- Architecture designs show deeper boundary reasoning and intervention cost ladders
- Tech debt audits find more actionable bugs (HMAC mismatches, hardcoded limits, mock code in production paths)
- Principles are internalized, not parroted — no "Principle X applied" labels in output
- Token efficiency improves with each iteration (52 tool calls vs 113 for the same audit)

## Part of the Nebutra Ecosystem

```
Nebutra Manifesto (philosophy)    → "Infinite fission, not infinite expansion"
Unicorn Harness (cognition)       → How to think     ← this repo
Nebutra Sailor (infrastructure)   → 53 modules built on these principles
Sleptons (community)              → OPC ecosystem powered by this thinking
```

## Contributing

Migration examples from new domains are welcome. Each principle's reference file has a "Migrate to your product" section — if your domain isn't covered, submit a PR.

## License

MIT — thinking frameworks should spread without friction.
