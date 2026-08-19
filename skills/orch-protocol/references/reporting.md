# Reporting and heartbeats

Your report file is `.orchestra/reports/<your-name>.md`. You are its only writer.

## Why it has to be continuous

The user halts runs to save quota, sometimes without warning (`/halt` kills agents outright).
Across sessions your transcript is gone entirely — a resumed crew is a **fresh spawn rehydrated
from these files.** Whatever you didn't write down never happened.

So: **write as you finish each sub-step, not at the end.** A report updated five times during a
slice survives anything. A report written once at completion is worthless the moment you're
killed.

## Format

```markdown
# builder-1 · S1 Auth pages
branch: orch/s1-auth-pages
status: building 3/5

## Sub-steps
1. [x] login page shell — app/login/page.tsx
2. [x] server action posting to the auth provider — app/login/actions.ts
3. [x] session cookie set on success
4. [ ] error states (bad password, rate limited)
5. [ ] loading state

## Log
- 14:04 created app/login/page.tsx, form + labels per design.md §3
- 14:11 actions.ts posts and returns a typed result; no password logging
- 14:19 cookie is httpOnly + sameSite=lax, 7-day expiry

## Files touched
app/login/page.tsx, app/login/actions.ts, lib/auth/session-cookie.ts

## Decisions I made
- Used a server action rather than a route handler — design.md §3 implies it, not explicit

## Open questions
- none

## Known incomplete
- Error states are stubbed; step 4 not started
```

**`Decisions I made`** matters more than it looks. The archivist reads it to catch drift, and a
resumed crew reads it to avoid re-deciding. Anything you chose that wasn't spelled out belongs
here.

**`Known incomplete`** is what makes a halt safe. Be blunt — half-finished work described
accurately is recoverable; half-finished work described as done is a trap.

## Declare your sub-steps up front

Before you start building, break your slice into 3–7 sub-steps and write them down. Then
`[STATUS]` the planner with `n/m` as each completes.

This is what turns "it's building" into "it's 3/5 and moving." Without it the planner cannot
distinguish progress from a hang, and neither can the board.

## Heartbeat rules

`[STATUS]` the planner:

- when you start a slice (`assigned` → `building`)
- as each sub-step completes
- when you finish (`building` → `built`)
- when you start and finish a defect fix
- whenever you become blocked

Do not batch these. A status message is cheap; a planner guessing at your state is not.

## Honesty

If a test fails, say it failed and paste the relevant output. If you skipped something, say you
skipped it and why. If you're unsure whether your slice works, say that rather than reporting
`built`.

The tester will find the truth anyway. A slice reported honestly as broken costs one defect
cycle; a slice reported falsely as done costs the integration phase.
