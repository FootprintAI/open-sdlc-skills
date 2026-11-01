# Notice: verification is a post-deploy follow-up, never a merge gate

**Applies to every skill that flags, merges, deploys, verifies or reports
on a `needs-verification` item** — `/pm-sprint-delivery`,
`/team-sprint-cycle`, `/engineer-implement`, `/reviewer-pr`,
`/merge-steward`, `/devops-deploy`, `/qa-sprint-verify`, `/scrum-master`,
`/coordinator-status`, `/coordinator-sync`, and any skill that files a
single issue whose definition of done needs a running environment (for
example `/security-cve-iso-issue`).

## The rule

Some acceptance criteria cannot be proven by CI: a UI flow that only
exists once deployed, a migration against real-shaped data, a config,
infra or secret change, a third-party integration, a fleet re-check. Those
criteria are **verification items**. Three things are always true of them:

1. **They never gate a merge or a deploy.** The implementation merges when
   it is approved and green, and ships with the next deploy. Holding a PR
   until someone checks it on an environment inverts the order — the
   environment only carries the change *after* it merges and deploys
2. **Each one is recorded in exactly one place** before its PR merges:
   the **verification queue on the sprint umbrella** when the issue is in
   an open sprint, otherwise a **dedicated verification issue** (below).
   An item recorded nowhere is the one thing that *does* hold a PR,
   because it is a criterion nobody will ever check
3. **They are verified together, after the deploy, against one deployed
   commit** — one batched pass (`/qa-sprint-verify`), every result stamped
   with that commit, every failure filed as a defect. Never one issue at a
   time, never by deploying a branch to a shared environment

"Shipped, awaiting verification" is therefore a normal state, not a stuck
one and not a done one. It becomes an impediment only when the
post-deploy window is overdue (see *Reporting it*).

## What "recorded" means — the check at merge time

A verification item is recorded when all of these hold:

- the issue carries the `needs-verification` label and **Verify on
  `<env>`** steps in its body (route, input, expected result, any
  fixture, flag or seeded record — runnable by someone who never saw the
  diff, days later)
- the PR body carries the same steps under `## Verify on <env>`
- a **queue row** exists for it — on the umbrella's verification queue or
  on the open verification issue — and the PR body names where:
  `Verification: #<umbrella or verification issue>`

`/reviewer-pr` checks the first two and asks for the third; `/merge-steward`
holds a PR only while any of the three is missing. Pending verification
itself is never a reason to hold.

## Where the queue lives — the decision rule

| Situation | Queue |
|---|---|
| The issue is `Part of` an open sprint umbrella | that umbrella's **Verification queue** section |
| No open umbrella covers it: a one-off issue, a hotfix, an issue filed by a standalone skill, work that merges after its sprint closed | the repo's open **verification issue** — create one if none is open |

One open verification issue per repo at a time. Rows are appended to it
as items are flagged; a new one is opened only after the previous one
closed. When a sprint umbrella opens while a verification issue is open,
both stay: the batched pass reads *every* queue in the repo (umbrella
queues and the verification issue) against the same deployed commit —
rows are never copied between them.

## The verification issue

```markdown
Title: Verification queue: <repo or area> (opened <YYYY-MM-DD>)
Labels: epic + verification

## Purpose
Items that merged outside an open sprint and still need a check against
a deployed environment. Nothing here blocks a merge; everything here is
verified in one batched pass per deployed commit by `/qa-sprint-verify`.

**Deployed to dev:** _not yet_ — `<commit>` @ `<url>`, <date>
**Deployed to prod:** _n/a_

| Source | Verify | Expected | Env / version | Status | Verified on |
|--------|--------|----------|---------------|--------|-------------|
| #41 / PR #45 | upload `sample.pdf` on `/documents` | row appears within 10s, page count 3 | dev · needs `abc1234` | ⏳ awaiting deploy | |
| #52 / PR #53 | `uname -r` on every `gpu-pool` host | ≥ 6.8.0-45 on all hosts | prod · needs rollout | ⏳ awaiting deploy | |

Status legend: ⏳ awaiting deploy (merged, not on the env yet) ·
🔍 ready to verify (in the deployed commit) · ✅ verified on `<commit>` ·
❌ failed (defect #N) · ⚠️ not verified (<why>) · ⏭ waived (<who + why>)
```

Columns, one row per item:

- **Source** — the issue and the PR that shipped it, linked
- **Verify** — what to do, condensed from the issue's **Verify on** steps
  (the full steps stay on the issue and the PR)
- **Expected** — the observable result that counts as a pass
- **Env / version** — the environment (`dev`, `prod`, `dev + prod`) and
  the merge commit the environment must contain before the row is
  checkable (`needs <sha>`); for non-git targets such as a fleet, what
  must have happened (`needs rollout`)
- **Status** — the legend above, the same one the umbrella queue uses
- **Verified on** — the commit the result was observed on, plus the date
  and a link to the evidence; empty until a pass posts a result

The umbrella's verification queue uses the same columns and legend, so a
pass, a standup or a digest reads both the same way.

## Lifecycle — who writes what

| Event | Who | Writes |
|---|---|---|
| Flagged at scoping | `/pm-sprint-delivery` | label + steps on the issue, row on the umbrella queue |
| Flagged mid-implementation | `/engineer-implement` | label + steps on the issue, `## Verify on` + `Verification: #N` on the PR; the PM's next update adds the umbrella row, or the engineer appends the row to the verification issue itself |
| Filed by a standalone skill | that skill | label + steps on the issue, row on the verification issue (opened if none) — at filing time, not when the fix merges |
| Merged | `/merge-steward` | nothing on the queue — a merged PR is not a verified issue; the row stays ⏳ |
| Deployed | `/devops-deploy` | deploy record (commit + URL) on every queue home in the repo; ⏳ → 🔍 for rows whose commit is contained; the window comment |
| Verified | `/qa-sprint-verify` | results comment on every queue home its rows touched; each row ✅/❌/⚠️ with the commit; one defect per ❌ via `/qa-issue-report` |
| All rows terminal | `/qa-sprint-verify` (or the PM) | closing comment on the verification issue and close it; the umbrella's queue is cleared by `/pm-sprint-delivery close` |

A ❌ row does not reopen the source issue by itself: the defect is the
new issue, cross-linked both ways; re-scoping is the PM's call.

## Reporting it

- **Its own state.** Boards, standups, status reports and digests show
  *Awaiting verification* (⏳ or 🔍) as a distinct state between *Merged*
  and *Done* — never folded into *Blocked*, *Stalled* or *Not done*
- **Overdue thresholds** (defaults; tune per team and say so when
  applied): a ⏳ row older than **2 days** with no deploy carrying it, or
  a 🔍 row on a deployed environment for more than **1 day** with no pass
  run, is a **verification backlog** impediment — owned by DevOps (the
  missing deploy) or QA (the missing pass). One batched deploy or one
  pass clears the whole backlog; it is never routed as N per-issue
  deploys
- **Never per-issue.** An urgent item is answered by the next deploy of
  the whole batch and the next pass, not by syncing its branch to a
  shared environment

## Containarium trackers

`verification` and `needs-verification` are not in a connection's default
label allow-list (`scope:*`, `model:*`, `agent:needs-approval`). Add them
with the other labels (`/devops-containarium-env` capability 9); until
then file with allowed labels, label afterwards, and report the rejection
— never drop the label silently.
