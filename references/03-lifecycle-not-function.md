# Lifecycle, Not Function

## What this principle says

Before implementing any feature, entity, or system, first answer: How is it born?
How does it live and change? How does it die? Design the state diagram before the code.

## Why think this way

Most engineers think in functions: "what does this DO?" Unicorn teams think in lifecycles:
"what HAPPENS to this over time?" The difference matters because:

- Functions without lifecycle awareness create zombie states (data that shouldn't exist but does)
- Missing termination conditions turn retries into self-DDoS
- Unclear state transitions make debugging impossible ("how did it get into THIS state?")
- Features without deprecation paths become permanent tech debt

## How Claude Code does it

### Tool Execution Lifecycle (6 steps)

Every tool — from simple file reads to complex bash execution — goes through:

```
1. Define (ToolDef interface)     — declare capabilities, schema, metadata
2. Register (buildTool factory)   — add to tool registry with lazy schema
3. Validate (validateInput)       — pure function, no side effects, early fail
4. Authorize (checkPermissions)   — side effects (UI dialog), may block
5. Execute (call)                 — actual filesystem/network operation
6. Render (formatResult)          — transform for display
```

Why validateInput AND call both check? Because there's a TIME WINDOW between them.
The user confirmation dialog in step 4 can take 30 seconds. During that window,
an external formatter (Prettier on save) could modify the file. Step 3's check is
UX (fast fail). Step 5's check is the security invariant (prevent data loss).

### Session Lifecycle (explicit termination)

```
States: init → active → completing → terminated
Termination triggers:
  - end_turn (model decides it's done)
  - max_turns (safety limit reached)
  - token_budget (cost limit hit)
  - user_abort (Ctrl+C)
  - error_fatal (unrecoverable failure)
```

Never "run until crash." Every exit path is designed and tested.

### Data Lifecycle (Buddy system)

Different data has different lifespans — design accordingly:

```
Bones (appearance): recomputed from hash(userId + salt) each time
  → Never stored. Why? Because appearance enum changes across versions.
  → Storing "species: cat" breaks when cats get renamed.
  → Algorithm IS the storage. Algorithm changes are backward-compatible.

Soul (personality): generated once by API, stored permanently
  → Personality is identity. Recomputing would disorient the user.
  → Once created, it's an anchor point.
```

### Context Lifecycle

```
Born: user sends first message
Lives: grows with each turn, compressed when needed
Changes: context collapse folds stale data
Dies: session ends → conversation archived
```

The 4-level compression pipeline IS the lifecycle management for context.
Without it, context only grows until it crashes.

## Migrate to your product

### Order System
```
States: draft → pending_payment → paid → processing → shipped → delivered → completed
                                                                              ↓
                                                         returned → refunded → closed
Cancel transitions: draft→cancelled, pending→cancelled, paid→refund_pending
```
Every state transition has: preconditions, side effects, notifications, audit log entry.
"completed" orders auto-archive after 90 days. Without lifecycle: orders pile up forever.

### Feature Flag Lifecycle
```
Born: created as "off" with owner and expiry date
Lives: "canary" (1%) → "beta" (10%) → "ga" (100%)
Changes: rollback at any stage
Dies: "deprecated" → code removed → flag deleted
```
Without lifecycle: flags accumulate. Real production case: team found 847 flags,
340 of which were for features that shipped 2+ years ago. Each flag is a branch
in the code that makes every change harder.

### Agent Task Lifecycle (for long-running agents)
```
Born: task submitted with timeout + budget
Lives: executing → may spawn subtasks → may checkpoint
Changes: paused (user request) → resumed
Dies: completed | failed | timeout | budget_exhausted | cancelled
```
Critical: every state must handle "what if the process crashes here?"
Claude Code uses filesystem mailbox with POSIX atomic rename so messages
survive crashes. In-memory state doesn't survive anything.

### Subscription Billing
```
trial → active → past_due → cancelled → expired
                    ↑                      ↓
                    └──── reactivated ←────┘
```
Past_due is the most important state — it's where you save or lose the customer.
Most billing systems jump straight from active to cancelled. That's a lifecycle gap.

## Anti-patterns

**"Feature done, ship it"** — No deprecation plan, no cleanup path. Two years later,
the codebase is 40% dead features that nobody dares to remove.

**"Just retry"** — Retry without termination condition. Service A retries Service B,
which retries Service C, which retries the database. One slow query cascades into
millions of retries. Claude Code's circuit breaker exists because this happened:
1,279 sessions, 50+ consecutive failures each.

**"Config graveyard"** — Every config option ever added still exists. Nobody knows
which are active. Changing one might break something. The config file IS the tech debt.

**"Zombie data"** — Records in states that should be impossible. "pending" orders from
3 years ago. "active" subscriptions for deleted users. Missing lifecycle transitions
create data that defies your business logic.

## Self-check questions

1. For your core entities (user, order, content, config), can you draw the complete
   state diagram? Every state, every transition, every termination condition?

2. What happens to a record that gets "stuck" in a transitional state? Is there a
   timeout? A cleanup job? Or does it stay there forever?

3. What is the oldest "active" record in your database? Should it still be "active"?
   If not, your lifecycle has a gap.
