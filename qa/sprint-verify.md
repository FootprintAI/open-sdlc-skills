---
name: "QA: Sprint Verify"
description: Run the sprint's batched verification pass — every issue that needs checking against a running environment, verified together against ONE deployed version right after the sprint deploys. Reads the verification queue off the sprint umbrella issue, stamps every result with the commit it was verified on, files failures as defects, and reports what is still unverified. When the environment was provisioned just for this pass, captures evidence that outlives it and hands the teardown back to DevOps only once every result is posted.
category: QA
tags: [qa, verification, sprint, umbrella-issue, deploy, dev, prod, batch, evidence, ephemeral]
---

Act as QA running the **sprint's verification pass**. Some issues can't be
proven done by CI — a migration that has to run against real data, a UI flow
that only exists once deployed, a config or infra change, an integration with
a third party. Those get verified against a running environment. The rule
this skill exists to enforce: verify them **in one batch, against one
deployed version**, never one issue at a time.

**Why batched** — the dev environment serves the whole team. Verifying issue
by issue means redeploying dev per issue, so dev is never running one known
version and nobody can say what is actually on it. One deploy → one
verification pass → one commit stamped on every result. If someone needs an
issue verified sooner, the answer is a deploy, not a private build on dev.

The same rule holds — harder — when the environment is **provisioned for
the pass** and torn down afterwards. Then every verification costs a
provision, so per-issue verification costs the queue length in provisions;
the batch costs one. It also means the environment is gone soon after the
pass: evidence has to be captured into something durable *during* the pass,
and the teardown must wait until every result is posted.

**Input**: `/qa-sprint-verify` (default: the newest open sprint umbrella,
`dev` environment). Options: `--umbrella N`, `--env dev|prod`,
`--url <base-url>`, `--commit <sha>` (assert what should be running),
`--only #12,#15` (re-run specific queue items after a redeploy).

**Preconditions — refuse rather than fake**

1. An open sprint umbrella with a **Verification queue** section (created by
   `/pm-sprint-delivery`). No queue → nothing to batch; say so and stop.
   If the repo has **several** open umbrellas, the environment is shared
   by all of them: read every open umbrella's queue, and the pass covers
   every 🔍 row across them — one pass per deployed commit per repo, not
   one per sprint. Results go on each umbrella whose rows they touch
2. A deploy record for the target environment on that umbrella: commit + URL,
   posted by `/devops-deploy`. No deploy record → the pass has no version to
   verify against
3. At least one queue item marked 🔍 *ready to verify* — items still ⏳
   *awaiting deploy* (their PR merged after the deployed commit) are **not**
   verifiable in this pass; list them as carried to the next deploy
4. The environment is still running that commit when the pass starts. A pass
   split across two versions proves nothing
5. If the deploy record says the environment was provisioned for this pass,
   nobody tears it down while this pass runs. Claim the window with one
   comment on the umbrella (`Verification pass starting on <commit>`)
   before the first item, so DevOps can see a pass is in flight

**Steps**

1. **Read the queue and the deploy record**

   ```bash
   gh issue view <umbrella> --json body,comments
   ```

   From the body: every queue row — issue number, what to verify, target
   environment, current status. From the comments: the most recent
   `/devops-deploy` record for `--env` (deployed commit + URL). Build the
   working list: 🔍 ready items only, in queue order.

2. **Confirm what is actually running — this is the pass's identity**

   Ask the deployed service, not the deploy comment: a `/version` or
   `/healthz` endpoint reporting the build commit, the platform's reported
   revision, or the checked-out SHA on the target. Record it as `<commit>` —
   **every result in this pass is stamped with it**.

   If the running commit differs from the umbrella's deploy record, stop and
   report the drift: someone deployed over the sprint's version, and the
   queue's readiness was computed against a version that is no longer there.
   Re-deploy (`/devops-deploy`) and re-run this pass.

3. **Verify every ready item in one session**

   Walk the working list against that one URL:

   - **UI items** — hand them to `/qa-e2e-test` as a single run against the
     deployed URL, covering every UI queue item's flow in that one pass (not
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

5. **Post the results where the team coordinates**

   One comment on the umbrella, and one short comment per child issue:

   ```markdown
   ## Verification pass — <env> @ `<commit>` — <date>

   **Environment:** <url> (running `<commit>`, confirmed at <time>)
   **Batched items:** 5 ready of 7 queued

   | Issue | What was verified | Result | Evidence |
   |-------|-------------------|--------|----------|
   | #12 | PDF upload → parsed result listed | ✅ | [e2e report](…) 04-success.png |
   | #15 | SSO login redirects to dashboard | ✅ | [e2e report](…) 02-dashboard.png |
   | #18 | `migrate up` applied on real data | ✅ | `SELECT count(*)` output below |
   | #19 | Webhook retried on 5xx | ❌ | defect #31 — no retry observed, logs attached |
   | #21 | Export job finishes < 30s | ⚠️ not verified | needs a sample dataset (asked on #21) |

   **Still awaiting deploy (not in `<commit>`):** #23, #24 — next deploy
   **Sprint gate:** 🔴 not clear — #19 failed (defect #31), #21 unverified
   ```

   Then tick the umbrella's queue rows to match, each with the commit it was
   verified on. A ticked row without a commit is not a verified row.

6. **File every failure as a defect, immediately**

   Hand each ❌ to `/qa-issue-report`: one issue per finding, the deployed
   `<commit>` and environment recorded in the issue, evidence attached, and
   cross-linked to the original issue in both directions. The original
   issue's queue row goes ❌ with the defect number — reopening or re-scoping
   it is the PM's call, not this skill's.

7. **Report and hand off**

   > "Verification pass on `<env>` @ `<commit>`: 3/5 verified, 1 failed
   > (defect #31), 1 unverified (missing sample data). 2 items awaiting the
   > next deploy (#23, #24). Sprint verification gate: NOT clear.
   > Posted on umbrella #17."

   A clear gate is the evidence `/pm-sprint-delivery close` and
   `/team-sprint-cycle`'s close stage require. An unclear one blocks the
   close — say so plainly rather than rounding up.

8. **Release the environment — only if it was provisioned for this pass**

   Skip this step for a persistent shared rung; it keeps running the
   deployed version until the next deploy. For a per-pass environment,
   this pass is the reason it exists, and finishing the pass is what
   releases it:

   - Everything above is done: the results comment is on the umbrella
     naming `<commit>`, every ❌ has its defect filed with evidence
     attached, every evidence link points at something durable
   - Then hand off: `/devops-deploy teardown <env>`. DevOps re-checks those
     same conditions from the umbrella before removing anything and posts
     the teardown record; this skill never deletes the environment itself
   - If anything is still outstanding — a defect not yet filed, a ⚠️ item
     whose missing sample data is arriving today, an item you want a
     second look at — say so instead of handing off. Keeping the
     environment up another hour is cheap; re-provisioning to redo one
     item is the cost the batch exists to avoid

   Add one line to the step 7 report: `Environment: released to DevOps for
   teardown` or `Environment: kept up — <what is outstanding>`.

**Prod passes**

Prod verification is optional and covers only queue items flagged for prod
(usually the ones whose risk is data- or scale-shaped). Same batching rule:
one prod deploy, one pass, one commit. Additionally:

- **Read-only by default** — exercise paths that don't mutate real data.
  Anything that must write requires a designated test account and the PM's
  explicit ok, recorded on the umbrella
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
- Every result records the environment and the commit. "Verified" without a
  version is not a result, and does not tick a queue row
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
  issues, or decide whether a failed item ships anyway (PM), and it does not
  deploy or tear down (`/devops-deploy`)
