# Boundary Is Product

## What this principle says

Every line you draw in your system — between modules, between services, between user
roles, between API endpoints — directly shapes user experience, error quality, and
system evolution. Boundaries are not organizational convenience; they are product decisions.

## Why think this way

Most teams draw boundaries by similarity: "these functions are related, put them together."
Unicorn teams draw boundaries by DIFFERENCE: "these two things LOOK similar but fail
differently, change at different rates, or serve different consumers — so they must be
separated."

The boundary IS the architecture. Everything else is implementation.

## How Claude Code does it

### Three-Phase Tool Execution (semantic boundary)

```
validateInput()    — pure function, no side effects
  Failure means: "your input is malformed"
  User sees: immediate error, no dialog shown

checkPermissions() — has side effects (UI dialog, policy check)
  Failure means: "you're not allowed to do this"
  User sees: permission request or denial message

call()             — actual operation (filesystem, network)
  Failure means: "the operation itself failed"
  User sees: error with actionable context
```

If these were one function, every error would say the same generic thing. The boundary
creates three different user experiences for three different problems.

### Project Identity vs Current Location

```
originalCwd — fixed at process start
  Used for: session file paths, CLAUDE.md discovery, skill lookups
  Semantic: "what project is this?"

cwd — follows `cd` operations
  Used for: tool file operations (read, write, glob)
  Semantic: "where am I working right now?"
```

If merged into one `cwd`, doing `cd ../backend` mid-session would break CLAUDE.md
discovery, change session file paths, and lose project context. Two concepts that
happen to both look like "current directory" are actually different things.

### Zero-Dependency Analytics Module

Analytics is imported by EVERYTHING: tools, commands, services, main loop.
If analytics imported ANY business module, every module transitively imports that
business module → dependency graph becomes near-complete → circular dependency risk.

Solution: analytics defines a zero-dependency event queue. The Datadog reporter
is injected at startup. The module that must be imported by everything depends on nothing.

### Stateful vs Stateless Split

```
QueryEngine (stateful) — owns message history, token costs, permission cache
  Lifetime: entire session

query() (stateless) — receives world as parameters, produces events via generator
  Lifetime: single turn
```

Boundary drawn by STATE LIFETIME, not by function type. This makes query() unit-testable
in total isolation (all dependencies injected). When SDK support was added, they wrapped
query() in QueryEngine without modifying query(). Open/closed principle via correct boundary.

## Migrate to your product

### Multi-Tenant SaaS — Where is the isolation boundary?

```
Option A: Application-level (WHERE tenant_id = ?)
  Pro: simple, single database
  Con: one bug leaks data across tenants
  
Option B: Database-level (Row-Level Security / separate schemas)
  Pro: isolation enforced by database engine, not application code
  Con: more complex migrations, connection management

Option C: Separate databases per tenant
  Pro: strongest isolation, independent scaling
  Con: operational complexity, cross-tenant queries impossible
```

The choice isn't technical — it's a product decision about your trust model.
Healthcare data? Option C. Blog platform? Option A is fine.

### API Design — Resource vs Action boundary

```
/users/:id/orders         — boundary says "orders belong to users"
/orders?userId=:id        — boundary says "orders are independent entities"
```

First: easy to get "user's orders" but hard to query orders across users.
Second: easy to query all orders but need explicit auth check per query.
The boundary determines what's easy and what's hard — that IS your product.

### Frontend/Backend — Where does the BFF live?

```
Frontend → BFF → Backend APIs
         ↑
    This boundary determines:
    - Who owns the data transformation?
    - Who decides what goes in one screen?
    - Who handles loading/error states?
```

BFF owned by frontend team: they ship faster, backend stays clean.
BFF owned by backend team: API is optimized, frontend team blocked by API changes.

## Anti-patterns

**"God module"** — One module that everything imports and that imports everything.
Any change ripples everywhere. Error messages are generic. Testing requires the world.

**"Premature microservices"** — Drawing boundaries before understanding the domain.
Result: distributed monolith that's harder to change than the monolith was.

**"Boundary by layer"** — `controllers/`, `services/`, `repositories/` directories.
Every feature change touches all three directories. Better: boundary by domain
(`users/`, `orders/`, `billing/`), each containing its own layers.

**"Shared everything"** — Shared types, shared utils, shared config across services.
The "shared" boundary becomes the coupling point that prevents independent deployment.

## Self-check questions

1. Can you draw your system's boundary diagram? For each line, can you explain
   what's DIFFERENT on each side (not what's similar)?

2. When your system errors, can the user tell from the error message WHICH boundary
   was violated? Or do they get a generic 500?

3. Do you have a module that's imported by everyone AND imports from everyone?
   That module is a boundary violation — it should depend on nothing, or be split.
