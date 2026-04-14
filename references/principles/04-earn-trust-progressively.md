# Earn Trust Progressively

## What this principle says

Never demand all trust upfront. Earn it at the moment the user needs it, at the
minimum cost to them. Trust is contextual, non-transitive, and graduated.

## Why think this way

Most products treat trust as binary: either the user trusts you or they don't. In
reality, trust is a spectrum that changes with context. A user who grants file-read
permission hasn't granted file-delete permission. A user who trusts your app in a
browser doesn't automatically trust it on their server. A user who subscribed last
month hasn't pre-approved a price increase.

The deeper insight: **premature trust requests are conversion killers**. Every
permission dialog, sign-up field, and pricing gate is a moment where the user
decides whether to keep going or leave. Minimize these moments. Make each one
earn its keep.

## How Claude Code does it

### Authorization non-transitivity

The system prompt explicitly states: "A user approving an action once does NOT mean
they approve it in all contexts." This is a counter-measure against a known LLM
failure mode — over-generalizing from a specific approval. The model is treated as
a potential trust escalation vector, and the system prompt engineers against it.

### Context-aware trust levels

`isManagedOAuthContext()` detects when Claude Code runs as a subprocess of Claude
Desktop vs. direct user invocation. In managed context, local API key lookup is
disabled — preventing credential conflicts. The rule: whoever spawned you owns
your auth context. Trust flows downward from the invoker, not upward from stored
credentials.

### Permission-on-demand

The permission system never pre-collects approvals. When a tool needs file-write
access, it checks:
1. In-memory permission cache → already approved this session?
2. File-based allowlist rules → matches a configured pattern?
3. Ask user → specific, minimal permission request

Step 3 only fires when 1 and 2 fail. The user is never asked for permissions
the system doesn't immediately need.

### Sunrise rollout pattern

The Buddy companion feature rolls out by local timezone, not UTC midnight. A
UTC flip would create a thundering herd — every user worldwide activates
simultaneously, spiking API load. Local-timezone gating creates a rolling wave
across timezones, smoothing the load curve. This is operational trust: the team
doesn't trust itself to handle a global synchronization point, so it eliminates
the need for one.

### Settings isolation for embedded contexts

`allowedSettingSources` can restrict configuration to only CLI flags, ignoring
all disk-based config. This lets SDK callers create hermetically sealed instances
unaffected by user config in `~/.claude/settings.json`. Trust is scoped to the
caller's intent, not to what happens to be on disk.

## Migrate to your product

### Product Onboarding
```
BAD:  Sign up → ask name, company, phone, address, payment → show product
GOOD: Sign up → email only → show product immediately → ask name when profile
      needed → ask payment when they hit a paid feature
```
Each field you add to signup is ~10% drop-off. Earn the right to ask by first
delivering value. The Nebutra Sailor license wizard is a good example: visitors
can clone/star/fork without any gate. License registration happens only when
they want community features.

### API Scope Design
```
BAD:  Single "admin" API token with access to everything
GOOD: Scoped tokens: read:users, write:orders, admin:billing
      Each scope requested at the moment a feature needs it
```
GitHub's OAuth scopes are the gold standard: apps request `repo:read` initially,
then `repo:write` only when the user tries to push.

### Pricing Tiers
```
BAD:  "Choose a plan" as the first screen after signup
GOOD: Free tier → usage grows → soft limit warning → upgrade prompt with
      context ("You've used 80% of your free API calls this month")
```
The upgrade prompt is more effective at 80% usage than at 0% because the user
has already experienced value and understands what they're paying for.

### Feature Rollouts
```
BAD:  Feature flag → on/off for everyone at UTC midnight
GOOD: Internal → 1% canary → 10% beta → 50% → 100%
      Each stage has rollback criteria and monitoring
      Geographic or timezone-based rollout smooths load
```

### Open Source Trust Model
```
BAD:  Require sign-up to access the code
GOOD: Code is public (clone, read, fork freely)
      Community features require registration
      Commercial use requires license
```
Separate "access to code" from "access to ecosystem." The code earns trust;
the ecosystem converts trust to revenue.

## Anti-patterns

**"Just ask for everything upfront"** — Mobile app requests camera, location,
contacts, microphone, notifications on first launch. User denies all or
uninstalls. Each permission should be requested at the moment the feature
needs it, with context explaining why.

**"One API key to rule them all"** — A single admin key is convenient for the
developer and catastrophic for security. When (not if) the key leaks, the
blast radius is total. Scoped keys limit damage to one capability.

**"Big bang launch"** — Ship the feature to 100% of users simultaneously.
If there's a bug, 100% of users hit it simultaneously. Graduated rollout
means the first 1% finds the bug before the other 99% encounter it.

**"Trust the LLM's judgment about permissions"** — If the model decides it
has permission based on prior context, it will over-generalize. Permission
decisions must be checked at runtime against the actual policy, not inferred
from conversation history.

## Self-check questions

1. How many fields/permissions/decisions does your user face before they get
   value from your product? Can you reduce that number?

2. When your system grants a permission, what scope does it have? Is it the
   minimum scope needed, or a convenient superset?

3. If you rolled out your last feature to 1% instead of 100%, would you have
   caught the bug earlier? Do you have the infrastructure for graduated rollout?
