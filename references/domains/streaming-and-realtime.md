# Streaming and Real-Time System Design

> Supplementary reference — extends Principle 3 (Lifecycle, Not Function) and
> Principle 1 (Minimum Intervention) into the domain of streaming data,
> real-time feedback, and backpressure management.

## What this reference covers

How to design systems that process and deliver data incrementally — streaming
API responses, real-time UI updates, progressive loading, and flow control
between producers and consumers with different speeds.

## Why this matters

Users don't want to wait for completion. They want progressive feedback: see
the first word while the last word is still being generated, see the first
search result while the remaining 99 are still loading, see the build progress
while the build is still running.

The deeper insight: **streaming is not a performance optimization — it is a UX
architecture**. The decision to stream affects data structures, error handling,
resource management, and API design at every layer. Adding streaming to a
batch-oriented system is a rewrite, not a refactor.

## How Claude Code does it

### AsyncGenerator as the Core Streaming Primitive

Claude Code uses `AsyncGenerator<StreamEvent>` as the universal interface
between the query loop, tool execution, and UI rendering:

```typescript
async function* query(deps, state): AsyncGenerator<StreamEvent> {
  // Each event is yielded as it happens — consumer controls pace
  yield { type: 'text_delta', content: '...' }
  yield { type: 'tool_use_start', toolId: '...' }
  yield { type: 'tool_result', result: '...' }
}
```

Three properties of AsyncGenerator that make it perfect for streaming AI:

**1. Backpressure** — The consumer (UI renderer) controls the pace. If the
UI is slow to render, the generator pauses at the next `yield` until the
consumer calls `.next()`. No buffer overflow, no dropped events, no
unbounded memory growth.

```typescript
// Consumer controls pace — generator pauses between yields
for await (const event of query(deps, state)) {
  await renderer.render(event)  // slow render = generator waits
}
```

**2. Cascading cancellation** — When the user presses Ctrl+C, calling
`.return()` on the generator propagates cancellation through the entire
chain. Resource cleanup runs via `Symbol.dispose` (ES2023 explicit resource
management). No orphaned API calls, no leaked connections.

```typescript
// Cancellation propagates through the chain
const gen = query(deps, state)
// ... user presses Ctrl+C
await gen.return()  // triggers finally{} blocks in generator
                    // Symbol.dispose cleans up connections
```

**3. Composable streaming** — Sub-agent output can be delegated via `yield*`,
making the parent consumer unaware that events came from a child:

```typescript
async function* parentQuery() {
  yield { type: 'text', content: 'Starting sub-task...' }
  yield* childAgentQuery()  // child events flow through transparently
  yield { type: 'text', content: 'Sub-task complete.' }
}
```

### Streaming Tool Execution (during API response)

Claude Code doesn't wait for the API response to complete before starting tool
execution. As soon as a `tool_use` block is complete in the stream, execution
begins — even while the model is still generating the next tool call:

```
Time →
API stream: [text...] [tool_use_1 complete] [text...] [tool_use_2 complete] [end]
Execution:            |── tool_1 running ──|           |── tool_2 running ──|
```

`StreamingToolExecutor` batches concurrent tool calls within one response,
respecting `isConcurrencySafe` metadata for parallelism decisions.

### Message Withholding (Withhold-then-Recover Pattern)

During streaming, some messages should be held back from the UI until a
decision point is reached:

```
Scenario: model generates text, then decides to call a tool

Without withholding:
  User sees: "Let me read the file" → [text disappears, tool runs]
  UX: jarring, content appears and vanishes

With withholding:
  Generator yields events but UI withholds display until:
  - If model stops with end_turn → release all text to UI
  - If model calls a tool → withhold pre-tool text, show tool execution
  Post-tool: recover withheld text in appropriate context
```

Three types of withheld content:
1. Pre-tool text (assistant thinking before tool call)
2. Partial tool arguments (streaming JSON that's not yet valid)
3. Intermediate reasoning (chain-of-thought before conclusion)

### Progressive UI Rendering (React + Ink)

The Ink renderer receives StreamEvents and updates the terminal UI incrementally:

```
Frame 1: "Searching..."               [spinner]
Frame 2: "Searching... found 3 files"  [spinner]
Frame 3: "Reading src/index.ts..."     [spinner]
Frame 4: "Here's what I found:"       [text output]
```

The blit optimization tracks which screen rows changed between frames and only
outputs those rows. Stable frames (user hasn't typed, no new events) cost zero
output. This is the same incremental diffing strategy React uses for the DOM.

### EndTruncatingAccumulator

For long-running tool output (e.g., a bash command that produces 10MB of output),
the accumulator keeps the first N bytes and the last M bytes, discarding the middle:

```
[first 1KB of output] ... [truncated 9.99MB] ... [last 1KB of output]
```

This is Principle 1 applied to streaming: the lightest intervention (keep
head + tail) before the heavier intervention (summarize or discard entirely).

### Circuit Breaker for Streaming Failures

When the API stream drops repeatedly (network issues, rate limits), the circuit
breaker prevents infinite reconnection storms:

```
CLOSED → errors accumulate → OPEN (stop trying for 60s) → HALF_OPEN (try one)
  ↑                                                              │
  └──── success ────────────────────────────────────────────────┘
```

Production data: without circuit breaker, 1,279 sessions accumulated 50+
consecutive failures each, generating 250K wasted API calls/day. The circuit
breaker is the cheapest intervention that prevents the most expensive waste.

## Migrate to your product

### SSE / WebSocket Streaming API
```typescript
// Server-Sent Events for progressive delivery
// Each event is self-contained — client can render immediately

app.get('/api/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream')
  res.setHeader('Cache-Control', 'no-cache')
  res.setHeader('Connection', 'keep-alive')

  for await (const event of processTask(req.body)) {
    res.write(`event: ${event.type}\n`)
    res.write(`id: ${event.id}\n`)          // enables resume on reconnect
    res.write(`data: ${JSON.stringify(event.data)}\n\n`)
  }
})

// Client reconnects with Last-Event-ID header
// Server replays events after that ID
// → no lost events on network interruption
```

### Progressive Loading Pattern
```
Instead of: loading... loading... loading... [all content at once]
Do:         [header] → [first item] → [second item] → [remaining items]

// React example with streaming data
function SearchResults({ query }) {
  const stream = useStreamingResults(query)

  return (
    <div>
      {stream.results.map(result => <ResultCard key={result.id} {...result} />)}
      {stream.loading && <Skeleton />}
      {stream.done && <p>{stream.results.length} results</p>}
    </div>
  )
}
```

### Backpressure in Data Pipelines
```
Producer (fast) → Buffer → Consumer (slow)

Without backpressure: buffer grows unbounded → OOM
With backpressure: producer pauses when buffer is full

// Node.js streams have built-in backpressure
readableStream
  .pipe(transformStream)   // transform pauses readable when slow
  .pipe(writableStream)    // writable pauses transform when slow

// AsyncGenerator provides natural backpressure
async function* producer() {
  for (const item of items) {
    yield item  // pauses here until consumer calls .next()
  }
}
```

### Real-Time Collaboration
```
Operational Transform / CRDT patterns:
1. Local optimistic update (instant feedback)
2. Send operation to server
3. Server broadcasts to other clients
4. Resolve conflicts via OT/CRDT merge

This is Principle 1 applied: the lightest intervention is local optimistic
update (free, instant). Server reconciliation is the heavier intervention
that only fires when needed (conflict).
```

### CLI / Terminal Progressive Output
```
Instead of: [wait 30 seconds] [dump all output]
Do:         "Analyzing..." → "Found 47 files" → "Checking types..." → "3 errors"

// Use carriage return for in-place updates
process.stdout.write('\rAnalyzing... found 47 files')
// Or use a framework (Ink, ora, listr2) for structured progress
```

### Long-Running Task Progress
```
For tasks taking minutes to hours:

1. SSE stream for real-time progress events
2. Polling endpoint as fallback (SSE drops on mobile/proxy)
3. Webhook notification on completion
4. Email/push notification as last resort

Each is a level in the notification intervention ladder:
  SSE (free, real-time) → Polling (cheap, 5s interval) →
  Webhook (reliable, slight delay) → Push notification (guaranteed, intrusive)
```

## Anti-patterns

**"Batch then send"** — Waiting for the entire LLM response before showing
anything. Users see a spinner for 10 seconds, then a wall of text. Stream
token by token — the UX difference is dramatic.

**"Unbounded buffer"** — Collecting streaming events in an array without
limit. A long session produces millions of events. Use ring buffers,
truncation, or streaming-to-disk for long-running processes.

**"Fire and forget"** — Emitting events without backpressure. If the
consumer is slow (slow network, slow renderer), events pile up in memory.
AsyncGenerator, Node streams, or explicit acknowledgment prevent this.

**"Restart from scratch on disconnect"** — SSE drops because of a network
blip. Without `Last-Event-ID`, the client must restart the entire operation.
Event IDs enable resume from the exact interruption point.

**"Synchronous tool execution"** — Waiting for the full API response before
executing any tools. Claude Code starts executing tools as soon as they're
complete in the stream, while the model continues generating. This parallelism
can cut total latency significantly.

## Self-check questions

1. When your system processes a long-running request, what does the user see?
   A spinner? Or progressive feedback? If a spinner, you're leaving UX value
   on the table.

2. What happens when your stream consumer is slower than your producer?
   Does memory grow unbounded? Or is there a backpressure mechanism?

3. If the connection drops mid-stream, can the client resume? Or must it
   restart from the beginning? Event IDs and checkpoints prevent re-work.
