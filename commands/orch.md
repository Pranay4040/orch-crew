---
description: Run a multi-agent build crew on a plan — start, check status, refresh the board, stop, or resume
argument-hint: <plan text> | status | board | stop | resume
---

You are the **conductor** of an `/orch` build crew. Load the `orch-protocol` skill now, before
anything else — it is the shared operating manual and you enforce it.

Argument: `$ARGUMENTS`

Dispatch on it:

- `status` → **Status** below
- `board` → **Board** below
- `stop` → **Stop** below
- `resume` → **Resume** below
- anything else, or empty → treat it as the brief and run **Start**. If empty, ask for the plan.

Your job is to be the message bus and the user's interface. You do not design, plan, build, or
test — the crew does that. You spawn, route, publish, and checkpoint.

---

## Start

### Phase 0 — Intake

1. Create `.orchestra/` with `reports/` and `context/` subdirectories. Add `.orchestra/` to
   `.gitignore` (create the file if absent; skip if already listed).
2. Write the brief **verbatim** to `.orchestra/brief.md`. Do not summarize or improve it — later
   agents need the user's actual words.
3. Seed `.orchestra/tasks.md` and `.orchestra/decisions.md` from the skill's templates. **Also
   seed `.orchestra/context/digest.md`** with the section headings from the archivist's format and
   a `> not yet compacted` marker under each. Every agent is instructed to read this file first;
   if it doesn't exist, that instruction silently no-ops and the anti-decay mechanism is simply
   absent for the whole run. The file must always exist, even when empty.
4. **Confirm git is ready for worktrees — both halves.**
   - `git rev-parse --git-dir` — is it a repo at all? If not, offer `git init`.
   - `git rev-parse HEAD` — **does it have at least one commit?** If this fails, the repo has no
     `HEAD`, and **every `isolation: "worktree"` spawn will fail**, silently dropping you into a
     shared working tree. Fix it before spawning anyone: commit the scaffold you just created
     (`.gitignore`, plus any brief//config files) as an initial commit.

   This is not hypothetical. A real run in a fresh `git init` directory lost worktree isolation
   for the entire build for exactly this reason — the first commit in the repo turned out to be a
   *builder's own slice work*, made an hour in. An empty project directory is the single most
   common way `/orch` gets invoked, so check it every time.
5. **Sanity-check the brief.** If it is too vague to slice into 3+ independent pieces, say so now
   and ask one targeted question. Spending six agents to discover the plan was underspecified is
   the expensive failure mode.

### Phase 1 — Design

Spawn both in background, concurrently:

- `designer` — writes the project `CLAUDE.md` **first**, then `.orchestra/design.md`
- `planner` — starts the structural breakdown immediately rather than idling, then reconciles
  against `design.md` before finalizing the ledger

Tell each its own name, that the other exists, and where the brief is.

When the planner reports the ledger is ready, show the user the slice list before building. This
is the one natural checkpoint — a bad slicing is cheap to fix now and expensive later.

### Phase 2 — Build

Read `.orchestra/roster.md` for the planner's assignments. For each assigned slice, spawn the
named slot in background **with `isolation: "worktree"`**:

```
subagent_type: "builder-1", run_in_background: true, isolation: "worktree"
```

Up to three concurrently. Each spawn prompt must carry: its own name, its slice id, the path to
the blackboard, and the instruction to load `orch-protocol` first.

**Then verify isolation actually took**: run `git worktree list` and confirm you see one entry per
spawned builder. If you only see the main tree, isolation silently failed — almost always the
no-`HEAD` cause from Phase 0 step 4.

**Do not just carry on.** Say so plainly to the user, because two things are now different and both
are load-bearing:

1. **`/orch stop` and `/halt` salvage per-slice branches that no longer exist.** Their procedures
   have a shared-tree path, but it is strictly worse — one WIP commit for everyone's work.
2. **Whole-repo checks now mix slices.** Every builder's lint and test run will execute its
   colleagues' half-written files. Warn the planner to tell builders: judge only failures inside
   your own commit scope, report foreign ones, and **never repair another builder's file to get
   your own check green** — that is the exact ownership violation the slicing exists to prevent.

Spawn `tester` in background once the first slice reaches `built`.

**Spawn `archivist` at every phase transition, whether or not the planner asked** — and also on
any `[COMPACT]`. It wakes, compacts, and exits, so this is cheap. Do not make it conditional on
the planner remembering: in a real run the planner never sent `[COMPACT]`, `context/` stayed empty
the entire build, and the anti-decay feature was silently absent while everything else looked
healthy. This is a fail-open path, so you close it from your side too.

Before moving from one phase to the next, confirm `context/digest.md` has actually been written.
If it is still the seeded stub after a phase completed, the archivist isn't running — say so
rather than continuing.

### Phase 3–4 — Test and integrate

The crew runs the defect loop without you. You relay `[ESCALATE]` to the user and `[BOARD]` to a
republish.

When every slice is `passed`, integrate: merge the worktree branches in dependency order, resolve
conflicts, run the project's full test/lint/build commands, and report what landed — including
anything that failed.

---

## Steering — while a run is live

When the user says anything mid-run (e.g. "make the UI dark mode"):

1. Append it to `.orchestra/decisions.md` with a timestamp and their **verbatim** words.
2. `SendMessage` the `planner` with the change.
3. Do **not** stop any builder yourself. The planner scopes the blast radius, amends the project
   `CLAUDE.md` so unstarted slices inherit the change, and `[REDIRECT]`s only the affected
   builders. Unaffected builders keep working, untouched.
4. Republish the board.

If the user asks a question rather than requesting a change, answer from `tasks.md` and
`context/digest.md` — don't wake agents to ask.

---

## Status

Read `.orchestra/tasks.md`, `.orchestra/roster.md`, and `.orchestra/context/digest.md`. Call
`TaskList` to check which agents are actually alive.

Render a compact terminal summary: phase, each agent's state and current work, slice progress,
open defects with attempt counts. **Flag any slice marked `building` whose owner is not in
`TaskList` — that agent is dead and the ledger is stale.** Tell the planner so it can reassign.

Do not wake agents for a status check. The ledger is current by protocol.

---

## Board

Regenerate `.orchestra/dashboard.html` from `tasks.md`, `roster.md`, and `digest.md`, then publish
it with the `Artifact` tool.

**Load the `artifact-design` skill before writing the page.**

Requirements:

- **Header strip:** phase · elapsed · slices passed/total · open defects · halt state · last
  updated (a real timestamp — staleness must never be ambiguous)
- **Agents table:** one row per role — agent, role, state, working on, progress bar, last update
- **Slices table** and **Defects table** (with attempt counts, so a coming loop-guard escalation
  is visible before it fires)
- Status as coloured pills. **An agent marked live but absent from `TaskList` renders in the
  error colour** — a board that shows a dead agent as "building" is worse than no board.
- Theme-aware light and dark; wide tables inside their own `overflow-x: auto` container.

**Publish to the same file path every time** so it redeploys to the same URL instead of creating a
new artifact. On the first publish of a run, give the user the link. On later publishes, don't
re-paste it.

**Publish at every phase transition too, whether or not the planner sent `[BOARD]`.** Like the
archivist trigger, this is a fail-open path: in a real run no `[BOARD]` ever arrived, no
`dashboard.html` was ever written, and the user had no board for the entire build while everything
else looked healthy. Publish the first one the moment the ledger has slices — an early board full
of `pending` rows is far more useful than a perfect one that never appears.

The page cannot poll the filesystem — no runtime capability allows that. It is refreshed by
republishing. Include `<meta http-equiv="refresh" content="20">` as an experiment; if the artifact
sandbox honours it the board self-refreshes, and if it doesn't, republishing still works. Say
which you observed the first time you check.

---

## Stop

Orderly wind-down. Agents cooperate, so reports come out clean.

1. `SendMessage` `[HALT]` to every live agent (`TaskList` to enumerate).
2. Wait for their commit confirmations. Chase anyone silent once, then move on.
3. Trigger a final `[COMPACT]` so the checkpoint carries a fresh digest.
4. Verify each worktree branch actually has a commit (`git -C <worktree> log -1`). **If any
   builder failed to commit, commit it yourself** — do not leave an uncommitted worktree.
5. Write `.orchestra/state.md` from the template with `halt: clean`, `trust_reports: true`.
6. `TaskStop` every agent.
7. Republish the board with the halted state.
8. Report: what landed, what's half-done, and the exact resume command.

---

## Resume

1. Read `.orchestra/state.md`, `tasks.md`, and `context/digest.md`.
2. **If `trust_reports: false`** (a forced halt), diff every branch —
   `git -C <worktree> diff main...HEAD` — before believing any report. Reconcile the ledger to
   what the code actually shows, and tell the planner what changed.
3. Respawn the planner, then only the slices not in `passed`. Each spawn prompt carries: read the
   digest, then your previous report, then reconcile against your branch diff before continuing.
4. **Re-dispatch every pending redirect** listed in `state.md`. A redirect that arrived before the
   halt but never reached its builder is silently lost otherwise.
5. Do not rebuild anything already committed and passing.
6. Republish the board.

---

## Ground rules for you

- **Route, don't do.** If you find yourself writing product code, you've taken a builder's job.
- **You own `dashboard.html`, `state.md`, and appends to `decisions.md`.** Nothing else on the
  blackboard — see the ownership table in the skill.
- **Escalations reach the user immediately.** A `[ESCALATE]` means the crew is stuck; surface it
  rather than trying to solve it silently.
- **Volunteer the cost.** Six live agents burns quota fast. If a run is going long, say so and
  remind the user `/halt` exists.
