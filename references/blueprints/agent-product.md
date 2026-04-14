# Agent Product Blueprint

> This is not an abstract principle — it is a concrete architecture decision checklist
> extracted from Claude Code, a production Agent that handles millions of sessions.
> Follow this blueprint to build an Agent that works, not a chatbot that calls tools.

## The 3 Things That Make an Agent an Agent

A chatbot calls an LLM once and returns the response. An Agent has exactly three
properties that a chatbot doesn't. Remove any one and you have a chatbot again:

```
1. The Loop     — while(true) { callModel → collectToolUse → executeTools → injectResults }
2. The Memory   — cross-turn message history persists between loop iterations
3. The Feedback — tool_result messages are injected back so the model sees what happened
```

These are non-negotiable. Everything else — UI, auth, multi-agent, plugins — is
layered on top. If your "Agent" doesn't have all three, it's a single LLM call
with extra steps.

Claude Code's implementation:
```typescript
async function* queryLoop(messages, tools, client) {
  while (true) {
    const safe = snipIfNeeded(messages)
    const { text, toolUses, stopReason } = await callModel(safe, tools, client)
    yield { type: 'text', text }
    if (stopReason === 'end_turn' || toolUses.length === 0) break
    const results = await Promise.all(toolUses.map(t => executeTool(t, tools)))
    messages.push({ role: 'assistant', content: [...toolUses] })
    messages.push({ role: 'user', content: results.map(toToolResult) })
  }
}
```

12 lines. This is the entire core of a production Agent. Everything else is
refinement.

## Architecture Decision Checklist

Work through these decisions in order. Each one builds on the previous.

---

### Decision 1: The Loop Structure

**Stateful Session vs Stateless Loop — separate them.**

Claude Code splits this into two layers:
- `QueryEngine` (stateful): owns message history, cumulative token costs, file
  state caches, permission records. Lives for the entire session.
- `query()` (stateless): receives the world as parameters, produces events through
  an AsyncGenerator. Lives for one turn.

Why this matters: the stateless loop is fully unit-testable with injected
dependencies. When you add SDK support or multi-agent later, you wrap the
stateless loop in a new stateful session manager without modifying the loop itself.

**Your decision:** separate your session state from your loop logic. The session
owns "what has happened." The loop owns "what to do next."

> Deep dive → `principles/02-boundary-is-product.md` (separate by state lifetime)

---

### Decision 2: Tool System Design

**Tools are not functions — they are a lifecycle.**

A "tool" in most Agent frameworks is a function: `name + description + handler`.
Claude Code's tools have a 6-step lifecycle:

```
1. Define    — ToolDef<I,O> interface with typed input/output schemas
2. Register  — added to tool pool (static, feature-gated, or dynamic MCP)
3. Lazy Load — schema constructed only when tool is first used (lazySchema)
4. Validate  — pure function, no side effects, early fail on bad input
5. Authorize — check permissions, may show UI dialog, may block
6. Execute   — actual operation, errors caught and returned as tool_result
```

**The rule that saves your Agent:** tool.call() must NEVER throw an unhandled
exception. An unhandled throw in a tool terminates the entire Agent session.
Wrap every tool execution in try/catch and return the error as a string result.
The model can read the error and decide what to do next. This is the difference
between "Agent crashed" and "Agent recovered."

```typescript
async function executeTool(toolUse, tools) {
  const tool = tools.find(t => t.name === toolUse.name)
  if (!tool) return `Error: tool "${toolUse.name}" not found`
  try {
    return await tool.call(toolUse.input)
  } catch (e) {
    return `Error: ${e instanceof Error ? e.message : String(e)}`
  }
}
```

**Tool metadata drives system behavior:**
- `isReadOnly`: scheduler can parallelize read-only tools safely
- `isConcurrencySafe`: explicitly marks tools safe for concurrent execution
- `estimatedTokenCost`: planner uses this for budget estimation
- `timeoutMs`: hard limit per tool call, enforced by the executor

The LLM does NOT decide concurrency — the tool metadata does. This eliminates
the entire class of bugs where "the model decided to run two write tools
simultaneously."

**Your decision:**
- Define a tool interface with at minimum: name, description, inputSchema, call()
- Add `isReadOnly` and `timeoutMs` metadata from day one
- Catch all errors in executeTool — never let a tool crash the loop
- Use lazySchema if you have >10 tools (schema tokens add up in every API call)

> Deep dive → `principles/06-policy-in-code-not-wiki.md` (encode tool safety in metadata, not docs)
> Deep dive → `principles/03-lifecycle-not-function.md` (tool lifecycle: define → register → validate → authorize → execute)

---

### Decision 3: Context Is a Scarce Resource

**This is the #1 difference between a prototype Agent and a production Agent.**

A prototype agent appends every message and tool result to the conversation until
the context window fills up, then crashes with `prompt_too_long`. A production
agent manages context as a scarce resource with an explicit budget.

Claude Code's 4-level compression pipeline (ordered by cost):

```
Level 1: Snip — truncate oversized tool results
  Cost: zero (string operation)
  Loss: partial (tail of long outputs)
  When: any single tool result > threshold (e.g., 5000 chars)
  How:  keep first N chars + "[...truncated, original length M chars]"

Level 2: Microcompact — replace images with text descriptions
  Cost: zero (substitution)
  Loss: minimal (images rarely re-needed verbatim)
  When: after snip, still over budget

Level 3: Context Collapse — fold duplicate/stale content
  Cost: zero (deduplication)
  Loss: near-zero (identical file reads collapsed)
  When: after microcompact, still over budget

Level 4: AutoCompact — AI summarizes the full conversation
  Cost: one API call (expensive, lossy)
  Loss: significant (exact code lines, error messages lost)
  When: only after 1-3 all fail
  Trigger: estimated tokens > (context_window - max(max_output, 20000) - 13000)
```

Level 4 has a circuit breaker: if AutoCompact fails 3 consecutive times, stop
trying. Without this, a bug in compression creates an infinite loop of failed
compression attempts.

After AutoCompact, Claude Code re-reads critical files that were referenced in
recent messages. This recovers precise information that the summary lost.

**The minimum viable implementation (covers 90% of cases):**

```typescript
// Level 1: snip long tool results
function snipIfNeeded(messages) {
  for (const msg of messages) {
    for (const block of msg.content) {
      if (block.type === 'tool_result' && block.text.length > 5000) {
        block.text = block.text.slice(0, 5000) +
          `\n[...truncated, original ${block.text.length} chars]`
      }
    }
  }
  return messages
}

// Level 4: autocompact when approaching limit
function shouldCompact(messages, maxContextTokens) {
  const estimated = JSON.stringify(messages).length * 0.3  // rough char→token
  return estimated > maxContextTokens * 0.7
}
```

Start with these two. Add levels 2-3 when you have real usage data showing
where your context budget goes.

**Your decision:**
- Implement snip (Level 1) from day one — it's free and prevents the most
  common crash
- Implement autocompact (Level 4) before your first long-running task test
- Add a circuit breaker on autocompact (max 3 consecutive failures)
- Measure where your tokens go: system prompt, tool schemas, tool results,
  conversation history — you can't optimize what you don't measure

> Deep dive → `principles/01-minimum-intervention.md` (intervention cost ladder)
> Deep dive → `domains/cache-aware-architecture.md` (prompt cache prefix optimization)

---

### Decision 4: Termination Conditions

**An Agent without explicit termination is a bomb.**

Your loop MUST have a list of exit conditions checked at the top of every
iteration. Claude Code checks:

```
- end_turn: model decided it's done
- max_turns: safety limit (prevents infinite loops)
- token_budget: cumulative cost ceiling
- user_abort: Ctrl+C / cancel signal
- error_fatal: unrecoverable failure (e.g., auth expired)
- max_consecutive_failures: circuit breaker (3+ tool failures in a row)
```

Without `max_turns`, a confused model calling tools forever will drain your
API budget. Without `max_consecutive_failures`, a broken tool creates an
infinite retry → replan → retry loop.

**Your decision:** implement ALL of these at the top of your loop:
```typescript
while (true) {
  if (turns >= MAX_TURNS) return terminate('max_turns')
  if (tokensUsed >= tokenBudget) return terminate('token_budget')
  if (abortSignal.aborted) return terminate('user_abort')
  if (consecutiveFailures >= 3) return terminate('circuit_breaker')
  // ... actual loop body
}
```

> Deep dive → `principles/03-lifecycle-not-function.md` (explicit termination conditions)

---

### Decision 5: Streaming, Not Batching

**Don't wait for the full API response before doing anything.**

Two levels of streaming that matter for Agent products:

**Level 1: Stream text to user as tokens arrive.**
The model generates 50-100 tokens/second. Waiting for the full response means
the user sees nothing for 5-10 seconds, then a wall of text. Streaming token
by token transforms "is it frozen?" into "it's thinking."

**Level 2: Start tool execution as soon as tool_use blocks complete in the stream.**
Claude Code's `StreamingToolExecutor` doesn't wait for the full response. As
soon as a `tool_use` block is complete (while the model may still be generating
the next tool call), execution begins:

```
API stream: [text...] [tool_1 complete] [text...] [tool_2 complete] [end]
Execution:            |── tool_1 ──────|           |── tool_2 ──────|
```

For a 3-tool response where each tool takes 2 seconds, batch execution takes
6 seconds. Streaming execution takes ~4 seconds (tools 1 and 2 overlap with
API generation time). At scale, this adds up.

**Your decision:**
- Level 1 (stream text) is non-negotiable — implement on day one
- Level 2 (streaming tool execution) is a significant optimization — implement
  when you have multi-tool workflows

> Deep dive → `domains/streaming-and-realtime.md` (AsyncGenerator, backpressure, progressive rendering)
> Deep dive → `principles/05-constraint-as-fuel.md` (latency constraint → streaming architecture)

---

### Decision 6: Permission and Safety Model

**When your Agent has write access, permission design is not optional.**

Claude Code's permission model:
```
1. Tool declares required permissions in metadata
2. System checks against allowlist (pre-approved patterns)
3. If not pre-approved, prompt user for specific permission
4. Cache approval for this session (not permanently)
5. Never infer permission from conversation context
```

The key insight: the LLM should NEVER decide its own permissions. The model's
job is to request tool use. The system's job is to decide if that use is allowed.
These are different trust boundaries.

For file operations specifically, Claude Code enforces the `readFileState`
invariant: you cannot edit a file you haven't read. This prevents blind
overwrites — the model must first see the current content, then propose a
specific edit.

**Minimum viable permission model:**
```
- Read operations: auto-approve (low risk)
- Write operations: require user confirmation (show file + diff)
- Shell commands: require user confirmation (show full command)
- Destructive operations (rm, git push --force): always confirm, show warning
```

**Your decision:**
- At minimum, confirm write operations before executing
- Enforce read-before-edit for file modifications
- Never let the model auto-approve its own actions

> Deep dive → `principles/04-earn-trust-progressively.md` (contextual, non-transitive trust)
> Deep dive → `principles/06-policy-in-code-not-wiki.md` (encode permissions in types, not comments)

---

### Decision 7: The Phased Build Path

**Don't build everything at once. Each phase produces a working artifact.**

```
Phase 1 (1 hour): Single LLM call
  - API client + streaming text output
  - No tools, no loop
  - Validates: API connectivity, streaming works

Phase 2 (2 hours): Tool call (one-shot)
  - Add 1 tool (read_file), pass to API
  - Execute tool, print result
  - No loop — tool result is not fed back to model
  - Validates: tool definition format, execution, error handling

Phase 3 (3 hours): QueryLoop + message history ← YOU ARE NOW AN AGENT
  - Implement the while(true) loop
  - Tool results injected as tool_result messages
  - Add 3 more tools (write_file, bash, glob)
  - Validates: multi-turn tool use, model sees tool results

Phase 4 (1 hour): REPL interaction
  - readline-based interactive loop
  - Session manager persists messages across turns
  - Validates: multi-turn conversation, context carries over

Phase 5 (2 hours): Context compression
  - Snip long tool results (Level 1)
  - AutoCompact when approaching token limit (Level 4)
  - Validates: long sessions don't crash
```

**Phase 3 is the inflection point.** Before it, you have a chatbot. After it,
you have an Agent. Don't skip to Phase 5 before Phase 3 works perfectly.

---

## What You're Giving Up (and Why That's OK)

| Capability | Claude Code Has It | Skip For Now? | Why |
|-----------|-------------------|---------------|-----|
| React/Ink TUI | Yes — Yoga Flexbox, blit rendering | Yes | `stdout.write` is fine for dev tools |
| 7-layer auth | Yes — multi-cloud, OAuth, keychain | Yes | `ANTHROPIC_API_KEY` env var is fine to start |
| Multi-agent | Yes — Fork, Team, Coordinator | Yes | Single agent covers 80% of use cases |
| MCP tools | Yes — dynamic runtime discovery | Yes | Hardcoded tools are fine until you need extensibility |
| Session persistence | Yes — JSONL transcript, /resume | Yes | Process restart losing history is acceptable for v1 |
| Parallel tool execution | Yes — StreamingToolExecutor | Maybe | Promise.all for concurrent-safe tools is easy to add |
| Plugin system | Yes — Plugin/Skill/Hook | Yes | Build the extension points when you have extensions |

**What you CANNOT skip:**
- The loop (Decision 1)
- Error handling in tools — never throw (Decision 2)
- Context compression — at least snip (Decision 3)
- Termination conditions — at least max_turns (Decision 4)
- Streaming text to user (Decision 5)

---

## Scaling From Solo Agent to Multi-Agent

When you outgrow a single agent, read `references/09-multi-agent-coordination.md`
for the full pattern. But here's the decision framework:

```
"I need to do N things in parallel, each independent"
  → Fork pattern (L1): share context prefix, collect results
  → Example: search 5 codebases simultaneously

"I need agents that communicate back and forth"
  → Team pattern (L2): separate contexts, message queue
  → Example: code review agent + fix agent iterating

"I need agents that can't see each other's work"
  → Coordinator pattern (L3): orchestrator mediates all communication
  → Example: security audit from 3 independent perspectives
```

Always start with L1. Upgrade to L2 only when L1's "no communication" constraint
blocks you. Upgrade to L3 only when L2's "shared visibility" is a security problem.

---

## Common Pitfalls (from production data)

**"Agent loops forever"** — Missing termination conditions. The model gets confused,
keeps calling the same tool with slightly different inputs. Fix: `max_turns` +
`max_consecutive_failures`. Claude Code data: without circuit breaker, 1,279
sessions generated 50+ consecutive failures, wasting 250K API calls/day.

**"Agent crashes mid-task"** — A tool threw an unhandled exception. Fix: catch ALL
errors in executeTool, return as string. The model reads the error and adapts.

**"Context too long"** — User had a 30-minute session, context hit 200K tokens.
Fix: implement snip (5 minutes of work) + autocompact (30 minutes of work).

**"Agent hallucinates file contents"** — Model was asked to edit a file it hadn't
read. Without the actual content in context, it guesses. Fix: enforce
read-before-edit invariant.

**"Costs are out of control"** — No token budget, no measurement. Fix: track
`usage.input_tokens + usage.output_tokens` from every API response. Set a
`maxTokens` budget per session. Kill the loop when budget is exhausted.

**"Whale sessions"** — One user's Agent spawned 292 sub-agents, consuming 36.8GB
memory in 2 minutes. Fix: hard cap on concurrent agents
(`TEAMMATE_MESSAGES_UI_CAP = 50`), circuit breaker on agent spawn rate.
