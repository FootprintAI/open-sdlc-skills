---
name: "QA: Sprint Verify"
description: Run the batched verification pass — every issue that needs checking against a running environment, verified together against ONE deployed version: the released one, once /release-verify has triggered the pass on the tag's commit. Reads its rows off one home, the tag's release verification ticket (whichever sprint scoped them, or none), stamps every result with the commit it was verified on, removes `awaiting-verification` on ✅ and ❌, files failures as defects, and reports what is still unverified. A mid-cycle run before the tag writes pre-release notes, never a terminal status. When the environment was provisioned just for this pass, captures evidence that outlives it and hands the teardown back to DevOps only once every result is posted.
category: QA
tags: [qa, verification, release-verification, sprint, release, deploy, dev, prod, batch, evidence, ephemeral]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. Verification is a post-deploy follow-up, never a merge gate; every `needs-verification` item is recorded as a row on the next release's verification ticket — see `_shared/verification.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as QA running the **batched verification pass**. Some issues can't be
proven done by CI — a migration that has to run against real data, a UI flow
that only exists once deployed, a config or infra change, an integration with
a third party. Those get verified against a running environment. The rule
this skill exists to enforce: verify them **in one batch, against one
deployed version**, never one issue at a time. That version is the
**released** one: verification follows the release, not the sprint
(`_shared/verification.md`), so the pass that decides a row runs on the
tag's commit, triggered by `/release-verify released`.

**Why batched** — the dev environment serves the whole team. Verifying issue
by issue means redeploying dev per issue, so dev is never running one known
version and nobody can say what is actually on it. One deploy → one
verification pass → one commit stamped on every result. If someone needs an
issue verified sooner, the answer is a deploy, not a private build on dev.

The same rule holds — harder — when the environment is **provisioned for
the pass** and torn down afterwards. Then every verification costs a
provision, so per-issue verification costs the row count in provisions;
the batch costs one. It also means the environment is gone soon after the
pass: evidence has to be captured into something durable *during* the pass,
and the teardown must wait until every result is posted.

**Input**: `/qa-sprint-verify` (default: the **tag's ticket** — the
pending release verification ticket whose **Tag commit** is the commit the
`dev` environment runs; `/release-verify released` names it in the
hand-off). Options: `--ticket N` (name the ticket), `--env dev|prod`,
`--url <base-url>`, `--commit <sha>` (assert what should be running),
`--only #12,#15` (re-run specific rows after a redeploy).

**One home, two kinds of pass.** The rows live on the release
verification ticket and nowhere else — a sprint umbrella carries no
queue, and there is no separate verification issue. A pass works one
ticket:

- **Post-release pass** (the default, and the only one that decides a
  row) — the tag's ticket, on the tag's commit. It covers every 🔍 row on
  that ticket, whatever its **Sprint** column says: one pass per released
  version per repo, not one per sprint and not one per issue
- **Pre-release pass** (optional, mid-cycle) — the environment runs a
  commit that is no tag's commit, so there is no tag's ticket to default
  to: the pass works the **next ticket**'s 🔍 rows and writes each result
  as a note in **Verified on** (`pre-release: ✅ on 7777777`). It leaves
  **Status** and the issue's label alone; the post-release pass still
  checks every row on the released version

**Preconditions — refuse rather than fake**

1. A **release verification ticket** with rows (`epic` + `release`,
   `_shared/verification.md`), found by label and told apart by its
   **Tag** field, never by its title. No ticket, or no rows on it →
   nothing to batch; say so and stop. An `awaiting-verification` issue
   with no row is not this pass's to add: report it, with
   `/release-verify` as the owner of the fix
2. A deploy record for the target environment on that ticket: commit + URL,
   posted by `/devops-deploy`. No deploy record → the pass has no version to
   verify against. For a tag's ticket the record's commit must equal the
   ticket's **Tag commit** — a result on any other commit is a result
   about another version, not a pass for this release
3. At least one row marked 🔍 *ready to verify* — rows still ⏳
   *awaiting deploy* (their PR merged after the deployed commit) are **not**
   verifiable in this pass; list them as carried to the next deploy
4. The environment is still running that commit when the pass starts. A pass
   split across two versions proves nothing
5. If the deploy record says the environment was provisioned for this pass,
   nobody tears it down while this pass runs. Claim the window with one
   comment on the ticket (`Verification pass starting on <commit>`)
   before the first row, so DevOps can see a pass is in flight

**Steps**

1. **Read the ticket's rows and the deploy record**

   ```bash
   gh issue list --label release --state open --json number,title,body
   gh issue view <ticket> --json body,comments
   ```

   Pick the ticket: `--ticket`, or the one the hand-off named, or the
   pending ticket whose **Tag commit** equals `--commit` / the running
   commit; none of those → the next ticket (no **Tag**), as a pre-release
   pass. From its body: every row — **Source**, **Sprint**, **Verify**,
   **Expected**, **Env / version** and the commit it needs, **Status**.
   From the comments: the most recent `/devops-deploy` record for `--env`
   (deployed commit + URL). Build the working list: 🔍 ready rows only, in
   row order. If a results comment for this commit and environment is
   already on the ticket, the pass has run: say so and stop, unless
   `--only` names rows to re-run.

2. **Confirm what is actually running — this is the pass's identity**

   Ask the deployed service, not the deploy comment: a `/version` or
   `/healthz` endpoint reporting the build commit, the platform's reported
   revision, or the checked-out SHA on the target. Record it as `<commit>` —
   **every result in this pass is stamped with it**.

   If the running commit differs from the ticket's deploy record — or,
   on a tag's ticket, from its **Tag commit** — stop and report the drift:
   someone deployed over the version, and the rows' readiness was computed
   against a version that is no longer there. Re-deploy (`/devops-deploy`,
   by tag for a release) and re-run this pass.

3. **Verify every ready item in one session**

   Walk the working list against that one URL:

   - **UI items** — hand them to `/qa-e2e-test` as a single run against the
     deployed URL, covering every UI row's flow in that one pass (not
     one e2e run per issue). Its screenshots are this pass's evidence, and
     its test-data cleanup rules apply to anything the run creates
   - **API / CLI / data items** — exercise the documented path directly
     (request + response, command + output, query + rows) and keep the raw
     output as evidence
   - Follow the issue's own **Verify on `<env>`** steps as written. If they
     are missing or don't match reality, that is a finding about the issue,
     not a licence to improvise a passing path — record the item as *not
     verified: no usable verification steps* and route it back to the issue's
     author
   - Evidence per item is mandatory: a screenshot, a response body, or a
     command's output. No evidence → not verified, regardless of what you saw
   - Evidence lives somewhere that outlives the environment: the e2e report
     and its screenshots committed to the repo, response bodies and command
     output pasted into the results comment or attached to the issue. A
     link into the environment itself (`https://dev…/documents/42`) is a
     pointer, not evidence — once the environment is torn down or
     redeployed it points at nothing, and the result goes with it

4. **Record each item's result**

   | Result | Meaning |
   |--------|---------|
   | ✅ verified | The stated criterion was observed on `<commit>`, with evidence |
   | ❌ failed | The criterion was not met — a defect gets filed in step 6 |
   | ⚠️ not verified | Could not run it (missing sample data, dependency down, no usable steps) — with the reason |

   "Not verified" is a legitimate outcome and a useful one. It is never
   silently upgraded to verified, and never reported as a failure of the code.

   In a **pre-release pass** the same three results are recorded as notes,
   not statuses (step 5): they are evidence about that commit, not about
   the release.

5. **Post the results where the team coordinates**

   One results comment on the ticket, and one short comment per source
   issue — the result, the commit, and a link to the ticket's results
   comment, nothing more (the issue's timeline then explains why its label
   changed; the evidence stays on the ticket):

   ```markdown
   ## Verification pass — <env> @ `<commit>` — <date>

   **Environment:** <url> (running `<commit>`, confirmed at <time>)
   **Ticket:** #61 — Release verification: v1.3.0 (tag commit `<commit>`)
   **Batched items:** 5 ready of 7 rows

   | Issue | What was verified | Result | Evidence |
   |-------|-------------------|--------|----------|
   | #12 | PDF upload → parsed result listed | ✅ | [e2e report](…) 04-success.png |
   | #15 | SSO login redirects to dashboard | ✅ | [e2e report](…) 02-dashboard.png |
   | #18 | `migrate up` applied on real data | ✅ | `SELECT count(*)` output below |
   | #19 | Webhook retried on 5xx | ❌ | defect #31 — no retry observed, logs attached |
   | #21 | Export job finishes < 30s | ⚠️ not verified | needs a sample dataset (asked on #21) |

   **Still awaiting deploy (not in `<commit>`):** #23, #24 — next deploy
   **Release:** 🔴 not clear — #19 failed (defect #31 → patch release), #21 unverified
   ```

   Then update the ticket's rows to match — **Status** per the legend
   and **Verified on** with the commit, date and evidence link. A row
   marked ✅ without a commit is not a verified row.

   **Advance the label with the row** (`_shared/verification.md`, *The
   label lifecycle*): on ✅ and on ❌, remove `awaiting-verification` from
   the source issue with the tracker's label verb
   (`gh issue edit <N> --remove-label awaiting-verification`). A ⚠️ row
   keeps the label — nothing was decided. ⏭ is a human's waiver and
   `/release-verify` records it; never write one here.

   **Pre-release pass:** leave **Status** at 🔍 and the label where it
   is. Write the result into **Verified on** as a note stamped with the
   commit — `pre-release: ✅ on 7777777 (2026-10-28, <evidence>)`,
   `pre-release: ❌ on 7777777 (defect #31)`, `pre-release: ⚠️ <why>` —
   and head the results comment `Pre-release pass`. The post-release pass
   may reuse that evidence only when the tag ends up pointing at the very
   commit it was observed on; otherwise it re-runs the row.

   Do not close the ticket and do not move rows between tickets: both are
   `/release-verify`'s (`close`, and membership by ancestry at
   `released`).

6. **File every failure as a defect, immediately**

   Hand each ❌ to `/qa-issue-report`: one issue per finding, the deployed
   `<commit>` and environment recorded in the issue, evidence attached, and
   cross-linked to the original issue in both directions. The original
   issue's row goes ❌ with the defect number — reopening or re-scoping
   it is the PM's call, not this skill's. A ❌ on a released tag never
   moves the tag: the fix ships as the next patch release, and the defect
   gets its own row on the next ticket from whoever flags it. A
   pre-release ❌ files its defect straight away too.

7. **Report and hand off**

   > "Verification pass on `<env>` @ `<commit>`: 3/5 verified, 1 failed
   > (defect #31), 1 unverified (missing sample data). 2 items awaiting the
   > next deploy (#23, #24). Release v1.3.0: NOT clear. Posted on
   > release verification ticket #61; `awaiting-verification` removed
   > from 4 issues."

   The ticket's rows are what `/release-verify` re-reads (`refresh`,
   then `close`) and what the sprint side summarizes by **Sprint**. This
   pass never gated the merge, the deploy or the cut; an unclear result
   keeps the tag's ticket open — say so plainly rather than rounding up.

8. **Release the environment — only if it was provisioned for this pass**

   Skip this step for a persistent shared rung; it keeps running the
   deployed version until the next deploy. For a per-pass environment,
   this pass is the reason it exists, and finishing the pass is what
   releases it:

   - Everything above is done: the results comment is on the ticket
     naming `<commit>`, every ❌ has its defect filed with evidence
     attached, every evidence link points at something durable
   - Then hand off: `/devops-deploy teardown <env>`. DevOps re-checks those
     same conditions from the ticket before removing anything and posts
     the teardown record; this skill never deletes the environment itself
   - If anything is still outstanding — a defect not yet filed, a ⚠️ item
     whose missing sample data is arriving today, an item you want a
     second look at — say so instead of handing off. Keeping the
     environment up another hour is cheap; re-provisioning to redo one
     item is the cost the batch exists to avoid

   Add one line to the step 7 report: `Environment: released to DevOps for
   teardown` or `Environment: kept up — <what is outstanding>`.

**Prod passes**

Prod verification is optional and covers only rows flagged for prod
(usually the ones whose risk is data- or scale-shaped). Same batching rule:
one prod deploy of the tag, one pass, one commit — a second pass on the
same tag's ticket. Additionally:

- **Read-only by default** — exercise paths that don't mutate real data.
  Anything that must write requires a designated test account and a
  human's explicit ok, recorded on the ticket
- **Never clean up by deleting on prod** — if the pass created something,
  report it for a human to remove; a delete on prod is not QA's call
- Never use real customer records as test data, and never paste customer
  data into an issue or report

**Guardrails**

- **One version per pass.** Never redeploy mid-pass, and never report items
  verified on different commits as one pass — that is exactly the drift this
  skill exists to prevent
- Never verify an item whose PR is not contained in the deployed commit —
  it stays ⏳ and carries to the next deploy
- Never skip a row because of its **Sprint** column: a pass that covers
  one sprint's rows and leaves the ticket's other 🔍 rows untouched on the
  same commit is half a pass, and the next deploy will move the version
  out from under them
- Never read rows from anywhere but a release verification ticket, and
  never write a ✅ or ❌ **Status** from a commit that is not the ticket's
  tag commit — before the tag, a result is a `pre-release:` note
- Never leave `awaiting-verification` on an issue whose row this pass
  made ✅ or ❌, and never remove it for a ⚠️ or a pre-release note
- Every result records the environment and the commit. "Verified" without a
  version is not a result, and does not change a row
- Never mark an item verified from a green CI run, a merged PR, or a code
  read — this pass exists precisely because CI could not prove it
- Never edit application code, fix a defect, or re-run a red item hoping it
  turns green — a red item goes green only on a new deployed commit
- Never point this pass at production unless the run was explicitly asked
  for prod, and never at a customer-facing environment for UI items that
  create data without the ok above
- Never hand an environment off for teardown with a ❌ that has no defect
  filed, a result that isn't posted, or evidence that only exists on the
  environment — and never ask for a teardown while another pass has claimed
  the window. Torn down too early, the environment takes the pass's proof
  with it
- This skill verifies and reports — it does not change sprint scope, close
  issues, or decide whether a failed item ships anyway (a human, recorded
  by `/release-verify`); it does not write rows, move them or close the
  ticket (`/release-verify`), and it does not deploy or tear down
  (`/devops-deploy`)
