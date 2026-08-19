---
name: orch-protocol
description: >
  Operating protocol for /orch build crews — blackboard layout, message tags, file ownership,
  report format, digest discipline, and the two halt procedures. Load at the start of any orch
  agent's turn, before reading the brief or writing anything.
---

# orch-protocol

You are one role in a multi-agent build crew coordinated by `/orch`. This is the shared
operating manual. The other agents are relying on you following it exactly.

## The one fact that breaks everything if you forget it

**You share no context with any other agent.** Your plain-text output is invisible to them.
Two consequences, and both are absolute:

1. **To communicate you MUST call `SendMessage`.** Writing something in your response reaches
   nobody. If you decided something the crew needs, send it.
2. **Anything the crew needs later MUST be on disk, in the file you own, written as you go.**
   You can be killed without warning — the user halts runs to save quota. An agent that saves
   its notes for the end leaves nothing behind when that happens.

## Where everything lives

The blackboard is `.orchestra/` at the project root. Read
[references/blackboard.md](references/blackboard.md) for the full layout, file formats, and the
**single-writer ownership table** — which file you are allowed to write is not negotiable, and
writing someone else's file destroys their work.

The short version:

| You need | Read |
|---|---|
| What we're building | `.orchestra/brief.md` |
| Current distilled truth | `.orchestra/context/digest.md` |
| Design + conventions | `.orchestra/design.md`, project `CLAUDE.md` |
| Who's doing what | `.orchestra/tasks.md` |

## Read the digest at every boundary

`.orchestra/context/digest.md` is about one page and is the crew's memory. It exists because
long runs decay — the reasoning behind an early decision falls out of everyone's context, and
agents start contradicting work that already shipped.

**Re-read it:**

- **builder** — before starting a slice, after any `[REDIRECT]`, before any defect fix
- **tester** — before verifying a slice
- **planner** — before slicing, and before every dispatch
- **designer** — before amending the design or the project `CLAUDE.md`

Its **Locked decisions** section is settled. Each entry carries its reason. You may not quietly
re-open one — if you believe a locked decision is wrong, `[BLOCKED]` the planner and wait.

## How you talk

A fixed tag vocabulary so routing is mechanical. Full table with payload shapes and worked
examples: [references/messages.md](references/messages.md).

The routing rules that matter most:

- **Everything defect-shaped goes through the planner.** The tester never messages a builder
  directly, ever. The planner triages, decides the fix, and dispatches.
- **The planner is the hub.** Status, blocks, and completions go to it. It owns the ledger.
- **Only background agents can reach `main`** (the user). Use it sparingly — that's the user's
  attention.

## How you report

Write your own report file continuously. Format and heartbeat rules:
[references/reporting.md](references/reporting.md).

Non-negotiable: **`[STATUS]` the planner on every state change**, not just at completion. Declare
your sub-steps up front (`3/5`) so progress is visible rather than a black box.

## When you're told to stop

Two procedures, and they are different. Read
[references/halting.md](references/halting.md) before acting on a `[HALT]`.

The essentials: on `[HALT]`, finish only the thought you're in, flush your report, commit your
worktree, and confirm. Do not start anything new. Do not argue for more time.

## Templates

Copy these rather than inventing structure: [templates/tasks.md](templates/tasks.md),
[templates/state.md](templates/state.md), [templates/report.md](templates/report.md).

## Ground rules

1. **Stay in your slice.** Editing files outside your assignment causes merge conflicts nobody
   asked for. If your slice genuinely needs a shared file changed, `[BLOCKED]` the planner.
2. **Never edit another agent's file.** See the ownership table.
3. **Don't guess at a decision — ask.** `[BLOCKED]` costs one message. A wrong guess costs a
   slice of rework, and the archivist will catch it as drift anyway.
4. **Follow the project `CLAUDE.md`.** It exists so three parallel builders produce code that
   looks like one person wrote it. Matching the existing style beats your personal preference.
5. **Report honestly.** If a test fails, say it failed. If you skipped something, say so. A
   silently broken slice is far more expensive than a slice reported broken.
