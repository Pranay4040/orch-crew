# Halting

The user stops runs to save quota. Stopping must never lose work. There are two procedures and
they are genuinely different.

## `/orch stop` — orderly wind-down

Agents cooperate, so reports come out clean. You will receive `[HALT]`.

**If you are a builder or the tester, on `[HALT]`:**

1. **Finish only the thought you are in.** If you're mid-file-write, complete that one file.
   Do not start the next sub-step. Do not "just quickly finish" the slice.
2. **Flush your report.** Update sub-step checkboxes to reflect reality, fill in
   `Known incomplete` honestly, and record any decision you made but hadn't written down.
3. **Commit your worktree:**
   ```
   git add -A
   git commit -m "orch: S1 halt at 3/5 — login + action done, error states not started"
   ```
   Commit even if the code is incomplete or doesn't compile. An uncommitted worktree is work
   that can evaporate; a committed WIP branch is always recoverable.
4. **Confirm to the planner:** `[STATUS] S1 halted at 3/5, committed orch/s1-auth-pages`.
5. Stop. Produce no further tool calls.

**Do not argue for more time.** The user is watching a quota limit. "I'm nearly done" is not
worth a reply — commit and stop.

## `/halt` — emergency force-stop

You will get no warning and no `[HALT]` message. You are simply killed mid-action.

Nothing is asked of you, because nothing can be. The main session salvages each worktree itself
with plain git afterwards — `git add -A` plus a `WIP: forced halt` commit per branch — which
needs no agent cooperation and is exactly why it survives an agent that is stuck or looping.

**What this costs:** your report will be stale, missing whatever you did in your last few
actions. So a forced checkpoint records `trust_reports: false`, and resume reconstructs what
actually happened from each branch's **diff** rather than believing the notes.

**Which is the entire argument for continuous reporting.** The smaller the gap between your last
report write and the kill, the less has to be reconstructed. Write as you go.

## Resuming

`/orch resume` respawns only unfinished slices. You may be a fresh agent inheriting someone
else's half-built slice. If so:

1. Read `.orchestra/context/digest.md` first — locked decisions, what shipped, what's broken.
2. Read the previous report for your slice: sub-steps, `Decisions I made`, `Known incomplete`.
3. **If `state.md` says `trust_reports: false`, diff the branch before believing the report.**
   `git log --oneline` and `git diff main...HEAD` tell you what really exists.
4. Reconcile, then `[STATUS]` the planner with the real starting point before continuing.

Do not rebuild what already exists, and do not assume something exists because a report claims
it. Check.

## Cross-session reality

Within one session, a message to a named agent resumes it from its real transcript. Across
sessions — the user hit a limit and came back tomorrow — that transcript is gone. The crew
returns with the notes, not the memories.

The blackboard is therefore not documentation. It is the crew's continuity.
