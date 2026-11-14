# Notice: verification is a post-deploy follow-up, never a merge gate

**Applies to every skill that flags, merges, deploys, verifies or reports
on a `needs-verification` item** — `/pm-sprint-delivery`,
`/team-sprint-cycle`, `/engineer-implement`, `/reviewer-pr`,
`/merge-steward`, `/devops-deploy`, `/qa-sprint-verify`, `/release-cut`,
`/release-verify`, `/scrum-master`,
`/coordinator-status`, `/coordinator-sync`, and any skill that files a
single issue whose definition of done needs a running environment (for
example `/security-cve-iso-issue`).

## The rule

Some acceptance criteria cannot be proven by CI: a UI flow that only
exists once deployed, a migration against real-shaped data, a config,
infra or secret change, a third-party integration, a fleet re-check. Those
criteria are **verification items**. Verification follows the **release**,
not the sprint: an item is checked on the released version that carries
its merge, whichever sprint scoped it, or none. Three things are always
true of them:

1. **They never gate a merge or a deploy.** The implementation merges when
   it is approved and green, and ships with the next deploy. Holding a PR
   until someone checks it on an environment inverts the order — the
   environment only carries the change *after* it merges and deploys
2. **Each one is recorded in exactly one place** before its PR is
   approved: a row on the **release verification ticket** for the next
   release that will carry the merge (below). `/reviewer-pr` checks this
   at review time and requests changes for an item recorded nowhere,
   because that is a criterion nobody will ever check. `/merge-steward`
   reads CI and reviews only — it never holds a PR for verification
   state, recorded or not, and it never writes a row; its one write is
   advancing the issue's label at merge (*The label lifecycle*). Anything
   that still slips through is caught at release time: `/release-verify`
   finds every merged `awaiting-verification` issue that has no row and
   gets it one before the post-release pass
3. **They are verified together, after the release, against one deployed
   commit** — one batched pass (`/qa-sprint-verify`), triggered by
   `/release-verify` once `/release-cut` has tagged and the tag's commit
   is running, every result stamped with that commit, every failure filed
   as a defect. Never one issue at a time, never by deploying a branch to
   a shared environment. A failure does not move the tag: the fix ships as
   the next patch release

"Shipped, awaiting verification" is therefore a normal state, not a stuck
one and not a done one. It becomes an impediment only when a released
version sits unverified (see *Reporting it*).

Keying verification to the version is the common shape in long-lived
projects — Mozilla advances a per-version status flag from fixed to
verified, GitLab keeps a verification workflow label on an issue until it
is verified, Go rolls a release's unfinished items to the next milestone —
and none of them keys it to a sprint.

## What "recorded" means — the check before approval

A verification item is recorded when all of these hold:

- the issue carries the `needs-verification` label and **Verify on
  `<env>`** steps in its body (route, input, expected result, any
  fixture, flag or seeded record — runnable by someone who never saw the
  diff, days later)
- the PR body carries the same steps under `## Verify on <env>`
- a **row** exists for it on the next release's verification ticket, and
  the PR body names the ticket: `Verification: #<release verification
  ticket>`

`/reviewer-pr` checks all three and requests changes while any is missing;
that review verdict is the only hold. `/merge-steward` never holds a PR for
verification — not for a missing record, not for a pending check. The
backstop for an item that merged unrecorded is `/release-verify`, which
finds it by its label and asks for its row before the pass.

The pointer on the PR names the ticket the row was first written on. A row
that later rolled forward is found by its **Source** on the ticket the
roll comment names.

## The label lifecycle

The label on the **issue** advances with the work; it is never on the PR
and never holds one.

| State | Label | Set by |
|---|---|---|
| Flagged; PR open or not yet opened | `needs-verification` | whoever flags the item |
| PR merged, not yet verified | `awaiting-verification` (replaces `needs-verification`) | `/merge-steward`, at the merge |
| Row is ✅, ❌ or ⏭ | none — the label is removed | whoever writes the result: `/qa-sprint-verify` for ✅/❌, `/release-verify` recording a human's ⏭ |

A ❌ also files a defect; the defect is its own issue with its own label
and its own row. A ⚠️ row keeps `awaiting-verification`: nothing was
decided.

The label is the queryable backstop. **Every issue labelled
`awaiting-verification` must have a row on an open release verification
ticket**; one that has none is an unrecorded item, and `/release-verify`
finds it with one query and gets it a row. It also advances the label on
an issue whose PR merged outside `/merge-steward`, so a merge by hand
does not hide an item.

## Where the queue lives — one home, keyed by release

There is one queue home: the **release verification ticket**. One ticket
per pending tag per repo.

| Ticket | What it holds |
|---|---|
| The **next ticket** — no tag yet, at most one open | every row flagged since the last cut. New rows are always written here |
| A **pending ticket** — its tag is cut, its pass is not finished | the rows whose merge commit is in that tag |

- **No gap.** `/release-cut` opens the next ticket in the same step that
  tags, so a merge always has a home. A repo with no open next ticket
  (first use) gets one from whoever flags the first item, from the
  template below
- **Membership is by ancestry.** A row belongs to a tag's ticket when its
  merge commit is an ancestor of the tag and not of the previous tag —
  never because of which ticket was open when it was flagged. At
  `released`, `/release-verify` moves every row that is not in the tag
  (unmerged, or merged after the tag's commit) forward to the next
  ticket, whole, and says so in one comment on each ticket
- **A patch release gets its own ticket.** When `v1.3.1` is cut while the
  next ticket is already named for `v1.4.0`, `/release-cut` opens
  `Release verification: v1.3.1` and the same ancestry rule fills it
- **A sprint umbrella carries no queue.** It links the ticket; a standup,
  a status report or a digest reads a sprint's items by filtering the
  ticket's **Sprint** column

## The release verification ticket

```markdown
Title: Release verification: <next tag or 'next release'> (opened <YYYY-MM-DD>)
       (retitled "Release verification: vX.Y.Z" when the tag is cut)
Labels: epic + release

## Release
**Since:** `v1.2.3` · **Tag:** _not cut_ · **Tag commit:** _none yet_
**Deployed to dev:** _not yet_ — `<commit>` @ `<url>`, <date>
**Deployed to prod:** _n/a_
**Post-release verification:** _not started_

## Verification rows
Every item that needs a check against a deployed environment and ships
with this release. Nothing here blocks a merge or the cut; everything
here is verified in one batched pass on the released version by
`/qa-sprint-verify`.

| Source | Sprint | Verify | Expected | Env / version | Status | Verified on |
|--------|--------|--------|----------|---------------|--------|-------------|
| #41 / PR #45 | #17 | upload `sample.pdf` on `/documents` | row appears within 10s, page count 3 | dev · needs `abc1234` | ⏳ awaiting deploy | |
| #52 / PR #53 | — | `uname -r` on every `gpu-pool` host | ≥ 6.8.0-45 on all hosts | prod · needs rollout | ⏳ awaiting deploy | |

Status legend: ⏳ awaiting deploy (merged, not on the env yet) ·
🔍 ready to verify (in the deployed commit) · ✅ verified on `<commit>` ·
❌ failed (defect #N) · ⚠️ not verified (<why>) · ⏭ waived (<who + why>)
```

`/release-verify` owns the ticket and adds its own sections below the
rows (open questions, ship decisions, post-release notes — see that
skill). The ticket is found by its `release` label and told apart by its
**Tag** field, never by its title.

Columns, one row per item:

- **Source** — the issue and the PR that shipped it, linked (the issue
  alone until a PR exists)
- **Sprint** — the sprint umbrella the issue is `Part of` (`#17`), or `—`
  for a one-off issue, a hotfix or an issue filed by a standalone skill
- **Verify** — what to do, condensed from the issue's **Verify on** steps
  (the full steps stay on the issue and the PR)
- **Expected** — the observable result that counts as a pass
- **Env / version** — the environment (`dev`, `prod`, `dev + prod`) and
  the merge commit the environment must contain before the row is
  checkable (`needs <sha>`, filled in once the PR has merged; `needs
  merge` until then); for non-git targets such as a fleet, what must have
  happened (`needs rollout`)
- **Status** — the legend above
- **Verified on** — the commit the result was observed on, plus the date
  and a link to the evidence; empty until a pass posts a result

**A mid-cycle dev pass is optional.** If one runs before the tag, its
results go on the same rows, in **Verified on**, stamped with the commit
they were observed on (`pre-release: ✅ on 7777777`). They do not make a
row terminal and do not move the label: the post-release pass still
verifies every row on the released version. Evidence observed on the very
commit the tag points at may be reused by that pass instead of re-run. A
pre-release ❌ files its defect straight away, like any other.

## Lifecycle — who writes what

| Event | Who | Writes |
|---|---|---|
| Flagged at scoping | `/pm-sprint-delivery` | `needs-verification` + steps on the issue; the row on the next ticket, **Sprint** naming the umbrella |
| Flagged mid-implementation | `/engineer-implement` | label + steps on the issue; `## Verify on` + `Verification: #N` on the PR; the row on the next ticket, before asking for review |
| Filed by a standalone skill | that skill | label + steps on the issue; the row on the next ticket (**Sprint** `—`) — at filing time, not when the fix merges |
| Merged | `/merge-steward` | the label: `needs-verification` → `awaiting-verification`. Nothing on the ticket — a merged PR is not a verified issue; the row stays ⏳ |
| Deployed | `/devops-deploy` | deploy record (commit + URL, and the tag when deploying a release) on every open ticket with a row that commit contains; ⏳ → 🔍 for those rows; the window comment |
| Tagged | `/release-cut` | the tag; **Tag** and **Tag commit** on that release's ticket, retitled to the version; the next ticket, opened in the same step |
| Released | `/release-verify` | membership by ancestry — rows not in the tag roll forward to the next ticket; a row for every unrecorded `awaiting-verification` item in the tag; the one post-release pass triggered on the tag's commit |
| Verified | `/qa-sprint-verify` | results comment on the ticket; each row ✅/❌/⚠️ with the commit; `awaiting-verification` removed on ✅ and ❌; one defect per ❌ via `/qa-issue-report`, linked from the row; a ❌ on a released tag means a patch release, never a moved tag |
| Waived | a human; `/release-verify` records it | ⏭ with who and why on the row; `awaiting-verification` removed |
| All rows terminal | `/release-verify` | closing summary on the ticket and close it |

A ❌ row does not reopen the source issue by itself: the defect is the
new issue, cross-linked both ways; re-scoping is the PM's call.

## Reporting it

- **Its own state.** Boards, standups, status reports and digests show
  *Awaiting verification* (⏳ or 🔍, label `awaiting-verification`) as a
  distinct state between *Merged* and *Done* — never folded into
  *Blocked*, *Stalled* or *Not done*
- **Overdue thresholds** (defaults; tune per team and say so when
  applied). The clock starts at the **tag**, not at the merge or the end
  of a sprint: a tag that has existed for more than **2 days** with no
  deploy record for its commit, or a deploy record for the tag's commit
  more than **1 day** old with no pass run, is a **verification backlog**
  impediment — owned by DevOps (the missing deploy) or QA (the missing
  pass). One deploy of the tag or one pass clears the whole backlog; it
  is never routed as N per-issue deploys
- **Not overdue: rows on the next ticket.** A merged item waiting for a
  cut is in its normal state however old it is. How long the repo goes
  between releases is a release decision; report it as the next ticket's
  age and row count, not as a verification impediment
- **Unrecorded is reported at once.** An `awaiting-verification` issue
  with no row has no clock to wait for — name it, and `/release-verify`
  as the owner of the fix
- **Never per-issue.** An urgent item is answered by the next release and
  its pass, not by syncing its branch to a shared environment

## Limits

- **One release line per repo.** Tags are cut from one branch, and a
  patch release is the next tag on that same line. A repo that maintains
  several release branches at once is not covered: ancestry would put one
  merge in more than one pending ticket
- **Releases are per repo.** A change that spans repos has a row on each
  repo's own ticket; whether the releases must go out together is an open
  question for a human (`/coordinator-sync` treats it as one), not
  something a ticket decides

## Containarium trackers

`release`, `needs-verification` and `awaiting-verification` are not in a
connection's default label allow-list (`scope:*`, `model:*`,
`agent:needs-approval`). Add them with the other labels
(`/devops-containarium-env` capability 9); until then file with allowed
labels, label afterwards, and report the rejection — never drop the label
silently.
