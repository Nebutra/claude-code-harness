# Minimum Intervention

## What this principle says

When you face a problem, always start with the lightest, cheapest, most reversible
solution. Only escalate to heavier interventions when lighter ones demonstrably fail.
This applies to EVERY domain: caching, error handling, UX flows, cost management,
feature complexity, team processes.

## Why think this way

Most engineers jump to "the right solution" — which is usually the most complete,
most powerful, most expensive one. This wastes resources on problems that didn't need
them, introduces complexity that creates new problems, and removes options for the future.

The deeper insight: **intervention cost ordering is an architectural decision**, not an
optimization afterthought. Design your system with explicit escalation ladders from day one.

## How Claude Code does it

### Context Compression Pipeline (4 levels)

The core problem: conversation context exceeds token limits.

```
Level 1: Snip — truncate oversized tool results
  Cost: zero (string operation)
  Loss: partial (tail of long outputs)
  When: any tool result > threshold

Level 2: Microcompact — replace images with text placeholders
  Cost: zero (substitution)
  Loss: minimal (images rarely re-needed verbatim)
  When: after snip, still over budget

Level 3: Context Collapse — deduplicate repeated file reads
  Cost: zero (dedup operation)
  Loss: near-zero (identical content folded)
  When: after microcompact, still over budget

Level 4: AutoCompact — AI summarizes the full conversation
  Cost: one full API call (expensive)
  Loss: significant (precise code lines, exact errors lost)
  When: only after 1-3 all fail
```

Key insight: Level 4 almost never fires in practice because 1-3 handle most cases.
The engineering investment in three "cheap" levels saves enormous cost at scale.

### Authentication Chain (7 levels)

```
1. In-memory cache (free, instant)
2. File-based token cache (cheap, fast)
3. Keychain/credential store (cheap, slightly slower)
4. Environment variable (free, static)
5. OAuth device flow (user interaction required)
6. API key prompt (user interaction required)
7. Managed OAuth (external system integration)
```

Each level tried in order. Most sessions never reach level 5.

### Permission Requests

```
1. Check in-memory permission cache → already approved?
2. Check file-based permission rules → matches allowlist?
3. Ask user → minimal, specific permission request
```

Never asks for permissions it doesn't need. Never asks again for something already approved
in this session.

## Migrate to your product

### Caching System
```
L1: In-process memory (Map/LRU) → ~0ms, free
L2: Redis/Memcached → ~1ms, cheap
L3: Database query → ~10ms, moderate
L4: Full recomputation → variable, expensive
```
Don't set `Cache-Control: no-cache` because one edge case needed fresh data.
Handle that edge case at L1, let everything else hit cache.

### Error Handling
```
L1: Retry with backoff (automatic, invisible to user)
L2: Fallback to degraded mode (partial functionality)
L3: Circuit breaker (stop attempting, show status)
L4: Alert human operator (last resort)
```
Claude Code data: without circuit breaker, 1,279 sessions generated 50+ consecutive
failures, wasting 250K API calls/day. The circuit breaker is L3 saving L4.

### Content Moderation
```
L1: Rule-based filter (regex, blocklist) → free, instant
L2: Lightweight classifier → cheap, fast
L3: Full LLM evaluation → expensive, slow
L4: Human review queue → most expensive, slowest
```
Run 100% through L1, only send L1-passes to L2, etc. Most content never reaches L3.

### Feature Complexity
```
L1: Can we solve this with existing features + documentation?
L2: Can we solve this with a config option?
L3: Can we solve this with a small feature addition?
L4: Do we need a new subsystem?
```
Most "feature requests" are L1 (users didn't know it existed) or L2 (one toggle).

## Anti-patterns

**"Redis for everything"** — Team puts all data in Redis because "it's fast." When Redis
goes down, the entire system goes down. L1 (in-memory for hot data) + L3 (database as
source of truth) would have been more resilient AND faster for the common case.

**"Alert on every error"** — Team alerts humans for every 500 error. Alert fatigue sets in
within a week. 90% of those errors were transient and self-recovered. L1 (auto-retry) would
have silenced the noise, letting humans focus on the 10% that matters.

**"Full solution on day one"** — Team spends 3 months building a comprehensive solution for
a problem that a 10-line script could have handled for the next 6 months. By month 3,
requirements changed anyway.

## Self-check questions

1. Can you draw your system's "intervention cost ladder" for its top 3 problems?
   Does each rung have a clear cost (time/money/UX impact)?

2. For your most common problem, is it being solved by your cheapest intervention?
   Or are you using a $10 tool for a $0.01 problem?

3. When was the last time your system escalated to the most expensive intervention?
   If it happens often, your cheaper rungs aren't working. If it never happens,
   you might not need it at all.
