---
name: "Team: Sprint Cycle"
description: Drive a complete sprint cycle end to end by chaining five roles and enforcing the hand-off between them. A project manager scopes the sprint first (umbrella issue + prioritized, flow-first child issues, per the /pm-sprint-delivery contract), engineers automatically implement every scoped issue (each routed to a Claude model tier — model:sonnet / model:opus / model:fable label + rationale — implemented one worktree per issue under that model via the /engineer-implement contract), DevOps releases the cycle's merged work to the dev environment via the /devops-deploy contract, QA verifies the cycle's needs-verification issues in one batched pass against that single deployed version via the /qa-sprint-verify contract, and once the PM closes the cycle a release manager cuts the release — tag on the codebase and GitHub, release notes, and the release image build workflow triggered — via the /release-cut contract. Pair with /loop for a self-running cycle.
category: Team
tags: [team, sprint, pm, engineer, devops, qa, verification, release, umbrella-issue, model-routing, worktree, deploy, tag, loop]
---

Act as the **cycle orchestrator**: you do not own scope (the PM stage does)
or code (the engineer stage does) — you own that the cycle *moves* and that
each hand-off actually happens.

This skill *runs* a cycle. To diagnose one that has stopped moving — stalled
claims, starved reviews, WIP sprawl — use `/scrum-master`, which watches the
board and routes impediments without touching the work.

Run a **complete sprint cycle** as five chained roles with a hard hand-off
between each:

1. **The project manager scopes** — nothing is implementable until the PM
   stage has committed it to the sprint umbrella. Scoping authority lives
   with the PM voice of this skill, following the `/pm-sprint-delivery`
   contract: flow-first prioritization, one umbrella, one child issue per
   task.
2. **Engineers implement automatically** — every issue the PM scoped gets
   routed to the model tier it needs (`model:*` label + rationale),
   implemented in its own git worktree under that model via the
   `/engineer-implement` contract, and tracked back on the umbrella.
3. **DevOps releases to dev** — once the scope's PRs merge, the cycle's
   result is deployed to the dev environment via the `/devops-deploy`
   contract and verified there.
4. **QA verifies the cycle in one batch** — the issues CI couldn't prove
   done are verified together against that one deployed version via the
   `/qa-sprint-verify` contract, never one issue at a time.
5. **Release manager cuts the release** — once the PM closes the cycle on
   a verified dev release, the release manager voice of this skill cuts a
   real release via the `/release-cut` contract: semver tag pushed to the
   codebase and GitHub, release notes drafted from the cycle's merged
   PRs, and the release image build workflow triggered.

The umbrella issue is the contract across all five stages: the PM writes
scope and the verification queue onto it, engineers pull work off it and
report PRs back, DevOps posts the dev deploy's commit + URL (which flips
queue items to *ready to verify*), QA posts the batched pass's results
stamped with that commit, and the release manager posts the cut tag +
build result as the cycle's final entry.

**One version on dev per cycle.** The dev environment is shared, so it
carries the cycle's deployed commit and nothing else: no per-issue
deploys, no branch pushed to dev "just to check something". That is what
makes a single verification pass meaningful — every result names the same
commit.

**Input**: Run bare (`/team-sprint-cycle`) to advance the cycle one step —
each invocation does the next thing the cycle needs: scope it if there is
no open sprint, implement the next scoped items if there is, deploy to dev
once the scope's PRs are merged, run the batched verification pass on that
deployed version, close it once the queue is clear, then cut the release.
Modes to jump to one stage: `scope` (PM stage only), `implement` (engineer
stage only), `deploy` (DevOps stage only), `verify` (batched verification
pass only), `close` (PM close-out only), `release` (release manager stage
only), `status` (report the umbrella's state). Optionally `--repo owner/name`, `--max N`
implementations per invocation (default 2), `--env` for the dev/demo
target (defaults to whatever `/devops-deploy` calls its non-prod rung —
`dev` or `demo`), and `--auto-release` to let the release stage cut and
publish without pausing for confirmation (off by default — see step 9).

**Continuous mode**: pair with the loop harness —
`/loop 15m /team-sprint-cycle` — and the cycle runs itself: the PM stage
scopes newly-ready issues in as they arrive (e.g., defects filed by
`/qa-issue-report`), the engineer stage works the scope in priority order,
DevOps releases merged work to dev as it lands, QA verifies each deploy's
batch of `needs-verification` issues against that one commit, the umbrella
stays current, a finished-deployed-and-verified scope triggers close-out,
and closing hands straight to the release manager stage — which cuts the release once
you've turned on `--auto-release`, or otherwise leaves a confirmation
waiting for you before the next `/loop` tick.

**Model routing rubric** (engineer stage)

Classify each scoped issue by the hardest thing it demands, not its line
count:

- **`model:sonnet`** — mechanical and fully specified. The issue says
  exactly what to change and where: typo/copy fixes, config or dependency
  bumps, adding a test for known behavior, single-file fixes with a clear
  repro and an obvious cause. No design judgment required.
- **`model:opus`** — standard feature work. Multi-file changes inside the
  existing design: implementing a scoped story from the design doc, a bug
  whose cause needs real investigation, refactors with an existing test
  net. Judgment required, but the architecture already answers the big
  questions.
- **`model:fable`** — the hard tier. Cross-cutting or ambiguous work:
  `.proto` contract changes rippling across backend and frontend,
  concurrency/performance bugs, defects with an unclear repro, anything
  where the design doc is silent and the issue needs partial design
  thinking to solve. Also anything that already failed once at a lower
  tier.

Two tie-breakers: when genuinely unsure between tiers, pick the **higher**
one (a too-big model wastes tokens; a too-small one wastes the whole
attempt). And an issue labeled `blocker` severity gets at least
`model:opus` — blockers are a bad place to discover a model was
underpowered.

**Steps**

1. **PM stage — scope the sprint (skipped if an open sprint umbrella
   already has uncommitted capacity)**

   Act as the project manager per the `/pm-sprint-delivery` contract:

   - **Adopt before creating**: if an open `Sprint <YYYY-Www>` umbrella
     exists (from `/pm-sprint-delivery` or a previous cycle), that is the
     cycle's umbrella — never open a competing one
   - Otherwise collect candidates (`gh issue list --state open`, plus
     gaps you spot), prioritize **flow first, optimize second** (Phase 1 =
     the flow cannot work without it; Phase 2 = the flow works without
     it), and create the umbrella + child issues exactly as
     `/pm-sprint-delivery` specifies — child issues carry acceptance
     criteria, phase label, and `Blocked by` links; the umbrella lists
     Phase 1 / Phase 2 / out-of-sprint with back-links (`Part of #N`)
   - A healthy cycle scope is roughly 3–7 Phase 1 issues. Creating or
     scoping more than ~5 issues in one invocation is a bulk change —
     present the proposed scope to the user for confirmation first
   - Mark the scope's **verification items** as `/pm-sprint-delivery`
     specifies: any issue whose acceptance criteria CI can't prove gets
     the `needs-verification` label, **Verify on `<env>`** steps in its
     body, and a row on the umbrella's verification queue (⏳ awaiting
     deploy). This happens at scoping time, not when the issue merges —
     step 7 can only batch what was flagged
   - An issue is scope-eligible only when it is **ready**: not blocked,
     actionable (acceptance criteria or a repro path — comment asking for
     criteria instead of guessing), not already in progress, and an
     engineering task rather than an open product/design question. "Not
     already in progress" is now concrete, not a guess: an issue with an
     open linked PR, or a live `${who}-${model} is starting processing
     it` claim comment (per the `/engineer-implement` contract) that
     doesn't look stalled, is in progress — skip it

2. **Route every scoped Phase 1 issue and label it**

   Still at the hand-off boundary — before any implementation, each
   scoped issue gets its model routing recorded **on the issue**:

   - Ensure the label family exists (one-time setup):

     ```bash
     gh label create model:sonnet -c "#0e8a16" -d "Route to Sonnet — mechanical, fully specified" 2>/dev/null
     gh label create model:opus   -c "#1d76db" -d "Route to Opus — standard multi-file feature work" 2>/dev/null
     gh label create model:fable  -c "#5319e7" -d "Route to Fable — cross-cutting, ambiguous, or previously failed" 2>/dev/null
     ```

   - Exactly one `model:*` label per issue; an existing `model:*` label
     is only ever replaced to **escalate**, never to downgrade
   - Comment the rationale in one or two sentences so the routing is
     reviewable: "Routed `model:opus`: multi-file change across
     handler + proto client, design doc covers it, no contract change."
   - Mirror the tier next to each entry in the umbrella's Phase 1 list

   In `scope` mode, stop here and report the scoped + routed sprint.

3. **Engineer stage — automatically implement the scoped issues**

   Work the umbrella's unchecked Phase 1 scope in dependency order
   (unblockers first), up to `--max` per invocation. One issue = one
   worktree = one branch = one PR; never in the main checkout, never two
   issues in one working copy:

   - Spawn an implementation agent with the issue's routed model and
     worktree isolation (the Agent tool's `model` override +
     `isolation: "worktree"`). If driving the CLI instead:

     ```bash
     git worktree add ../<repo>-issue-<N> -b feat/<N>-<slug> main
     cd ../<repo>-issue-<N> && claude --model <sonnet|opus|fable> \
       -p "/engineer-implement #<N>"
     ```

   - Independent scoped issues may run in parallel (that is what the
     worktrees are for), still capped at `--max`
   - Each implementation follows the `/engineer-implement` contract in
     full: **claim the ticket first** (`${who}-${model} is starting
     processing it`, e.g. `agent-7f3a1c-opus`) — `${who}` is this
     specific agent/worktree instance's own id (the Agent tool's agent
     id when spawned that way, otherwise a short id minted at claim
     time), never the shared account every parallel worktree
     authenticates as, and `${model}` is this issue's routed tier — then
     test-first, design doc respected, team stack, small scoped PR with
     `Closes #<N>` and test evidence — this skill adds the cycle,
     routing, and isolation; it relaxes no engineering rule. The claim
     comment is what lets several issues implemented in parallel (all
     under the same `gh`/git identity) be told apart on the issue itself,
     without cross-checking which worktree is which
   - Remove each worktree once its PR is open (`git worktree remove`) —
     the branch lives on the remote; the worktree does not outlive the
     invocation

4. **Escalate when an attempt stalls**

   If an implementation under the routed model fails — tests can't reach
   green after a genuine effort, the agent reports the task exceeds the
   issue's spec, or the diff balloons past reviewable size:

   - Stop the attempt; keep the branch as evidence
   - Escalate one tier (`sonnet → opus → fable`), swap the `model:*`
     label, and comment what stalled: "Escalating to `model:fable`: repro
     is nondeterministic, needs concurrency analysis"
   - Re-run step 3 for that issue under the new model, giving it the
     stalled branch as context
   - A stall at `model:fable` is not a model problem — comment findings
     on the issue, mark it 🔴 on the umbrella, flag it for a human or for
     `/architect-design`, and move on. Never silently retry forever

5. **PM voice again — update the umbrella every invocation that changed
   something**

   - Tick Phase 1 checkboxes for issues whose PR merged; scope in
     newly-ready candidates (respecting the bulk guardrail)
   - Post one progress comment per changed invocation:

     ```markdown
     ## Cycle update — <date>

     **Status:** 🟢/🟡/🔴
     **Merged:** #42 (PR #45)
     **In review:** #43 → PR #46 (waiting on /reviewer-pr)
     **In flight:** #41 (`model:fable`, escalated from opus — <why>)
     **Scoped in:** #47 (new defect from /qa-issue-report, `model:sonnet`)
     **Dev release:** <not yet | `<commit>` @ `<dev-url>`, verified <date>>
     **Verification:** <queue empty | 3 ✅ on `<commit>`, 1 ❌ (defect #31),
       2 ⏳ awaiting the next deploy>
     **Cut release:** <not yet | `vX.Y.Z` tagged, build `<passed|failed>`>
     **Stalled:** none
     ```

6. **DevOps stage — release the cycle's merged work to dev (mode:
   deploy, or automatically once the scope's PRs are merged)**

   Closing on unverified code is paperwork, not delivery — before the
   cycle closes, ship what it built:

   - Follow the `/devops-deploy` contract in full: **committed state
     only** (a dirty tree or an untested commit does not deploy), sync
     the merged commit to the target container so remote == committed
     state exactly, run the service, and verify — health check / core
     route responding, metrics sane
   - Target the **dev** rung (`/devops-deploy <service> to dev`, or
     `demo` if that's what this repo calls its non-prod environment via
     `--env`) — this skill never deploys to prod; prod stays a separate,
     human-confirmed `/devops-deploy` call outside the cycle
   - If a merged issue is user-facing, a quick `/qa-e2e-test` happy-path
     pass against the deployed dev URL is the deploy's proof, exactly as
     the `/devops-deploy` contract calls for
   - Record the deployed commit + dev URL as a comment on the umbrella,
     and mark which verification-queue items that commit contains
     (⏳ → 🔍) per the `/devops-deploy` contract — this is the evidence
     step 7 verifies and step 8's closing summary points to
   - **No deployable service** (library, CLI-only project): skip this
     step, say so explicitly in the umbrella comment and the closing
     summary, and move straight to close
   - A failed deploy or failed verification **blocks closing the
     cycle** — report it red, fix forward or roll back per
     `/devops-deploy`'s own guardrails, and only proceed once dev is
     green (or the skip case above applies)

   In `deploy` mode, stop here and report the dev release.

7. **Verification stage — one batched pass on the deployed version (mode:
   verify, or automatically once the dev deploy is green)**

   The cycle's `needs-verification` issues have been waiting on exactly
   this: a known version on a shared environment. Verify them together,
   never as they merge:

   - Run `/qa-sprint-verify --env dev` against the commit step 6
     deployed. One pass, one commit, every 🔍 item in it
   - Results land on the umbrella and on each child issue, stamped with
     the commit; ❌ items become defects via `/qa-issue-report`,
     cross-linked to the issue they came from
   - ❌ and ⚠️ items are **cycle scope again**, not a closing footnote:
     scope the filed defect into this cycle if it fits (route it a
     `model:*` tier per step 2 and work it through step 3), otherwise
     carry it explicitly with the reason. A blocker-severity failed
     verification does not carry — it gets fixed, redeployed, and
     re-verified before close
   - ⏳ items merged after the deployed commit are not verifiable here.
     Either deploy again (back to step 6, which re-opens the window on
     the new commit) or carry them to the next cycle — never verify them
     against a version that doesn't contain them
   - **Empty queue** (nothing needed a deployed environment this cycle):
     say so on the umbrella and move on — an empty queue is a legitimate
     result, an unchecked one is not

   In `verify` mode, stop here and report the pass.

8. **Close the cycle (mode: close, or when Phase 1 is done, the dev
   release is verified, and the verification queue is clear — or the
   deploy was explicitly skipped per step 6)**

   Close out as the PM:

   - Verify every Phase 1 issue is closed via a merged PR, or has an
     explicit carry-over note (what remains and why)
   - Verify the verification queue is clear: every row ✅ with its commit,
     ❌ with a filed defect and a decision, or ⏭ waived with the user's
     say-so. Rows still ⏳ or 🔍 block the close — that is merged work
     nobody checked
   - Post a closing summary: shipped vs. scoped, escalations that
     happened (and what that says about the routing rubric), carry-overs,
     the dev deploy evidence from step 6 (commit + URL, or the
     no-deployable-service note), and the step 7 pass result (commit,
     ✅/❌/⚠️ counts, defects filed)
   - Close the umbrella if this skill created it; if it belongs to a
     standalone `/pm-sprint-delivery` run, post the summary as a comment
     and leave closing to that flow
   - Once closed, hand straight to step 9 — a closed cycle without a cut
     release is an incomplete cycle, not a finished one

9. **Release manager stage — cut the release (mode: release, or
   automatically once the cycle closes on a verified dev release)**

   Switch to the release manager voice and follow the `/release-cut`
   contract in full:

   - **Precondition**: the cycle's umbrella is closed, its DevOps stage
     (step 6) posted a verified dev release for the commit being
     released, and its verification queue (step 7) is clear — refuse to
     cut against an unclosed, undeployed, or unverified cycle
   - Scope the release to what this cycle actually merged: find the last
     tag, infer the semver bump (major/minor/patch) from the scoped
     issues' labels and PR content, and draft categorized release notes
     (Added / Fixed / Documentation / Internal) from the cycle's merged
     PRs
   - **Confirm before publishing, unless `--auto-release` was passed** —
     cutting a tag and triggering a build is a publish action, visible
     externally and awkward to undo, so it pauses for a yes by default.
     With `--auto-release` (an explicit, standing user choice — never
     assumed), skip the pause but still record the version and rationale
     for later audit
   - Tag the codebase (`git tag` + `git push`) and GitHub
     (`gh release create`) at the **same commit** — verify they match
   - Trigger the release image build workflow (or confirm a tag-push
     trigger already started it), watch it to completion, and report
     pass/fail — a red build makes this a **failed** release, not a
     tagged one
   - Comment the cut (version, tag URL, release URL, build run result)
     on the umbrella issue as the cycle's final entry

   In `release` mode, this is the only stage that runs; it still enforces
   the precondition above. The next invocation starts the next cycle's PM
   stage from whatever is ready — carry-overs and last cycle's Phase 2
   first (waterfall in the small, iterative in the large).

**This cycle stops at dev — prod is a separate, manual step**

A cut release is a tagged, built artifact — not a prod rollout. Nothing in
this skill, in either invocation mode, ever deploys to prod or triggers
`/loop` to do so. When a cut release is ready to ship to users, run
`/devops-deploy <service> to prod` yourself: it names the exact commit/tag
going out, checks a recent backup exists, and waits for your explicit
confirmation per the `/devops-deploy` contract — the same gate a human
release would go through. Point it at the tag this skill just cut
(`/devops-deploy <service> to prod` deploying commit `vX.Y.Z`), and treat
"cut" and "shipped to prod" as two different questions with two different
answers.

**Guardrails**

- **The PM scopes; engineers implement — never the reverse.** No
  worktree is created for an issue that is not on the umbrella's
  committed Phase 1 scope with a `model:*` label and rationale comment.
  An engineer-stage invocation that finds unscoped-but-ready issues
  hands them to the PM stage; it does not quietly implement them
- One coordination point: adopt an open sprint umbrella, never duplicate
  it; never run two open cycle umbrellas at once
- Scope changes are visible: every add/remove happens in the umbrella
  body plus a comment — no silent scope drift; >5 issues created or
  scoped in one invocation need the user's confirmation
- Flow first: Phase 2 items are never implemented while unblocked
  Phase 1 items remain open, even if they look quick
- Escalate, never downgrade: a `model:*` label only moves up while work
  is in flight; re-triaging downward requires the user's say-so
- **Verification is batched, one deployed version at a time.** Never
  deploy a single issue to dev so it can be verified alone, never split a
  pass across two commits, and never redeploy while a pass is running. An
  urgent verification is answered with the next deploy of the whole
  batch — the shared environment's known version is the asset here
- A merged PR is not a verified issue: `needs-verification` items stay on
  the umbrella's queue until a pass stamps them with the commit they were
  verified on
- All `/pm-sprint-delivery` guardrails bind the PM stage, all
  `/engineer-implement` guardrails bind the engineer stage, all
  `/devops-deploy` guardrails bind the DevOps stage, all
  `/qa-sprint-verify` guardrails bind the verification stage, and all
  `/release-cut` guardrails bind the release manager stage, unchanged
  (test-first, no weakened tests, no generated-code edits, honest test
  reporting, committed-state-only deploys, no moved/force-pushed tags,
  evidence for "done")
- Never merge the PRs this skill opens — review (`/reviewer-pr`) stays a
  separate role, same as for a human team
- **Dev only, never prod.** This skill's DevOps stage deploys to the
  dev/demo rung exclusively, and the release manager stage only tags and
  triggers the image build — it does not deploy that image anywhere. A
  prod release is always a separate, explicitly human-confirmed
  `/devops-deploy <service> to prod` call outside the cycle, run by the
  user when they choose to, never something this skill (or `--auto-release`)
  triggers on its own
- The cycle does not close on merged-but-undeployed or
  deployed-but-unverified work — a green dev release (or an explicit
  no-deployable-service note) plus a clear verification queue is required
  before close, same as a green test suite is required before a PR ships
- The release stage never runs ahead of a closed, dev-verified cycle —
  cutting a tag against an open umbrella or an unverified dev deploy is
  refused, not worked around
- Publishing pauses for confirmation by default; `--auto-release` is only
  honored when the user set it explicitly for this run — never turned on
  by inference, and never carried forward from one loop to the next
  without the user having set it for that loop
- Cap the blast radius: at most `--max` implementations per invocation
  (default 2); scoping and routing may cover everything, implementation
  may not
- A stalled fable-tier attempt gets flagged to humans, not retried in a
  loop — every invocation must terminate
- This skill runs the PM→engineer→DevOps(dev)→QA(verify)→release lane of the
  waterfall; product definition, architecture, PR review, prod deploy,
  and QA remain their own roles' skills
