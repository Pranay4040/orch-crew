---
description: Emergency force-stop of a running /orch crew — kills every agent immediately, then salvages the work with git
---

**Emergency force-stop.** The user is out of quota, or the run is going wrong. Act immediately —
do not investigate first, do not ask for confirmation, do not finish anything in progress.

This is the opposite order from `/orch stop`. There, agents flush and commit themselves. Here you
kill first and salvage afterwards, because salvage needs no agent cooperation — which is exactly
what makes it work on an agent that is stuck, looping, or ignoring messages.

## Procedure

**1. Kill everything, now.**

`TaskList` to enumerate live agents, then `TaskStop` each one. Do not send `[HALT]`. Do not wait
for replies. Do not give anyone a chance to finish.

**2. Salvage the work yourself.**

First run `git worktree list` to see which layout you are in — do not assume.

**If there are per-slice worktrees**, salvage each one:

```bash
git -C <worktree-path> add -A
git -C <worktree-path> commit -m "WIP: forced halt — <slice id> state unverified"
```

**If there is only the main tree** (isolation failed at spawn — a common case in a repo that had
no commits when the run started), everyone was sharing one checkout. Salvage it as a single
commit:

```bash
git add -A
git commit -m "WIP: forced halt — shared tree, multiple slices, state unverified"
```

Say clearly in your report that this commit **mixes several builders' work** and cannot be
separated per slice. That materially degrades resume: there are no per-slice branches to diff, so
the ledger's last known statuses are the only guide, and they are stale by definition after a
forced halt.

Commit either way even if nothing compiles. Broken committed code is recoverable; uncommitted code
is gone. Nothing to commit is not an error — note it and move on.

**3. Write the checkpoint.**

`.orchestra/state.md` from the template, with:

```
halt: forced
trust_reports: false
```

`trust_reports: false` is the important field. Agents died mid-action, so their reports are
missing whatever they did last. Resume must reconstruct from branch diffs rather than believing
the notes — that flag is what tells it to.

Fill in each slice's last *known* status from `tasks.md`, and copy any pending redirects that
hadn't reached their builders. Mark statuses as unverified — they are.

**4. Report honestly.**

Tell the user:

- which agents were killed, and what each was mid-way through
- which branches got a WIP commit, and which had nothing to save
- **what is now uncertain** — name it plainly rather than implying a clean stop
- the resume command

Do not republish the board unless it's free to do so; the user is conserving quota, which is why
they ran this.

## If no run is active

Say so in one line and stop. Don't scaffold anything, don't investigate, don't offer alternatives.
