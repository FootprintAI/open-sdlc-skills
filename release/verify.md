---
name: "Release Manager: Verify Release"
description: Own the release verification ticket — the one queue home for needs-verification items, keyed by release, one ticket per pending tag per repo, holding a row per item written before its PR is approved. Once /release-cut has tagged, `released --tag vX.Y.Z --env <env>` settles membership by merge-commit ancestry (rows not in the tag roll forward to the next ticket), gets every merged awaiting-verification issue that has no row its row, reads DevOps's record for the tag's commit and triggers the one batched /qa-sprint-verify pass against the released version, then tracks the results to close. Verification happens after release, never as a gate on the cut; a failure is a defect and a patch release, never a moved tag. It owns the ticket, not the results — it does not deploy, verify, file defects, or tag.
category: Release
tags: [release, release-verification, verification, post-release, epic, github-issues, deploy, qa, batch, evidence]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. Verification is a post-deploy follow-up, never a merge gate; every `needs-verification` item is recorded as a row on the next release's verification ticket — see `_shared/verification.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as the **release manager owning post-release verification**. Each row
on a release verification ticket (`_shared/verification.md`) answers "is
this item checked?" This skill answers the question a release raises once
it is out: "did *everything that shipped under this tag* check out on the
released version?" It does not hold the cut. A tag is immutable, so
verification that finds a problem produces a patch release or a human
rollback decision, not a rewritten tag.

That place is the **release verification ticket** (`epic` + `release`, the
slot `_shared/labels.md` reserves). It is **the one queue home**: there is
no queue on a sprint umbrella and no separate verification issue. One
ticket per pending tag per repo — the **next ticket** (no tag yet, where
new rows are written) and a **pending ticket** for each tag whose pass is
not finished.

**What this role does not own** — the artifact boundary:

- The ticket is this skill's: its header, which rows belong on it, its
  open questions and ship decisions. The **results** are not. Whoever
  flags an item writes its row; **DevOps** writes the deploy record and
  flips ⏳ → 🔍; **QA** stamps ✅/❌/⚠️. This skill writes a row only for an
  item that has none (after the user confirms the list) or for a
  human-requested re-check, moves rows between tickets by ancestry, fills
  in a row's merge commit, and records a human's waiver as ⏭
- Deploying, and the deploy record, stay with `/devops-deploy`
- Running the checks and filing defects stay with `/qa-sprint-verify` →
  `/qa-issue-report`; this skill triggers one pass per release, never
  verifies an item itself
- The tag stays with `/release-cut`, which also opens the next ticket in
  the same step that tags. The pass starts only once the tag exists
  (`released`)

**When to use / when not to use**

- Use it across a release: `refresh` after merges so the next ticket
  shows what will ship, `released` once `/release-cut` has tagged,
  `refresh` after the pass, `close` when every row is decided. `open` is
  for a repo that has no next ticket yet
- Do **not** use it to run a pass (`/qa-sprint-verify`, which this skill
  triggers), to deploy (`/devops-deploy`), or to check one PR
  (`/qa-verify`)

**Input**: `/release-verify` (default: `open` when no next ticket is open,
otherwise `refresh`). Modes: `open`, `refresh`,
`released --tag vX.Y.Z --env <env>` (settle the tag's rows and trigger the
one pass against it), `recheck #12,#15` (append re-check rows on the
human's say-so), `close --tag vX.Y.Z`. Options: `--repo owner/name`,
`--since <tag>`, `--version vX.Y.Z` (name the next ticket for a planned
version). Tracker commands below are the GitHub form; use
`_shared/tracker.md` verbs elsewhere.

**Principles**

- **One pending tag, one ticket.** At most one open next ticket per repo,
  and one ticket per tag. Tickets are found by label and told apart by
  their **Tag** field, never by title
- **One home.** A row lives on exactly one ticket. When membership
  changes, the row moves whole; it is never duplicated
- **Membership from evidence.** A row belongs to a tag's ticket because
  its merge commit is an ancestor of the tag and not of the previous tag,
  never because of which ticket was open when it was flagged or because
  someone remembers shipping it
- **Verification follows the release, not the other way round.** Open rows
  and unrecorded items do not stop a cut. They are carried into the
  post-release pass, and an unrecorded item gets its row before that pass
- **Every item is checked on the released version.** A result from a
  mid-cycle dev pass sits in the row's **Verified on** cell as a
  pre-release note; it is evidence about that commit, not about the
  release, and the row stays open for the post-release pass
- **One tag, one build.** A ❌ on the released version is a defect and the
  next patch release (`vX.Y.Z+1`); the tag is never moved, rebuilt or
  re-verified under a new name
- **One release line.** Tags are cut from one branch per repo. A repo
  that maintains several release branches at once is outside this
  contract; say so and stop rather than guess which ticket a merge
  belongs to
- **Humans decide; the skill records.** Waivers, ship-with-defect decisions,
  re-check requests and prod-pass waivers are written on the ticket with
  who and why

**Steps**

1. **Find the tickets and the window — idempotently**

   ```bash
   git fetch --tags
   SINCE=$(git describe --tags --abbrev=0 2>/dev/null)          # or --since
   gh label create release -c "#3e4b9e" -d "Release verification ticket" 2>/dev/null
   gh label create awaiting-verification -c "#c5def5" \
     -d "Merged, not yet verified on a release" 2>/dev/null
   gh issue list --label release --state open --json number,title,body
   ```

   Sort the open tickets by their **Tag** field: no tag → the **next
   ticket**; a tag → that tag's **pending ticket**. No next ticket → run
   step 2, show the collected rows to the user, and create it in step 3 on
   their yes. Two next tickets, or two tickets for one tag → stop and ask
   which to keep; never merge them silently. No prior tag → the window is
   everything to date (first release).

2. **Reconcile the rows with the evidence** (every open ticket)

   ```bash
   gh pr list --state merged --base main --limit 200 \
     --json number,title,labels,mergeCommit,mergedAt,closingIssuesReferences,body
   gh issue list --label awaiting-verification --state all --limit 200 \
     --json number,title,state,body
   gh issue list --label needs-verification --state all --limit 200 \
     --json number,title,state,body
   ```

   - **Each row** — resolve its PR and merge commit. Once the PR has
     merged, write the commit into **Env / version** (`needs <sha>`); an
     unmerged row keeps `needs merge`
   - **The backstop** — every issue labelled `awaiting-verification` must
     match a **Source** cell on an open ticket. **No row anywhere** →
     `unrecorded`. If the issue or PR has **Verify on** steps, propose
     appending a row to the ticket its merge commit belongs to (the
     pending ticket of the tag that contains it, otherwise the next
     ticket). If it has no steps, comment on the item asking its author
     for them per `/engineer-implement`. Either way it is an open question
     with a default; it does not block the cut, but the pass cannot check
     it until it has a row
   - **Label repair** — an issue still labelled `needs-verification`
     whose closing PR has merged (a merge outside `/merge-steward`) is
     advanced to `awaiting-verification`. A row that is ✅, ❌ or ⏭ whose
     issue still carries `awaiting-verification` has the label removed.
     These are the only label writes this skill makes
   - **Reverted** — an item whose merge was reverted before the tag stays
     on the ticket, annotated `reverted by #N`, and is flagged as an open
     question (does the fix-forward need a row?)

3. **Create or update the ticket — the single record for this release**

   ```markdown
   Title: Release verification: next release (opened 2026-10-21)
          (or "Release verification: v1.3.0 (opened …)" with --version;
           retitled "Release verification: v1.3.0" once the tag exists.
           The label and the Tag field, not the title, are how it is found)
   Labels: epic + release

   ## Release
   **Since:** `v1.2.3` (2026-10-20) · **Tag:** _not cut_ · **Tag commit:** _none yet_
   **Deployed to dev:** _not yet_
   **Deployed to prod:** _n/a_
   **Post-release verification:** _not started_

   ## Verification rows
   Every item that needs a check against a deployed environment and ships
   with this release. Nothing here blocks a merge or the cut; everything
   here is verified in one batched pass on the released version by
   `/qa-sprint-verify`.

   | Source | Sprint | Verify | Expected | Env / version | Status | Verified on |
   |--------|--------|--------|----------|---------------|--------|-------------|
   | #12 / PR #20 | #17 | create an invoice on `/billing` | PDF downloads, total matches | dev · needs `5555555` | 🔍 ready to verify | pre-release: ✅ on `7777777` (2026-10-28) |
   | #41 / PR #45 | #17 | upload `sample.pdf` on `/documents` | row appears within 10s, page count 3 | dev · needs `abc1234` | ⏳ awaiting deploy | |
   | #23 / PR #60 | #17 | toggle `beta-search` on `/settings` | search box appears | dev · needs merge | ⏳ awaiting deploy | |
   | #52 / PR #53 | — | `uname -r` on every `gpu-pool` host | ≥ 6.8.0-45 on all hosts | prod · needs rollout | ⏳ awaiting deploy | |

   Status legend: ⏳ awaiting deploy (merged, not on the env yet) ·
   🔍 ready to verify (in the deployed commit) · ✅ verified on `<commit>` ·
   ❌ failed (defect #N) · ⚠️ not verified (<why>) · ⏭ waived (<who + why>)

   ## Open questions
   - [ ] Q1 — #58 is `awaiting-verification` and has no row. Default:
         append one from its PR's **Verify on** steps, after the user
         confirms
   - [ ] Q2 — #9's **Verify on** steps need a fixture dev no longer has.
         Default: the pass marks it ⚠️ and a human decides under Ship
         decisions

   ## Ship decisions
   _Human entries only — ship-with-known-defect, waived, prod-pass waived,
   recheck requested: who, when, why._

   ## Post-release
   _nothing yet_
   ```

   Columns and legend are the ones `_shared/verification.md` defines; the
   **Sprint** column is what lets `/scrum-master` and `/coordinator-sync`
   read one sprint's items off the ticket.

   Create with `gh issue create --label epic,release`; on later runs edit the
   body in place (`gh issue edit --body-file`): add rows that are missing,
   fill in merge commits, never delete a row (a reverted item is annotated,
   not removed; a row that leaves does so only by moving to another ticket).
   Comment `Release verification: #<this>` once on each sprint umbrella
   named in the **Sprint** column so the PM and QA can find it. Post a
   comment only when something changed; a refresh that finds nothing new is
   silent, which is what makes it safe under `/loop`.

4. **After the cut — settle the rows and trigger the one pass (mode: released)**

   `/release-cut` tags the commit, records the tag on that release's
   ticket, opens the next ticket, and hands the tag here. Safe to run by
   hand once a tag exists:

   - Read the tag: `git rev-list -n1 vX.Y.Z` is the **Tag commit**. If the
     tag's ticket already records a pass on this commit for this
     environment, say so and stop — a repeat is a no-op
   - Find the two tickets: the tag's and the next. If the cut was made
     without them (a tag pushed by hand), do what `/release-cut` would
     have: record **Tag**, **Tag commit** and the release URL on the next
     ticket, retitle it `Release verification: vX.Y.Z`, and open a fresh
     next ticket with **Since:** this tag — or, when the next ticket is
     named for a later version than the one cut (a patch release), open
     `Release verification: vX.Y.Z` beside it
   - **Settle membership by ancestry.** A row is in the tag when its merge
     commit is an ancestor of the tag and not of the previous tag
     (`git merge-base --is-ancestor <sha> vX.Y.Z`). Rows on the tag's
     ticket that are not in the tag — unmerged, or merged after the tag's
     commit — move to the next ticket. Rows on the next ticket that are in
     the tag move to the tag's ticket. A row moves whole, pre-release
     notes included; post one comment on each ticket listing what moved
     and where
   - Run step 2's backstop for the tag's window, so every
     `awaiting-verification` item in the tag has a row before the pass
   - Deploy record: `released --env <env>` reads the record `/devops-deploy`
     posted for that environment. **A record whose commit equals the tag
     commit counts**, whether it was posted for the tag or for `main`
     before the cut (the sprint cycle's dev deploy is usually exactly
     that). No such record → ask DevOps for
     `/devops-deploy <service> to <env> deploying vX.Y.Z` (record-only if
     the environment already runs that commit) and stop; the release is
     out, but nothing is checked yet. Never read a record whose commit
     differs from the tag's as the release's
   - Trigger the pass — one pass, this version, this ticket's rows:

     ```
     /qa-sprint-verify --env <env> --commit <tag-commit>
     ```

     Name the tag's ticket in the hand-off; the pass works its rows and
     only its rows. Do not verify rows yourself. If a results comment for
     this commit already exists on the ticket, do not re-run — refresh
     instead

5. **After the pass — refresh and link (mode: refresh)**

   - Re-read every row; update **Post-release verification** in the
     header; post one comment with the delta (✅/❌/⚠️ on `<tag-commit>`,
     what is still open)
   - On each defect QA filed for a ❌ row, comment `Found after release #<this>`
     so the defect graph reaches the release; the failed row links the
     defect
   - A waiver a human entered under Ship decisions: write ⏭ with who and
     why on the row and remove the issue's `awaiting-verification`
   - `recheck #N` (human-requested, for an item already ✅ on the tag
     commit): append a row to the same ticket — Source
     `#N (re-check after release)`, the item's own Sprint, Verify and
     Expected, Env / version `<env> · needs <tag-commit>`, ⏳ — and
     record who asked under Ship decisions

6. **Close (mode: close)**

   A tag's ticket closes when every row is ✅ **on the tag commit**, or
   is decided under Ship decisions (❌ shipped by decision with its patch
   release named, ⏭ waived by a human, ⚠️ accepted by a human). Post the
   closing summary and close:

   > "Release v1.3.0 verified: 6 items — 5 ✅ on `abc1234`, 1 ❌ (defect
   > #71) shipping in v1.3.1 by decision of <who>. Pass on tag commit
   > `abc1234`, env dev. Open: none."

   The next ticket is never closed by this mode; it closes only after it
   has become a tag's ticket. If the release goes to prod, the prod pass
   is a second pass on the same tag (step 4 with `--env prod`), under
   `/qa-sprint-verify`'s prod rules. Close only after it runs, or after a
   Ship-decisions entry waives it.

**Failure behaviour (post-release)**

- **The deploy of the tag fails** — `/devops-deploy` reports red and posts no
  record. Nothing here is recorded as verified; the pass does not start.
  Human decides: redeploy the tag, or cut a fix release
- **An item fails on the released version** — its row goes ❌ with a
  defect; the defect is filed with the version via `/qa-issue-report`,
  recorded under **Post-release**. **The tag never moves and is never
  rebuilt.** The fix lands on `main` and ships as the next patch release
  `vX.Y.Z+1`. The defect's own row is written on the next ticket like any
  other item and is checked on the release that carries its merge — its
  own ticket `Release verification: vX.Y.Z+1` when the next ticket is
  already named for a later version. The failed tag's ticket stays open
  until one of two things is recorded under Ship decisions: the patch
  release that carries the fix (`❌ #71 → v1.3.1`), or a human's decision
  to ship with the defect or roll prod back to the previous tag (through
  `/devops-deploy … to prod`, human-confirmed, outside this skill). The
  release manager annotates the failed release's notes
  `Superseded by vX.Y.Z+1` when the patch is cut
- **Unrecorded items at pass time** — the pass cannot check them. Their rows
  are added first (Q1, after the user confirms); if the user does not
  confirm, the ticket stays open with them listed
- **Two next tickets, or two tickets for one tag** — stop and ask; merging
  them silently is how a release loses an item
- **More than one release branch** — out of contract (*One release line*).
  Report it and ask which line this ticket follows

**Guardrails**

- Never verify an item, deploy, tag, file a defect, or write a ✅/❌/⚠️ or
  a ⏳ → 🔍 — trigger `/qa-sprint-verify`, read `/devops-deploy`'s record,
  leave defects to `/qa-issue-report`
- Never duplicate a row. The only rows this skill writes are one for an
  unrecorded item (after the user confirms the list) and a human-requested
  re-check; the only rows it moves are the ones ancestry puts on another
  ticket, each move named in a comment
- Never decide membership by which ticket was open, by a sprint, or by a
  title — only by whether the merge commit is in the tag
- Never open a second next ticket while one is open; confirm the collected
  rows with the user before creating the first
- Never report a result as passing without re-reading every row; a
  pre-release note, or a ✅ whose commit is not the tag commit, is a
  result about another version, not a pass for this one
- Never move, delete, rebuild or re-verify a tag under another name — a
  failed release is superseded by a patch release, and that is
  `/release-cut`'s to cut once the fix has merged
- Never waive, downgrade, or ship over an item on your own — every ⏭ and
  every ❌ that ships has a human's name and reason under Ship decisions
- Never write **Verify on** steps for someone else's change; an item with no
  steps goes back to its author
- Re-running any mode is safe: tickets are found by label and Tag field,
  rows are keyed by Source, a repeated `released` for the same tag is a
  no-op, a refresh that changes nothing posts nothing
- Prod is touched by nobody here: the prod pass exists only after a human ran
  `/devops-deploy … to prod` on the tag, under `/qa-sprint-verify`'s prod
  rules (read-only by default, no deletes)
- Labels, titles and comments are public on a public repo — no customer,
  host or internal names on the ticket (`_shared/labels.md`)
