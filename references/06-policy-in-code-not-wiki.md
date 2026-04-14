# Policy In Code, Not Wiki

## What this principle says

Architectural rules, security policies, and invariants that live only in
documentation WILL be violated. Encode them into identifiers, type systems,
linters, CI gates, and automation. The cost of encoding is low; the cost of
violation is high. The investment is asymmetric.

## Why think this way

Documentation gets stale. Comments drift from code. New team members don't read
the wiki. Even experienced engineers forget rules under deadline pressure. The
only reliable enforcement is enforcement that runs automatically — at compile
time, at commit time, or at deploy time.

The deeper insight: **the shape of your code should make wrong things hard and
right things easy**. If doing the wrong thing requires extra effort (fighting
the type system, suppressing a lint error, overriding a CI check), developers
will naturally do the right thing. If doing the wrong thing is the path of
least resistance, they will naturally do the wrong thing, no matter how many
wiki pages you write.

## How Claude Code does it

### DANGEROUS_ function prefix

`DANGEROUS_uncachedSystemPromptSection(content, _reason)` packs three enforcement
mechanisms into one function signature:

1. **The name**: `DANGEROUS_` appears in every diff that adds a new call site.
   Code reviewers see it. `grep DANGEROUS_` across the codebase is a one-command
   cost audit.

2. **The `_reason` parameter**: TypeScript's underscore prefix signals "unused at
   runtime," but the parameter forces every caller to write a justification string.
   The justification outlasts the author — future maintainers understand WHY this
   uncached section was added.

3. **The function itself**: centralizing uncached prompt sections means there's
   exactly one place to add monitoring, exactly one place to add cost tracking,
   and exactly one grep to find all exceptions.

### Bootstrap isolation ESLint rule

`bootstrap/state.ts` is a leaf node in the dependency graph. It cannot import any
internal business module. This is enforced by a custom ESLint rule
(`bootstrap-isolation`), not by a comment.

Why it matters: if `state.ts` imported a business module, and that module imported
something that imported `state.ts`, Node.js module initialization would produce a
circular dependency — sometimes silently returning `undefined` for imported values.
Debugging this costs hours. The ESLint rule costs minutes to write and prevents the
problem forever.

### Tool metadata as scheduler policy

Each tool declares `isReadOnly()` and `isConcurrencySafe()` as metadata. The
scheduler reads this metadata to automatically decide which tools can run in
parallel. The LLM doesn't make concurrency decisions — the type system does.

This eliminates an entire class of bugs: "the model decided to run two write
tools simultaneously and they corrupted each other's output." The policy (which
tools are safe to parallelize) is encoded in the tool definition, not in a prompt
instruction that the model might ignore.

### INTERNAL_ONLY sections

Internal-only commands and prompt sections are gated by code-level arrays and
conditionals, not by "please don't use this" documentation. The array
`INTERNAL_ONLY_COMMANDS` is checked at registration time. If you're not in an
internal context, the commands don't exist — they're not hidden, they're absent.

## Migrate to your product

### Naming Conventions as Policy
```typescript
// BAD: policy lives in a style guide nobody reads
// "Internal functions should not be called from outside their module"

// GOOD: policy lives in the name
function _internal_computeHash() { ... }
function unsafe_rawDatabaseQuery() { ... }
function DEPRECATED_oldPaymentFlow() { ... }
```
Prefixes appear in diffs, in grep, in IDE autocomplete. They're self-documenting
speed bumps.

### Branded Types as Compile-Time Policy
```typescript
// BAD: any string can be a userId or an orderId
function getOrder(userId: string, orderId: string) { ... }
getOrder(orderId, userId)  // compiles fine, bug at runtime

// GOOD: branded types prevent mixing
type UserId = string & { readonly __brand: 'UserId' }
type OrderId = string & { readonly __brand: 'OrderId' }
function getOrder(userId: UserId, orderId: OrderId) { ... }
getOrder(orderId, userId)  // TypeScript error at compile time
```

### Opaque Tokens as Gating Mechanisms
```typescript
// BAD: "always check budget before calling a tool" (wiki policy)

// GOOD: you literally cannot call execute() without a BudgetToken
type BudgetToken = { readonly __brand: 'BudgetToken' }
function acquireBudget(ledger: Ledger, estimate: number): BudgetToken | Error
function executeTool(input: unknown, budget: BudgetToken): Promise<Result>
```
The developer doesn't need to remember the policy. The type system remembers.

### Architecture Tests
```typescript
// Test that no route file imports prisma directly
test('all DB access goes through repository layer', () => {
  const routeFiles = glob.sync('src/routes/**/*.ts')
  for (const file of routeFiles) {
    const content = fs.readFileSync(file, 'utf-8')
    expect(content).not.toMatch(/from ['"]@nebutra\/db['"]/)
    expect(content).not.toMatch(/prisma\./)
  }
})

// Test that no package imports from a restricted module
test('analytics has zero business dependencies', () => {
  const pkg = JSON.parse(fs.readFileSync('packages/analytics/package.json'))
  const deps = Object.keys(pkg.dependencies || {})
  const businessPackages = deps.filter(d => d.startsWith('@myapp/'))
  expect(businessPackages).toEqual([])
})
```

### CI Gates
```yaml
# .github/workflows/policy.yml
- name: No secrets in code
  run: |
    if grep -rn "sk-proj-\|AKIA\|ghp_" src/; then
      echo "::error::Hardcoded secrets detected"
      exit 1
    fi

- name: No console.log in production code
  run: |
    if grep -rn "console\.log" src/ --include="*.ts" | grep -v "test\|spec"; then
      echo "::error::console.log in production code"
      exit 1
    fi
```

### Dependency Direction Rules
```javascript
// .dependency-cruiser.js or eslint-plugin-boundaries config
// "packages/ui may not import from packages/api"
// "packages/db may not import from packages/billing"
{
  forbidden: [
    { from: { path: "packages/ui" }, to: { path: "packages/api" } },
    { from: { path: "packages/db" }, to: { path: "packages/billing" } },
  ]
}
```

## Anti-patterns

**"The README says don't do this"** — Three months later, a new engineer does
exactly that. The README didn't change. The code didn't prevent it. The bug ships.

**"Code review will catch it"** — Reviewers are human. Under deadline pressure,
review quality drops. A linter never gets tired, never has a bad day, and never
approves a PR just to unblock a release.

**"We'll add tests later"** — Architecture tests are the cheapest tests to write
(often 5-10 lines of grep/glob logic) and prevent the most expensive bugs
(circular dependencies, layer violations, security policy breaches). Write them
first, not later.

**"Comments explain the constraint"** — Comments are invisible in diffs,
non-searchable in most review tools, and ignored by compilers. The constraint
should be visible in the code's SHAPE, not in its annotations.

## Self-check questions

1. What are your team's 3 most important architecture rules? How many are
   machine-checked (linter, type system, CI) vs. human-checked (code review,
   documentation)?

2. Could a new engineer, on their first day, violate a critical invariant
   without any tool stopping them? If yes, that invariant is a wiki policy,
   not a code policy.

3. When was the last time a production bug was caused by someone not following
   a documented rule? Is that rule now automated?
