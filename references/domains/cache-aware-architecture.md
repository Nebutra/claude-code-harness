# Cache-Aware Architecture

> Supplementary reference — extends Principle 1 (Minimum Intervention) and
> Principle 5 (Constraint As Fuel) into the domain of LLM cost engineering
> and general cache-first system design.

## What this reference covers

In any system where computation is expensive — LLM inference, database queries,
API calls, rendering — cache design is not an optimization afterthought. It is
an architectural decision that pervades every layer. This reference covers how
to design systems where cache-awareness is a first-class concern from day one.

## Why this matters

Token cost is the fundamental unit of AI product economics. Every architectural
decision in an LLM product is secretly a token budget decision. But the same
thinking applies to any system with expensive computation: CDN cache design,
database query planning, build systems, rendering pipelines.

The insight: **cache topology should drive data structure design, not the other
way around**. If you design your data structures first and add caching later,
you'll fight the cache at every turn. If you design your data structures to be
cache-friendly from the start, caching becomes nearly free.

## How Claude Code does it

### System Prompt as a Memoization Structure

The system prompt is not a string — it is a `string[]` array where each element
is an independent cache unit with its own `cache_control` settings.

```
┌──────────────────────────────────────────────┐
│ Static region (scope: 'global')               │
│ - Role definition                            │
│ - Tool usage priorities                       │
│ - Doing Tasks guidelines                      │
│ - Actions authorization rules                 │
│ Shared across ALL users worldwide             │
│ Cached once, amortized over millions of calls │
├──────────────────────────────────────────────┤
│ SYSTEM_PROMPT_DYNAMIC_BOUNDARY sentinel       │
├──────────────────────────────────────────────┤
│ Dynamic region (per-session, no cache)        │
│ - CLAUDE.md content                          │
│ - Git status                                 │
│ - MCP server instructions                    │
│ - Memory prompts                             │
│ Different for every user/session              │
└──────────────────────────────────────────────┘
```

The `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` sentinel is the architectural split point.
Everything above it is immutable and globally cached. Everything below is
per-session and always recomputed. The sentinel is not organizational
convenience — it is a token economy mechanism. A 150K+ token static prompt
shared across millions of requests means the per-request cost of the static
portion approaches zero.

### Fork Prefix Sharing

When spawning parallel sub-agents, Claude Code's Fork mechanism engineers
message structure for maximum cache prefix overlap:

```
Parent conversation:           [system] [msg1] [msg2] ... [msgN]
                                        ↓ shared prefix
Fork child A: [system] [msg1] [msg2] ... [msgN] [ack_placeholder] [task_A]
Fork child B: [system] [msg1] [msg2] ... [msgN] [ack_placeholder] [task_B]
Fork child C: [system] [msg1] [msg2] ... [msgN] [ack_placeholder] [task_C]
```

All children share an identical message prefix down to the last byte. Only the
final instruction block (`task_A/B/C`) differs. The shared prefix is a cache
hit. The unique instruction is the only part paying full price.

The `ack_placeholder` between parent messages and child instructions is a
synthetic `tool_result` that keeps the message structure valid while preserving
the shared prefix. Without it, each child's first unique message would start
at a different position, breaking the prefix.

Result: near-100% cache hit rate across parallel agents. The team reports
saving ~5-15 Gtok/week by eliminating CLAUDE.md and gitStatus from the
Explore Agent's context.

### Tool Ordering for Cache Stability

Tools are ordered: built-in tools first, MCP tools second. This is not
alphabetical or by importance — it's cache-aware. Built-in tools are stable
across sessions. MCP tools change when the user configures new servers. By
placing the volatile tools at the END of the tool list, the stable prefix
(built-in tools) remains cacheable even when MCP tools change.

### DANGEROUS_uncachedSystemPromptSection

When a prompt section MUST be uncached (e.g., real-time instructions that
change every call), the function name forces visibility:

```typescript
DANGEROUS_uncachedSystemPromptSection(content, _reason)
```

"Dangerous" because every uncached section increases per-request cost. The
`_reason` parameter forces the author to justify the cost. `grep DANGEROUS_`
across the codebase is a cost audit.

### LazySchema for Tool Definitions

Tool schemas consume input tokens. 40+ tools × schema tokens = significant
overhead per call. `lazySchema()` wraps schemas in TypeScript getters so they
are only materialized when a tool is actually used. Zero startup cost, zero
token cost for unused tools.

## Migrate to your product

### LLM Product — Prompt Engineering as Cache Engineering
```
Design prompt structure for cache:
1. Identify which parts of your prompt are STABLE (instructions, examples,
   persona) and which are DYNAMIC (user context, conversation history)
2. Place stable parts FIRST — they become the cache prefix
3. Place dynamic parts LAST — they're the only per-request cost
4. Use cache_control breakpoints between stable and dynamic regions
5. Measure cache hit rate as a first-class metric

Cost formula:
  cache_write = 1.25× input token price (first time only)
  cache_read = 0.10× input token price (every subsequent hit)
  uncached = 1.00× input token price (every time)

A 100K token system prompt that hits cache = 10K token effective cost.
The same prompt without caching = 100K token cost per request.
```

### Multi-Agent / Fork Architecture
```
When spawning N parallel sub-tasks:
1. Structure messages so all children share the longest possible prefix
2. Put the task-specific instruction LAST (only unique part per child)
3. Use synthetic placeholder messages to maintain structural validity
4. Measure: total tokens across N children / tokens for 1 child
   → ratio near 1.0 means your prefix sharing is working
```

### CDN / Static Asset Caching
```
Same principle: stable content first, dynamic content last.

1. Separate static assets (CSS, JS bundles) from dynamic content (API responses)
2. Use content-hash filenames for static assets → infinite cache TTL
3. Use short TTLs only for truly dynamic content
4. Measure cache hit ratio per asset type — any ratio below 90% for
   static assets means your caching strategy has a leak
```

### Database Query Caching
```
Intervention cost ladder for data access:
L1: In-process cache (Map/LRU) → 0ms, free, process-local
L2: Distributed cache (Redis) → 1ms, cheap, shared
L3: Materialized view → 10ms, pre-computed, stale-while-revalidate
L4: Full query → 50-500ms, always fresh, expensive

Design queries so L1 handles the hot path:
- User's own profile? L1 (changes rarely, accessed constantly)
- Feed content? L2 (shared across users, invalidated on publish)
- Analytics aggregate? L3 (expensive to compute, tolerance for staleness)
- Search results? L4 (too dynamic to cache effectively)
```

### Build System Caching (Turbo/Nx)
```
Same principle: stable inputs produce cacheable outputs.

1. Task graph with content-hash inputs → cache hit if inputs unchanged
2. Remote cache (shared across team) → O(1) rebuild for unchanged packages
3. Dependency topology determines invalidation cascade
4. Measure: % of tasks that hit cache on a typical PR
   → below 70% means your task boundaries are too coarse
```

## Anti-patterns

**"Cache everything"** — Caching dynamic, user-specific data at a global level
means serving stale data to the wrong user. Cache STABLE things globally;
cache DYNAMIC things per-session or not at all.

**"Add caching later"** — If your data structures aren't cache-friendly, adding
a cache layer on top creates cache invalidation nightmares. Design for cache
from the start: stable prefix + dynamic suffix.

**"One big prompt"** — A single concatenated string can't have per-section cache
control. Array-of-sections with individual cache headers is the correct shape.

**"Ignore cache hit rate"** — If you're not measuring cache hits per section,
you're spending money you don't need to spend. Add metrics from day one.

## Self-check questions

1. Can you draw your system's data flow and mark which parts are stable (cacheable)
   vs. dynamic (per-request)? Is the stable part placed first in each request?

2. When you spawn parallel sub-tasks (workers, agents, batch jobs), do they share
   a common prefix? Or does each one redundantly include the full context?

3. What is your cache hit rate for the most expensive operation in your system?
   If you don't know, you're not measuring the right thing.
