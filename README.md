# orch — a multi-AI-agent orchestrator for Claude Code

**orch is a multi-AI-agent orchestrator.** You give it a plan; it runs a crew of AI agents that
build the software together, talking to each other rather than reporting to you one at a time.

`/orch <plan>` spawns seven Claude Code subagents: a **designer**, a **planner**, three
**builders**, a **tester**, and an **archivist**. They message each other directly, coordinate
through a shared file blackboard, and you can redirect them mid-build or stop the whole thing
without losing work.

Everything is Claude Code native — slash commands, subagents, skills, `SendMessage`, git worktrees.
**No API keys, no services, nothing to install or keep running.**

> **Read the cost section before you use this.** It is genuinely expensive, and for small projects
> it is the wrong tool. That section is not boilerplate.

---

## Install

```bash
git clone https://github.com/<you>/orch-crew
cd orch-crew
cp -r commands agents skills ~/.claude/
```

Optionally append `CLAUDE.md.example` to your `~/.claude/CLAUDE.md`.

**Restart Claude Code.** Command, agent, and skill registries load at session start, so nothing is
visible until you do.

---

## Use

```
/orch build a unit converter — length, weight, temperature, with a category dropdown
```

| Command | What it does |
|---|---|
| `/orch <plan>` | Start a run |
| `/orch status` | Render the ledger — no agents woken |
| `/orch board` | Republish the dashboard artifact |
| `/orch stop` | Orderly wind-down: agents flush, commit, then stop |
| `/orch resume` | Respawn only unfinished slices |
| `/halt` | **Emergency force-stop.** Kills first, salvages with git after |

You can also just talk to it mid-run. "Make the UI dark mode" goes to the planner, which scopes
which slices are affected and redirects only those builders. The rest never stop.

---

## How it works

### The blackboard

Subagents share **no context** with each other. Everything they need is on disk, in
`.orchestra/`:

```
.orchestra/
  brief.md         your plan, verbatim
  design.md        the spec
  tasks.md         THE LEDGER — slices, statuses, defects
  decisions.md     append-only log of your redirections
  reports/*.md     per-agent running notes
  context/
    digest.md      the crew's memory — one page, rewritten in place
  state.md         halt checkpoint
```

**Single-writer ownership.** Every file has exactly one writing agent. Builders never touch the
ledger — they report, the planner records. This is the entire concurrency model, and it is why
parallel agents don't silently clobber each other.

### The message protocol

Fixed tags so routing is mechanical: `[STATUS]` `[BLOCKED]` `[DEFECT]` `[DISPATCH]` `[REDIRECT]`
`[VERIFY]` `[DONE]` `[ESCALATE]` `[COMPACT]` `[DRIFT]` `[BOARD]` `[HALT]`.

**The planner is the hub.** Notably, the tester never messages a builder directly — every defect
goes through the planner, which triages code-bug vs design-flaw, decides the fix, and dispatches
it. That keeps the ledger complete and stops builders fixing symptoms.

### The defect loop

```
tester finds bug → [DEFECT] planner → triage
   ├─ code bug    → [DISPATCH] the owning builder with the fix
   └─ design flaw → amend the spec FIRST, then dispatch
        → builder fixes → [VERIFY] tester → pass, or attempt++ and loop
```

**Loop guard:** after 3 failed attempts on one defect the planner stops and escalates to you. Without
it, a bug the crew can't solve burns tokens forever.

### The archivist — why quality doesn't decay

Long multi-agent runs get dumber. Builder-2 starts slice 5 an hour in, and the design reasoning,
your mid-run redirect, and the two defects that reshaped the data model are all far outside its
context. So it re-derives, guesses, or contradicts shipped work.

The archivist keeps `context/digest.md` — **one page, rewritten in place, never appended** —
holding locked decisions *with their reasons*. Every agent re-reads it at boundaries. It also does
drift detection: when an agent contradicts a locked decision, that surfaces immediately, while it's
one slice of rework instead of three.

### Stopping without losing work

Two speeds, and the difference is who does the saving:

- **`/orch stop`** — agents flush reports and commit their own worktrees, then stop. Clean notes.
- **`/halt`** — kills everything instantly, *then* the main session salvages each worktree with
  plain git. Needs no agent cooperation, so it works on an agent that's stuck or looping. Reports
  will be stale, so the checkpoint records `trust_reports: false` and resume reconstructs from
  branch diffs instead of believing the notes.

---

## Cost — read this part

This is the honest version. The system is expensive and the expense is not evenly distributed.

### Measured on a real run

Building a **calculator app** — small, self-contained, the kind of thing one session handles fine:

- **Phase 1 alone cost ~57k tokens and ~4 minutes** — designer plus planner, before a single line
  of code was written.
- The planner's ledger grew to **274 lines**, which all six agents then re-read on every cycle.

### Where the credits actually go

| Driver | Why it's expensive |
|---|---|
| **Model tier** | planner, designer, and tester default to Opus. Three premium agents deliberating. |
| **Ledger re-reads** | The planner writes long notes, then *every* agent re-reads them, every cycle. Compounding. |
| **Deliberation depth** | The planner and designer genuinely debate edge cases. Great for a real product, absurd for a calculator. |
| **Mid-run redirects** | Changing your mind can double a slice's scope and force a re-check of every affected file. |
| **Sub-step chatter** | A `[STATUS]` per sub-step across three builders adds up. |
| **Defect loops** | Each cycle is builder + tester + planner. The loop guard caps it at 3, but 3 is still 3. |

### When NOT to use this

**If the job fits in one normal Claude Code session, don't use `/orch`.** You pay all the
coordination overhead and get none of the benefit. A calculator, a landing page, a script, a
refactor of a few files — just ask Claude directly.

| Your work | Use |
|---|---|
| Fits in one session | Plain Claude Code. No crew. |
| Medium, a few days, several independent areas | `/orch` with fewer builders and cheaper models |
| Large, multi-session, must survive halts and keep decisions straight | Full `/orch` — this is what it's for |

### Cutting the cost

1. **Downgrade models** in the agent frontmatter. `model: sonnet` for planner, designer, and tester
   is the single biggest saving.
2. **Fewer builders.** Delete `builder-2.md` / `builder-3.md` for a one-at-a-time crew — much
   cheaper and far easier to steer.
3. **Drop the archivist** on short runs. It only pays for itself when the run is long enough to
   decay.
4. **Write a specific brief.** The crew amplifies whatever you paste. Vague briefs cause expensive
   deliberation and rework.
5. **`/halt` early and often.** It's designed to be cheap and safe; use it the moment a run looks
   wrong.

---

## Advantages

- **Real parallelism** — three slices build at once in isolated worktrees.
- **Steer without stalling** — a redirect reaches only the affected builders; the rest keep working.
- **Bugs round-trip without you** — the defect loop closes on its own; you hear about it after 3
  failed attempts.
- **Quitting is free** — halt mid-build, resume tomorrow in a new session.
- **Conventions decided once** — the designer writes the project's `CLAUDE.md` before any code, so
  three parallel builders produce code that looks like one person wrote it.
- **You stop reading everything** — a ledger and a one-page digest instead of a transcript.

## Limits and known sharp edges

- **Cost.** See above. It's the main one.
- **The planner is a single point of failure** — everything routes through it by design.
- **Three builder slots is the ceiling.** Agent names are latest-wins, so slots are fixed files.
- **Parallel worktrees conflict on shared files.** Mitigated by slicing along file boundaries, not
  eliminated.
- **Cross-session resume loses transcripts.** A resumed crew is a *fresh* spawn rehydrated from
  `.orchestra/`. It comes back with the notes, not the memories — which is why continuous reporting
  is a protocol rule.
- **The dashboard cannot poll your filesystem.** No artifact runtime capability allows it, so the
  board refreshes by republishing, not on a timer. It carries a "last updated" stamp for that reason.
- **A vague brief produces a vague build.**

## Requirements

- Claude Code
- **A git repo with at least one commit.** `git worktree add` needs a valid `HEAD` — in a fresh
  `git init` directory with no commits, isolation silently fails and all builders share one tree.
  `/orch` checks for this and commits a scaffold if needed, but it's worth knowing.

## License

MIT
