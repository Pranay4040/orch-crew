# Checkpoint — <project name>

halt: clean | forced
trust_reports: true | false
phase: <phase when halted>
halted_at: <HH:MM>

<!--
halt: forced  →  agents were killed without flushing. Reports are stale.
                 trust_reports MUST be false. Resume by diffing branches.
halt: clean   →  agents flushed and committed. Reports are trustworthy.
-->

## Slices at halt

| id | slice | owner | status | progress | branch | committed |
|----|-------|-------|--------|----------|--------|-----------|
| S1 | Auth pages | builder-1 | building | 3/5 | orch/s1-auth-pages | yes |

## Open defects

| id | slice | severity | state | attempt | summary |
|----|-------|----------|-------|---------|---------|

## Pending redirects not yet applied

<!-- User steers that arrived but hadn't reached the affected builders. These MUST be
     re-dispatched on resume or the redirect is silently lost. -->

## Resume instructions

1. Read `context/digest.md` — locked decisions, shipped, broken.
2. Respawn only slices not in `passed`.
3. If `trust_reports: false`, diff each branch before believing its report.
4. Re-dispatch every pending redirect above.
5. Do not rebuild anything already committed.
