# The blackboard

`.orchestra/` at the project root. It is the crew's only shared memory, because agents share no
context with each other.

```
.orchestra/
  brief.md         # the user's plan, verbatim, never edited
  design.md        # designer output
  tasks.md         # THE LEDGER — slices and defects
  roster.md        # slot → slice → worktree branch
  decisions.md     # append-only log of the user's redirections
  reports/*.md     # per-agent running notes
  state.md         # halt checkpoint
  context/
    digest.md      # the distilled memory, rewritten in place, ~1 page
    timeline.md    # append-only one-line facts
    drift.md       # contradictions caught by the archivist
  dashboard.html   # the live board, published as an Artifact
```

## Single-writer ownership

Parallel agents editing one file is the classic multi-agent data-loss bug: two writes race, one
silently wins, work vanishes. So **every file has exactly one writer.**

| File | Sole writer | Everyone else |
|---|---|---|
| `brief.md` | main (once, at intake) | read only |
| `design.md` | designer | read only |
| `CLAUDE.md` (project) | designer | read only |
| `tasks.md` | **planner** | read only |
| `roster.md` | planner | read only |
| `decisions.md` | main (append-only) | read only |
| `reports/builder-N.md` | builder-N | read only |
| `reports/tester.md` | tester | read only |
| `context/digest.md` | **archivist** | read only |
| `context/timeline.md` | archivist (append-only) | read only |
| `context/drift.md` | archivist | read only |
| `dashboard.html` | main | — |
| `state.md` | main | read only |

**Builders never write the ledger.** You report your state; the planner records it. If you think
`tasks.md` is wrong, `[STATUS]` the planner with the correction — do not edit it yourself.

## `tasks.md` — the ledger

Planner-owned. Two tables, always both present.

```markdown
# Ledger — <project name>
phase: build
updated: 14:32

## Slices
| id | slice         | owner     | status   | files touched   | depends on | progress |
|----|---------------|-----------|----------|-----------------|------------|----------|
| S1 | Auth pages    | builder-1 | building | app/login/**    | —          | 3/5      |
| S2 | Cart totals   | builder-2 | defect   | lib/cart/**     | —          | 5/5      |
| S3 | Checkout flow | builder-3 | blocked  | app/checkout/** | S2         | 1/4      |

## Defects
| id | slice | severity | summary                     | state      | attempt |
|----|-------|----------|-----------------------------|------------|---------|
| D1 | S2    | high     | total is NaN on empty cart  | dispatched | 1       |
```

**Slice statuses:** `pending` → `assigned` → `building` → `built` → `testing` → `passed`, with
`testing` → `defect` → `fixing` → back to `testing`. `blocked` is reachable from any state and
means a planner decision is needed. `dead` means the owning agent is gone (see liveness, below).

**Defect states:** `open` → `dispatched` → `fixing` → `verifying` → `closed`, or `escalated`
after 3 failed attempts.

**`files touched` is a contract, not a note.** It's how the planner keeps parallel builders off
each other's files. Stay inside it.

## `roster.md`

```markdown
| slot      | slice | worktree branch      | spawned |
|-----------|-------|----------------------|---------|
| builder-1 | S1    | orch/s1-auth-pages   | 14:02   |
| builder-2 | S2    | orch/s2-cart-totals  | 14:02   |
```

## `context/digest.md`

Archivist-owned. **Rewritten in place, never appended** — if it grows past roughly a page it
stops being cheap to re-read and the whole mechanism fails.

```markdown
# Digest — updated 14:28

## What we're building
Two sentences, from the brief.

## Locked decisions
- D-01 Next.js 16 + Tailwind v4 — Phase 1; matches the user's existing stack
- D-07 Cart totals exclude tax — after D1; tax is a checkout-time concern

## Conventions in force
See project CLAUDE.md. Changed since: dark-only palette (user redirect, 14:15).

## Shipped and verified
- S1 Auth pages — /login posts to a server action, sets an httpOnly cookie

## Known broken
- D1 open: cart total NaN on empty cart (S2, attempt 1)

## Your redirects
- 14:15 make the UI dark mode → applied to S2, S4

## Open questions
- Where does the order id persist? (S3 blocked on this)
```

**Locked decisions carry their reason.** That's what stops a later agent re-litigating them. An
agent that wants one changed must `[BLOCKED]` the planner.

## `decisions.md`

Append-only, main-owned. Every user redirection, in order, with a timestamp and the verbatim
request. This is the audit trail — never rewrite history here.

## Housekeeping

`.orchestra/` is gitignored by default. It is machine-owned: don't hand-edit it during a run, and
don't `git add` it unless the user asks for a build audit trail.
