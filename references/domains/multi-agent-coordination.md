# Multi-Agent Coordination

> Supplementary reference — extends Principle 2 (Boundary Is Product),
> Principle 3 (Lifecycle, Not Function), and Principle 4 (Earn Trust
> Progressively) into the domain of multi-agent system design.

## What this reference covers

How to design systems where multiple autonomous agents (LLM-powered or otherwise)
collaborate on tasks. Covers: hierarchy design, communication patterns, permission
propagation, resource isolation, and failure handling across agent boundaries.

## Why this matters

Single-agent systems hit a ceiling: one context window, one tool at a time, one
line of reasoning. Multi-agent systems break through this ceiling but introduce
coordination complexity that can be worse than the ceiling itself.

The insight: **agent coordination is a system design problem, not an AI problem**.
The same patterns that work for microservices, distributed systems, and team
management apply to multi-agent systems — with the added constraint that each
agent has bounded context and probabilistic behavior.

## How Claude Code does it

### Three-Layer Hierarchy (pick the lightest that works)

```
Layer 1: Subagent (Fork)
  - One-shot, disposable
  - Shares parent context via cache prefix
  - No communication between siblings
  - No persistent state
  - Cost: cheapest (cache hits)
  - Use when: parallel independent tasks (search, analysis, exploration)

Layer 2: Team / Swarm
  - Persistent, communicating
  - Each agent has own context and tools
  - Communication via filesystem mailbox (POSIX atomic rename)
  - Shared permission context with Leader Permission Bridge
  - Cost: moderate (separate contexts, separate API calls)
  - Use when: tasks need back-and-forth (code review + fix cycle)

Layer 3: Coordinator
  - Pure orchestrator, no direct execution
  - Only ~6 tools: TeamCreate/TeamDelete/SendMessage/Agent/TaskStop/SyntheticOutput
  - Workers cannot see each other's results (isolation)
  - Coordinator aggregates and synthesizes
  - Cost: highest (coordinator + N workers, no cache sharing)
  - Use when: tasks require synthesis across isolated workers
```

This is Principle 1 (Minimum Intervention) applied to coordination: always use
the lightest coordination pattern that solves the problem. Don't use a Coordinator
when Fork suffices. Don't use Team when Subagent suffices.

### Why Coordinator and Fork are Mutually Exclusive

Fork shares the parent's full conversation history with all children. Coordinator's
safety model requires that workers cannot see each other's results or the
coordinator's aggregation state. A Forked Worker would see every other worker's
output in the shared history — breaking isolation, wasting tokens, and potentially
leaking credentials from one worker's output into another's context.

Two patterns that look composable are architecturally incompatible because they
have **opposite assumptions about context visibility**. This is Principle 2:
the boundary between Fork and Coordinator exists because they differ in their
trust model.

### Permission Propagation (6 Rules)

Agent systems create permission escalation risks. Claude Code enforces 6 rules:

```
Rule 1: Child inherits parent's permission SCOPE, not specific approvals
  → Parent approved file-read for /src → child can read /src
  → Parent approved file-write for /src/a.ts → child CANNOT write /src/b.ts

Rule 2: Permission approvals are NON-TRANSITIVE across sessions
  → Approving in session A doesn't auto-approve in session B

Rule 3: Managed context overrides local config
  → Running as subprocess of Desktop → ignore ~/.claude/settings.json
  → allowedSettingSources can restrict to CLI flags only

Rule 4: Workers in Coordinator mode get MINIMAL permissions
  → Only the tools specified in the worker's definition
  → Cannot escalate by requesting tools not in their profile

Rule 5: Leader Permission Bridge for async escalation
  → Worker hits permission wall → writes request to mailbox
  → Leader polls mailbox → can approve/deny/delegate to human
  → Worker resumes with result → no blocking the entire team

Rule 6: Permission scope is explicit, never inferred
  → Model cannot decide it has permission based on conversation context
  → Every permission check goes through the actual policy at runtime
```

### Communication via Filesystem Mailbox

Agents running as separate processes cannot use in-memory queues. Claude Code
uses filesystem-based mailboxes with POSIX atomic rename:

```
1. Writer creates message in temporary file: /mailbox/.tmp-{uuid}
2. Writer atomically renames: /mailbox/.tmp-{uuid} → /mailbox/{timestamp}-{uuid}
3. Reader polls /mailbox/ directory for new files
4. Reader processes and deletes the file
```

Atomic rename guarantees: no partial reads, no duplicate processing (file either
exists or doesn't), crash-safe (incomplete writes stay in .tmp and get cleaned up).

This is simpler and more reliable than Redis pub/sub for small-scale inter-process
communication. It survives process crashes because the filesystem is the persistence
layer.

### Agent Definition Priority Chain

Multiple sources can define agent behavior. Claude Code resolves conflicts with
a clear priority chain:

```
1. PluginAgentDefinition (highest — user-installed plugins)
2. CustomAgentDefinition (user's ~/.claude/agents/ directory)
3. BuiltInAgentDefinition (lowest — shipped with the product)
```

This is Principle 4 (Earn Trust): the user's own definitions always win.
The product's built-in agents are defaults, not mandates.

## Migrate to your product

### Agent Task System (for AI products)
```
Design your agent task infrastructure around the 3-layer model:

L1 — Fire-and-forget workers (cheapest):
  Use for: parallel web searches, file analysis, data extraction
  Pattern: spawn N workers with shared context prefix, collect results
  Communication: none (results returned to parent)
  Failure: individual worker failure doesn't affect siblings

L2 — Persistent collaborators (moderate):
  Use for: code review + fix cycles, research + writing, multi-step workflows
  Pattern: agents with own context, communicate via durable message queue
  Communication: structured messages through persistent channel
  Failure: agent failure triggers retry or reassignment

L3 — Orchestrated pipeline (most expensive):
  Use for: complex synthesis, multi-perspective analysis, quality assurance
  Pattern: coordinator decomposes task, assigns to isolated workers, aggregates
  Communication: coordinator mediates all communication
  Failure: coordinator can re-plan or substitute workers
```

### Microservice Coordination
```
Same hierarchy maps to microservices:

L1 — Scatter-gather (parallel independent calls):
  API gateway fans out to 5 services, merges responses
  No inter-service communication needed

L2 — Saga pattern (sequential with compensation):
  Order service → Payment → Inventory → Shipping
  Each step communicates result to next
  Failure triggers reverse compensation

L3 — Orchestrator (centralized coordination):
  Workflow engine decomposes complex process
  Workers are stateless executors
  Orchestrator owns all state and retry logic
```

### Permission Propagation for Multi-Tenant Systems
```
Map the 6 rules to your authorization model:

Rule 1 → Scope inheritance: child process inherits parent's tenant context
Rule 2 → Non-transitive sessions: API keys are per-session, not permanent
Rule 3 → Context-aware override: CI/CD runs use service accounts, not user creds
Rule 4 → Minimal worker permissions: background jobs get only the scopes they need
Rule 5 → Async escalation: when a job needs elevated permissions, queue for human
Rule 6 → Runtime policy check: never cache permission decisions across requests
```

### Inter-Process Communication Patterns
```
Choose by reliability needs:

Filesystem mailbox (simplest, crash-safe):
  Use for: local agents, CLI tools, development environments
  Pro: zero infrastructure, survives crashes
  Con: polling latency, doesn't scale beyond single machine

Redis Streams (moderate, scalable):
  Use for: multi-machine agents, moderate throughput
  Pro: consumer groups, acknowledgment, replay
  Con: requires Redis, potential data loss on Redis crash

Durable queue (Inngest, SQS, Kafka):
  Use for: production agent systems, guaranteed delivery
  Pro: persistence, retry, dead-letter, observability
  Con: operational complexity, latency overhead
```

## Anti-patterns

**"Every agent talks to every agent"** — Full mesh communication between N agents
means N² channels. Each channel is a potential failure point and debugging surface.
Use a hub-and-spoke pattern (coordinator) or no communication at all (fork).

**"Shared mutable state"** — Agents sharing an in-memory object or database row
without isolation creates race conditions. Use message passing or immutable
snapshots. Claude Code's filesystem mailbox is write-once: messages are never
modified, only created and consumed.

**"Implicit permission escalation"** — Agent A approved for file-read spawns
Agent B which decides it also needs file-write. Without explicit permission
propagation rules, this escalation is invisible. Every permission grant must
be explicit and scoped.

**"Single giant agent"** — The temptation to stuff everything into one agent
with a massive context window. This works until: the context fills up, the
task is too complex for one reasoning chain, or you need parallelism. Design
for multi-agent from the start; a single agent is just L1 with N=1.

**"Re-inventing distributed systems"** — Agent coordination IS distributed
systems. The problems (consensus, failure detection, message ordering, exactly-
once delivery) are the same. Use proven patterns (saga, outbox, idempotency
keys) rather than inventing new ones.

## Self-check questions

1. When your system needs to do N things in parallel, what coordination pattern
   do you use? Is it the lightest one that works, or the most powerful one
   you know?

2. If Agent A spawns Agent B, what permissions does B inherit? Is this explicit
   in code, or implicit by "it just works"? Can B escalate beyond A's scope?

3. What happens when an agent crashes mid-task? Is the work lost? Is it
   retried? Is there a checkpoint to resume from? If you don't know, your
   agent lifecycle is incomplete.
