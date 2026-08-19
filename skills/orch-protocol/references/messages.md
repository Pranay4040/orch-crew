# Message protocol

Every message starts with a tag in square brackets. Tags make routing mechanical instead of the
crew chatting in prose. Anything untagged risks being misread.

`SendMessage` takes `to` (an agent name) and `message`. Names: `planner`, `designer`,
`builder-1`, `builder-2`, `builder-3`, `tester`, `archivist`, and `main` (the user's
conversation — background agents only).

## The tags

| Tag | From → To | Payload |
|---|---|---|
| `[STATUS]` | builder/tester → planner | slice id, new state, sub-step `n/m`, one line of detail |
| `[BLOCKED]` | any → planner | slice id + the precise decision needed, with the options you see |
| `[DEFECT]` | tester → **planner** | slice, severity, repro steps, expected vs actual |
| `[DISPATCH]` | planner → builder | defect id + the fix directive (what to change, not just what's wrong) |
| `[REDIRECT]` | planner → builder(s) | what changed + what to do differently now |
| `[VERIFY]` | planner → tester | defect id ready for re-test |
| `[DONE]` | builder → planner | slice or defect id complete + where the work is |
| `[ESCALATE]` | planner → main | needs the user: design flaw, or a defect stuck after 3 attempts |
| `[COMPACT]` | planner → archivist | re-read the blackboard and refresh the digest |
| `[DIGEST]` | archivist → planner | digest refreshed; re-read before the next dispatch |
| `[DRIFT]` | archivist → planner | agent X contradicted locked decision Y |
| `[BOARD]` | planner → main | slice-level ledger change — republish the board |
| `[HALT]` | main → all | flush reports, commit, and stop |

## Routing rules

**The tester never messages a builder.** Not for a small bug, not to save a round trip, never.
Defects go to the planner, which triages (code bug vs design flaw), records the defect in the
ledger, and dispatches. This is what keeps the ledger complete and the planner's view
authoritative.

**The planner is the hub.** If you're not sure who to tell, tell the planner.

**`main` is the user's attention.** Only `[ESCALATE]` and `[BOARD]` go there. Do not send
progress chatter to `main` — that's what the board and the ledger are for.

## Writing good messages

Be specific enough to act on without a follow-up question. The recipient cannot see your
context, your files, or your reasoning.

**Bad:**
```
[STATUS] making progress on the auth thing
```

**Good:**
```
[STATUS] S1 building 3/5 — login form + server action done, session cookie next
```

**Bad:**
```
[DEFECT] the cart is broken
```

**Good:**
```
[DEFECT] S2 high — cart total returns NaN when the cart is empty.
Repro: open /cart with no items. Expected: "$0.00". Actual: "$NaN".
Source: lib/cart/total.ts:14 divides by items.length without a zero guard.
```

**Bad:**
```
[BLOCKED] not sure how to do this
```

**Good:**
```
[BLOCKED] S3 — checkout needs a persisted order id, and the design doesn't say where.
Options I see: (a) localStorage, (b) a server action writing to the existing sqlite db,
(c) URL param. Leaning (b) for durability. Confirm before I build it.
```

A `[DISPATCH]` must say what to change, not merely restate the bug — the planner has already
decided the fix, and the builder shouldn't have to re-derive it:

```
[DISPATCH] D1 in S2 — add a zero guard in lib/cart/total.ts:14; return 0 for an empty cart
rather than dividing. Locked decision D-07 says totals exclude tax, so don't touch the tax
path while you're in there.
```
