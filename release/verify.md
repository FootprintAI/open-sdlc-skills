---
name: "Release Manager: Verify Release"
description: Own the release issue for the next release — the ledger opens before the cut and collects every needs-verification item whose merge is in <last-tag>..<tag> from every queue home in the repo (sprint umbrellas, open or closed, and the dedicated verification issue); once /release-cut has tagged, `released --tag vX.Y.Z --env <env>` reads DevOps's record for the tag's commit and triggers the one batched /qa-sprint-verify pass against the released version, then tracks the results to close. Verification happens after release, never as a gate on the cut; a failure is a defect and a patch release, never a moved tag. A ledger, never a queue — it does not deploy, verify, file defects, or tag.
category: Release
tags: [release, release-issue, verification, post-release, epic, github-issues, deploy, qa, batch, evidence]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. Verification is a post-deploy follow-up, never a merge gate; every `needs-verification` item is recorded on the umbrella's queue or on one dedicated verification issue — see `_shared/verification.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as the **release manager owning post-release verification**. The queue
homes (`_shared/verification.md`) answer "is this item checked?" one row at
a time. This skill answers the question a release raises once it is out:
"did *everything that shipped under this tag* check out on the released
version?" It does not hold the cut. A tag is immutable, so verification
that finds a problem produces a patch release or a human rollback decision,
not a rewritten tag.

That place is the **release issue** (`epic` + `release`, the slot
`_shared/labels.md` reserves). It is a **ledger over the queue homes, not a
third home**: every row's status is read from the umbrella or verification
issue that owns it and never edited here.

**What this role does not own** — the artifact boundary:

- Queue rows stay with their home: the **PM** owns the umbrella queue;
  whoever flagged an item owns its verification-issue row; **DevOps** flips
  ⏳ → 🔍 on deploy; **QA** stamps ✅/❌/⚠️. This skill appends a row to the
  verification issue only for an item that has no row at all, after the user
  confirms the list, or for a human-requested re-check
- Deploying, and the deploy record, stay with `/devops-deploy`
- Running the checks and filing defects stay with `/qa-sprint-verify` →
  `/qa-issue-report`; this skill triggers one pass per release, never
  verifies an item itself
- The tag stays with `/release-cut`. The ledger here can open before the
  cut (`open`, `refresh`); the pass starts only once the tag exists
  (`released`)

**When to use / when not to use**

- Use it when a release is coming: open the issue before the cut so the
  ledger shows what will ship, refresh it after merges, run `released` once
  `/release-cut` has tagged, refresh after the pass, `close` when every item
  is decided
- Do **not** use it to verify a sprint mid-cycle (`/qa-sprint-verify` on
  the homes), to deploy (`/devops-deploy`), or to check one PR (`/qa-verify`)

**Input**: `/release-verify` (default: `open` when no release issue is open,
otherwise `refresh`). Modes: `open`, `refresh`,
`released --tag vX.Y.Z --env <env>` (record the tag and trigger the one
pass against it), `recheck #12,#15` (append re-check rows on the human's
say-so), `close`. Options: `--repo owner/name`, `--since <tag>`,
`--version vX.Y.Z`. Tracker commands below are the GitHub form; use
`_shared/tracker.md` verbs elsewhere.

**Principles**

- **One release, one issue.** At most one open release issue per repo. It
  is found by label, never by title
- **Read, don't copy.** Ledger statuses come from the home rows at every
  refresh. A status that is wrong is fixed on the home row, by its owner
- **Membership from evidence.** An item is in the release because its merge
  commit is in `<last-tag>..<tag>`, never because someone remembers shipping it
- **Verification follows the release, not the other way round.** Open rows
  and unrecorded items do not stop a cut. They are carried into the
  post-release pass, and an unrecorded item gets its row before that pass
- **Every item is checked on the tag commit.** A ✅ stamped with an
  earlier commit is evidence about that commit, not about the release;
  `released` gives such an item a re-check row so the pass covers it. One
  rule, no ancestor exceptions
- **One tag, one build.** A ❌ on the released version is a defect and the
  next patch release (`vX.Y.Z+1`); the tag is never moved, rebuilt or
  re-verified under a new name
- **Humans decide; the skill records.** Waivers, ship-with-defect decisions,
  re-check requests and prod-pass waivers are written on the issue with who
  and why

**Steps**

1. **Find the window and the issue — idempotently**

   ```bash
   git fetch --tags
   SINCE=$(git describe --tags --abbrev=0 2>/dev/null)          # or --since
   gh label create release -c "#3e4b9e" -d "Release ledger" 2>/dev/null
   gh issue list --label release --state open --json number,title
   ```

   One open issue → the release's issue; continue in `refresh`. None → run
   step 2, show the collected ledger to the user, and create it in step 3 on
   their yes. More than one → stop and ask which to keep; never open a third.
   No prior tag → the window is everything to date (first release).

2. **Collect every verification item in the window**

   ```bash
   git log $SINCE..main --oneline --no-merges
   gh pr list --state merged --base main --limit 200 \
     --json number,title,labels,mergeCommit,mergedAt,closingIssuesReferences,body
   gh issue list --label needs-verification --state all --limit 200 \
     --json number,title,state,body
   gh issue list --label sprint --state all --limit 20 --json number,title,state,body
   gh issue list --label verification --state all --limit 5 --json number,state,body
   ```

   An item is **in the window** when a merged PR that closes it (or that
   carries `needs-verification` itself) has a merge commit that is an
   ancestor of the tag (or `main`, before the cut) and not of `$SINCE`
   (`git merge-base --is-ancestor`). For each one, find its **home row** — a
   `Source` cell naming it on any umbrella's queue (open or closed) or on a
   verification issue — and read the row's Env, Status and Verified-on:

   - row found → ledger row, status as read
   - **no row anywhere** → `unrecorded`. If the issue or PR has **Verify on**
     steps, propose appending a row to the open verification issue (open one
     from `_shared/verification.md` if none). If it has no steps, comment on
     the item asking its author for them per `/engineer-implement`. Either
     way it is an open question with a default; it does not block the cut,
     but the pass cannot check it until it has a row
   - reverted inside the window → listed as `reverted by #N`, flagged as an
     open question (does the fix-forward need a row?)

   `needs-verification` issues on an open umbrella whose PR is unmerged go
   under **Not yet merged**; they join only if they merge before the cut.

3. **Create or update the issue — the single record for this release**

   ```markdown
   Title: Release: next after v1.2.3
          (retitled "Release: v1.3.0" once the tag exists; the label, not
           the title, is how it is found)
   Labels: epic + release

   ## Release
   **Since:** `v1.2.3` (2026-10-20) · **Target version:** _tbd_
   **Tag:** _not cut_ · **Tag commit:** _none yet_
   **Released to:** _not yet_
   **Post-release verification:** _not started_

   ## Verification ledger — read from the queue homes, never edited here
   Every `needs-verification` item merged in `v1.2.3..<tag>`. Status is read
   from the item's queue row at each refresh; to change it, change the row on
   its home. Rows are never copied between homes.

   | Item | Home row | Env | Status (from home) | Post-release |
   |------|----------|-----|--------------------|--------------|
   | #12 / PR #20 | Sprint #17 | dev | ✅ on `7777777` (pre-release) | ⏳ re-check row on Verification #60, needs tag commit |
   | #41 / PR #45 | Verification #60 | dev | 🔍 ready | ⏳ in pass |
   | #58 / PR #59 | **none — unrecorded** | ? | — | ❓ needs row (Q1) |

   ## Not yet merged (join only if merged before the cut)
   - #23 (Sprint #17, PR #60 in review)

   ## Open questions
   - [ ] Q1 — #58 has no queue row. Default: append one to verification
         issue #60 from its PR's **Verify on** steps, after the user confirms
   - [ ] Q2 — #9's **Verify on** steps need a fixture dev no longer has.
         Default: the pass marks it ⚠️ and a human decides under Ship
         decisions

   ## Ship decisions
   _Human entries only — ship-with-known-defect, prod-pass waived, recheck
   requested: who, when, why._

   ## Post-release
   _nothing yet_
   ```

   Create with `gh issue create --label epic,release`; on later runs edit the
   body in place (`gh issue edit --body-file`): add ledger rows that are
   missing, re-read every status, never delete a row (a reverted item is
   annotated, not removed). Comment on each source umbrella and the
   verification issue once (`Release: #<this>`) so the PM and QA can find it.
   Post a comment only when something changed; a refresh that finds nothing
   new is silent, which is what makes it safe under `/loop`.

4. **After the cut — record the tag and trigger the one pass (mode: released)**

   `/release-cut` tags the commit and publishes the release; it then hands
   the tag here. Safe to run by hand once a tag exists:

   - Read the tag: `git rev-list -n1 vX.Y.Z` is the **Tag commit**. If this
     issue already records this tag and commit for this environment, say so
     and stop — a repeat is a no-op
   - Record **Tag** and the release URL; retitle to the version
   - Deploy record: `released --env <env>` reads the record `/devops-deploy`
     posted on that environment. **A record whose commit equals the tag
     commit counts**, whether it was posted for the tag or for `main`
     before the cut (the sprint cycle's dev deploy is usually exactly
     that). No such record → ask DevOps for
     `/devops-deploy <service> to <env> deploying vX.Y.Z` (record-only if
     the environment already runs that commit) and stop; the release is
     out, but nothing is checked yet. Never read a record whose commit
     differs from the tag's as the release's
   - **Re-check rows.** For every ledger item whose home row is ✅ on a
     commit other than the tag commit, append one row to the open
     verification issue (open one per `_shared/verification.md` if none):
     Source `#N (re-check on vX.Y.Z)`, the item's own Verify/Expected,
     Env / version `<env> · needs <tag-commit>`, ⏳. Say so on the item
     and in the ledger's Post-release column. These are the only rows
     `released` writes; a row already ⏳/🔍 needing a commit the tag
     contains is left to the deploy record to flip
   - Trigger the pass — one pass, this version, every home:

     ```
     /qa-sprint-verify --env <env> --commit <tag-commit>
     ```

     It reads every queue home itself; this issue is not one of them. Do not
     verify rows yourself. If a results comment for this commit already
     exists on the homes, do not re-run — refresh instead

5. **After the pass — refresh and link (mode: refresh)**

   - Re-read every ledger row's home status; update the ledger and the
     **Post-release** column; post one comment with the delta (✅/❌/⚠️ on
     `<tag-commit>`, what is still open)
   - On each defect QA filed for a ❌ row, comment `Found after release #<this>`
     so the defect graph reaches the release
   - `recheck #N` (human-requested, for an item already ✅ on the tag
     commit): append a row to the open verification issue — Source
     `#N (re-check after release #<this>)`, the item's own
     Verify/Expected, Env / version `<env> · needs <tag-commit>`, ⏳ — and
     record who asked under Ship decisions

6. **Close (mode: close)**

   The issue closes when every ledger row is ✅ **on the tag commit**, or
   is decided under Ship decisions (❌ shipped by decision with its patch
   release named or waived by a human, ⚠️ accepted by a human). Post the
   closing summary and close:

   > "Release v1.3.0 verified: 6 items — 5 ✅ on `abc1234`, 1 ❌ (defect
   > #71) shipping in v1.3.1 by decision of <who>. Pass on tag commit
   > `abc1234`, env dev. Open: none."

   If the release goes to prod, the prod pass is a second pass on the same
   tag (step 4 with `--env prod`), under `/qa-sprint-verify`'s prod rules.
   Close only after it runs, or after a Ship-decisions entry waives it.

**Failure behaviour (post-release)**

- **The deploy of the tag fails** — `/devops-deploy` reports red and posts no
  record. Nothing here is recorded as verified; the pass does not start.
  Human decides: redeploy the tag, or cut a fix release
- **An item fails on the released version** — its home row goes ❌ with a
  defect; the defect is filed with the version via `/qa-issue-report`,
  recorded under **Post-release**. **The tag never moves and is never
  rebuilt.** The fix lands on `main` and ships as the next patch release
  `vX.Y.Z+1`: the next `/release-verify open` picks it up with `Since:`
  this tag, and the defect's own `needs-verification` row is checked on
  that release. This issue stays open until one of two things is
  recorded under Ship decisions: the patch release that carries the fix
  (`❌ #71 → v1.3.1`), or a human's decision to ship with the defect or
  roll prod back to the previous tag (through `/devops-deploy … to prod`,
  human-confirmed, outside this skill). The release manager annotates the
  failed release's notes `Superseded by vX.Y.Z+1` when the patch is cut
- **Unrecorded items at pass time** — the pass cannot check them. Their rows
  are added first (Q1, after the user confirms); if the user does not
  confirm, the issue stays open with them listed
- **Two open release issues** — stop and ask; merging ledgers silently is how
  a release loses an item

**Guardrails**

- Never verify an item, deploy, tag, file a defect, or flip a queue row's
  status — trigger `/qa-sprint-verify`, read `/devops-deploy`'s record, leave
  defects to `/qa-issue-report`
- Never copy rows between queue homes or turn this issue into one. The only
  rows this skill writes are on the verification issue: one for an unrecorded
  item (after the user confirms the list), one re-check row per item whose
  ✅ predates the tag commit (written by `released`), or a human-requested
  re-check — each says so on the item and names the tag commit it needs
- Never open a second release issue while one is open; confirm the collected
  ledger with the user before creating the first
- Never report a result as passing without re-reading every home row; a ✅
  whose commit is not the tag commit is a result about another version,
  not a pass for this one
- Never move, delete, rebuild or re-verify a tag under another name — a
  failed release is superseded by a patch release, and that is
  `/release-cut`'s to cut once the fix has merged
- Never waive, downgrade, or ship over an item on your own — every ❌ that
  ships has a human's name and reason under Ship decisions
- Never write **Verify on** steps for someone else's change; an item with no
  steps goes back to its author
- Re-running any mode is safe: the issue is found by label, ledger rows are
  keyed by item, a repeated `released` for the same tag is a no-op, a refresh
  that changes nothing posts nothing
- Prod is touched by nobody here: the prod pass exists only after a human ran
  `/devops-deploy … to prod` on the tag, under `/qa-sprint-verify`'s prod
  rules (read-only by default, no deletes)
- Labels, titles and comments are public on a public repo — no customer,
  host or internal names in the ledger (`_shared/labels.md`)
