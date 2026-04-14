# Self-Assessment Audit Checklist

> All 21 self-check questions from the 7 principles, plus Agent-specific checks.
> Run through this list against your product. Every "no" is a gap worth investigating.

## Principle 1 — Minimum Intervention

- [ ] Can you draw your system's "intervention cost ladder" for its top 3 problems?
  Does each rung have a clear cost (time/money/UX impact)?
- [ ] For your most common problem, is it being solved by your cheapest intervention?
  Or are you using a $10 tool for a $0.01 problem?
- [ ] When was the last time your system escalated to the most expensive intervention?
  If it happens often, your cheaper rungs aren't working. If never, you might not need it.

## Principle 2 — Boundary Is Product

- [ ] Can you draw your system's boundary diagram? For each line, can you explain
  what's DIFFERENT on each side (not what it groups together)?
- [ ] When your system errors, can the user tell from the error message WHICH boundary
  was violated? Or do they get a generic 500?
- [ ] Do you have a module that's imported by everyone AND imports from everyone?
  That module is a boundary violation — it should depend on nothing, or be split.

## Principle 3 — Lifecycle, Not Function

- [ ] For your core entities (user, order, content, config), can you draw the complete
  state diagram? Every state, every transition, every termination condition?
- [ ] What happens to a record that gets "stuck" in a transitional state? Is there a
  timeout? A cleanup job? Or does it stay there forever?
- [ ] What is the oldest "active" record in your database? Should it still be "active"?
  If not, your lifecycle has a gap.

## Principle 4 — Earn Trust Progressively

- [ ] How many fields/permissions/decisions does your user face before they get
  value from your product? Can you reduce that number?
- [ ] When your system grants a permission, what scope does it have? Is it the
  minimum scope needed, or a convenient superset?
- [ ] If you rolled out your last feature to 1% instead of 100%, would you have
  caught the bug earlier? Do you have the infrastructure for graduated rollout?

## Principle 5 — Constraint As Fuel

- [ ] What are your top 3 constraints right now? For each one: have you tried to
  turn it into a product feature instead of working around it?
- [ ] If you removed your biggest constraint tomorrow, would your product actually
  be better? Or just more generic?
- [ ] What has your competitor NOT built because they don't face your constraint?
  That gap is your potential moat.

## Principle 6 — Policy In Code, Not Wiki

- [ ] What are your team's 3 most important architecture rules? How many are
  machine-checked (linter, type system, CI) vs. human-checked (code review, docs)?
- [ ] Could a new engineer, on their first day, violate a critical invariant
  without any tool stopping them? If yes, that invariant is a wiki policy, not code.
- [ ] When was the last time a production bug was caused by someone not following
  a documented rule? Is that rule now automated?

## Principle 7 — Soul Before Scale

- [ ] If you personified your product, what three adjectives describe its personality?
  Can you find those adjectives reflected in actual code decisions?
- [ ] Open your product's error page, `--help` text, and empty state. Would you be
  proud to show these to a user?
- [ ] Was your most recent technical choice made purely for technical reasons? Or did
  it also consider how the user would FEEL when interacting with the result?

---

## Agent-Specific Checks (from blueprints/agent-product.md)

- [ ] Does your agent have ALL three invariants? (loop + memory + feedback)
- [ ] Can a tool crash your entire agent session? (tool.call() must never throw)
- [ ] What happens when context hits the token limit? (snip + autocompact minimum)
- [ ] Does your loop have explicit termination conditions? (max_turns, token_budget,
  user_abort, consecutive_failures — all four)
- [ ] Does the LLM decide its own permissions? (it shouldn't — system decides)
- [ ] Can the agent edit a file it hasn't read? (read-before-edit invariant)
- [ ] Do you stream text to the user as tokens arrive? (non-negotiable)
- [ ] What's your circuit breaker threshold? (without one: infinite failure loops)

---

## Scoring

Count your checkmarks:

| Score | Assessment |
|-------|-----------|
| 25-29 | Unicorn-grade — you're shipping with conviction |
| 20-24 | Strong foundation — address the gaps before scaling |
| 15-19 | Prototype-grade — the gaps will hurt at production scale |
| 10-14 | Significant architectural debt — prioritize before adding features |
| <10 | Start with the Agent blueprint and principles 2, 3, 6 |
