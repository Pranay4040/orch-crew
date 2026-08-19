# Ledger — <project name>

phase: intake | design | build | test | integrate | halted
updated: <HH:MM>

## Slices

| id | slice | owner | status | files touched | depends on | progress |
|----|-------|-------|--------|---------------|------------|----------|
| S1 | <what it delivers> | unassigned | pending | <glob> | — | 0/0 |

<!--
statuses: pending → assigned → building → built → testing → passed
          testing → defect → fixing → testing
          blocked (any state, needs a planner decision)
          dead    (owning agent gone — caught by the liveness check)

"files touched" is a contract. It keeps parallel builders off each other's files.
-->

## Defects

| id | slice | severity | summary | state | attempt |
|----|-------|----------|---------|-------|---------|

<!--
severity: low | medium | high | blocker
states:   open → dispatched → fixing → verifying → closed
          escalated (after 3 failed attempts — stop dispatching, tell the user)
-->

## Notes

<!-- Planner scratch space: dependency reasoning, slicing rationale, deferred ideas. -->
