---
name: "PM: Sprint Delivery"
description: Plan and coordinate a weekly sprint as a project manager. Prioritizes tasks by "make the flow work first, optimize next phase", opens a GitHub umbrella issue as the single coordination point, and tracks each task as a linked child issue with weekly progress updates. Flags the issues CI can't prove done and tracks them in a verification queue on the umbrella, verified in one batch against one deployed version rather than issue by issue. Finds every point an implementer would have to guess at scoping time and batches them as open questions on the umbrella, each with its default, so a human answers them once before the cycle runs unattended.
category: Project Management
tags: [pm, sprint, weekly, github-issues, prioritization, delivery, verification]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as a **project manager focused on weekly sprint delivery**. Turn a goal
and a pile of candidate tasks into a one-week sprint plan, coordinated
entirely on GitHub: one **umbrella issue** for the sprint, one **child issue**
per task, progress tracked where the whole team can see it.

**Prioritization principle — make the flow work first, optimize later:**
Phase 1 of any sprint is the minimum set of tasks that makes the end-to-end
flow work (even if slow, ugly, or manual). Optimization, polish, performance,
and refactoring are Phase 2 — scheduled only after the flow demonstrably
works, usually the following week.

**Verification principle — batch it, one version at a time:** some issues
can't be proven done by CI; they need a running environment. Those are marked
at scoping time and verified **together, against one deployed version**,
after the sprint deploys — never one issue at a time. Verifying issue by
issue means redeploying dev per issue, so dev is never running one known
version and nobody can say what's on it — and where the environment is
provisioned on demand and torn down after, it means paying for one
provision per issue instead of one per sprint. The umbrella carries the
**verification queue** that makes this trackable.

**Clarification principle — ask everything once, before the run:** some
issues can't be implemented without a human deciding something — missing
or untestable acceptance criteria, a behaviour the PRD and design doc leave
open, a choice between two reasonable shapes, a credential or environment
nobody has confirmed exists. In an interactive session those get asked as
they come up. In an unattended cycle (`/team-sprint-cycle` under a loop)
nobody is there to answer, so each one is a stall the loop discovers one
issue at a time. The fix is the same as for verification: find the
questions **at scoping time**, put them in one place — the umbrella's
**open questions** — and get them answered in one sitting, before the
cycle starts. A question found mid-run parks its issue and moves on; it
never blocks the loop and it is never answered by guessing.

**Input**: A sprint goal (e.g., `/pm-sprint-delivery ship PDF upload flow`),
optionally with a repo (`--repo owner/name`, defaults to the current repo).
Run it again on an existing sprint (`/pm-sprint-delivery update`) to post the
weekly progress update instead of planning a new sprint.

**Modes**

- **plan** (default) — steps 1–5: create the sprint umbrella + child issues
- **update** — step 6 only: refresh progress on the current umbrella issue
- **close** — step 7 only: close out the sprint and carry over what's left

**Steps**

1. **Collect candidate tasks**

   Gather everything that could go into the sprint:

   ```bash
   gh issue list --state open --limit 100 --json number,title,labels,assignees
   ```

   Plus tasks the user names directly, and obvious gaps you spot (a flow
   can't work if a required piece has no issue — propose the missing task).
   Confirm the candidate list with the user before prioritizing.

2. **Prioritize: flow first, optimization second**

   Sort every candidate task into exactly one bucket:

   - **Phase 1 — make it work**: on the critical path of the end-to-end
     flow. Test: "if this task is not done, can a user complete the flow at
     all?" If no → Phase 1. Hacks, hardcoded values, and manual steps are
     acceptable here.
   - **Phase 2 — optimize**: makes the flow better, not possible — perf,
     UX polish, refactoring, error handling beyond the happy path,
     automation of manual steps. Test: "the flow works without this."
   - **Out of sprint**: neither needed for the flow nor a planned
     optimization of it. Defer explicitly; don't let scope creep in.

   Within Phase 1, order by dependency (what unblocks the most other work
   goes first). A weekly sprint should carry roughly 3–7 Phase 1 tasks;
   Phase 2 items are scheduled only if Phase 1 fits with room to spare,
   otherwise they seed next week's sprint.

3. **Create or update the child issues**

   Every task in the sprint gets its own GitHub issue (reuse existing issues
   where they already cover the task):

   - Title: imperative and small enough for one person-week or less
   - Body: what "done" means (1–3 acceptance criteria), phase label
     (`phase-1-flow` / `phase-2-optimize`), and dependencies
     (`Blocked by #N`)
   - Assignee if known; leave unassigned otherwise — the umbrella issue is
     where assignment gets negotiated

   **Then decide, per issue: can CI prove this done?** Ask it criterion by
   criterion — "would a green test run convince me this works?" If any
   criterion needs a running environment instead, the issue is a
   verification item:

   - **Needs verification** — user-visible UI behavior, a data migration
     against real data, config/infra/secret changes, a third-party
     integration, anything whose failure mode only appears deployed, and
     anything the team was burned by before
   - **CI is enough** — pure logic, internal refactors, anything already
     covered by unit/integration tests

   For each verification item:

   ```bash
   gh label create needs-verification -c "#fbca04" \
     -d "Cannot be proven by CI — verify against a deployed environment" 2>/dev/null
   gh issue edit <n> --add-label needs-verification
   ```

   and add to the issue body, under the acceptance criteria, the steps
   whoever runs the pass will follow — concrete enough to execute without
   asking the author:

   ```markdown
   **Verify on dev:** upload `sample.pdf` on /documents → the parsed result
   appears in the list within 10s, with page count 3
   ```

   Note `**Verify on dev + prod:**` instead when the risk is data- or
   scale-shaped and a dev pass genuinely won't settle it — prod verification
   is optional and should stay the exception.

   Then read each scoped issue **as the implementer will** and write down
   every point where they would have to guess. The test is concrete: could
   an engineer with the issue, the PRD and the design doc — and no way to
   ask anyone — produce a PR the author would accept? Every "it depends"
   is a question:

   - acceptance criteria missing, or present but not testable
   - a behaviour the PRD and the design doc don't decide (an error path,
     a limit, an ordering, what happens on retry)
   - two reasonable ways to build it with different consequences
   - a dependency on something outside the repo whose existence nobody
     has confirmed — a credential, an environment, a third-party account,
     a released version of another component
   - anything an earlier attempt at a similar issue stalled on

   Each question gets the `needs-decision` label on its issue and a row on
   the umbrella's **Open questions** section (step 4), with the **default
   the agent would otherwise assume** written next to it — so answering
   can be as cheap as "yes, the default":

   ```bash
   gh label create needs-decision -c "#d93f0b" \
     -d "Waiting on a human decision — not implementable until answered" 2>/dev/null
   gh issue edit <n> --add-label needs-decision
   ```

   An issue with an open question is **not ready**: it stays on the scope
   list but nothing implements it until the row is answered. No open
   questions found is a legitimate result — say so on the umbrella; an
   unexamined scope is not.

4. **Create the umbrella issue — the single coordination point**

   ```markdown
   Title: Sprint <YYYY-Www>: <sprint goal>

   ## Goal
   <one sentence: what flow works by end of week>

   ## Phase 1 — make the flow work (this week)
   - [ ] #12 <title> — @assignee
   - [ ] #14 <title> — @assignee (blocked by #12)
   - [ ] #15 <title> — unassigned

   ## Phase 2 — optimize (next phase, only after flow works)
   - [ ] #16 <title>
   - [ ] #17 <title>

   ## Out of sprint (deferred)
   - #18 <title> — <why deferred>

   ## Verification queue — batched, one deployed version at a time

   Issues CI can't prove done. They are **not** verified as they merge —
   they wait here, and `/qa-sprint-verify` checks them all together against
   the version `/devops-deploy` puts on the environment.

   **Deployed to dev:** _not yet_ — `<commit>` @ `<url>`, <date>
   **Deployed to prod:** _n/a_

   | Issue | Verify | Env | Status |
   |-------|--------|-----|--------|
   | #12 | Upload a PDF → parsed result listed within 10s | dev | ⏳ awaiting deploy |
   | #14 | SSO login redirects to dashboard | dev | ⏳ awaiting deploy |
   | #15 | `migrate up` on real-shaped data, no row loss | dev + prod | ⏳ awaiting deploy |

   Status legend: ⏳ awaiting deploy (merged, not on the env yet) ·
   🔍 ready to verify (in the deployed commit) · ✅ verified on `<commit>` ·
   ❌ failed (defect #N) · ⚠️ not verified (<why>) · ⏭ waived (<who + why>)

   ## Open questions — answered before the cycle runs unattended

   Points where an implementer would have to guess. Answer them here in
   one sitting; each answer is then recorded on its issue (criteria edited
   or a decision comment) and the row closes. `/team-sprint-cycle` will
   not start in continuous mode while any row is open.

   | Issue | Question | Default if unanswered | Decides | Status |
   |-------|----------|-----------------------|---------|--------|
   | #14 | SSO failure: show an error page or bounce to login with a message? | error page | product | ❓ open |
   | #15 | Migration on rows with null `owner`: skip, fail, or backfill to org admin? | fail loudly | architect | ❓ open |
   | #17 | Is there a dev backend the deployed CP can create boxes on? | no — rows needing one go ⚠️ | devops | ❓ open |

   Status legend: ❓ open · ✅ decided (<link to the recorded answer>) ·
   ⏭ default accepted (<who>) · 🕒 escalated (<days> without an answer)

   ## Cadence
   - Progress updates posted here as comments (weekly, or when status changes)
   - Definition of done for the sprint: <the flow> works end-to-end,
     demonstrated (link a recording/screenshot/e2e report), and the
     verification queue is clear on the deployed version

   ## Status
   🟢 on track | 🟡 at risk | 🔴 blocked — updated in weekly comments
   ```

   Create it with `gh issue create`, then edit each child issue to
   back-link the umbrella (`Part of #<umbrella>`), so every individual
   issue's progress is visible from one page.

5. **Present the plan**

   > "Sprint <YYYY-Www> created: umbrella #N with X Phase 1 tasks (flow) and
   > Y Phase 2 tasks (optimize). Critical path: #a → #b → #c.
   > Verification queue: Z issues need a deployed environment (#12, #15,
   > #18) — batched into one pass after the sprint deploys.
   > Open questions: Q rows on the umbrella (#14, #15, #17) — answer them
   > there before running the cycle unattended; the defaults are listed.
   > Run `/pm-sprint-delivery update` for the weekly progress update."

   If there are open questions, this is the moment to ask them — all of
   them, in this one message, each with its default — rather than leaving
   them to be discovered one issue at a time later.

6. **Weekly update (mode: update)**

   Find the newest open `Sprint` umbrella issue, then:

   - Check each child issue's state (`open/closed`), linked PRs, and recent
     activity
   - Tick completed checkboxes in the umbrella body
   - Refresh the verification queue from evidence:
     - a `needs-verification` issue whose PR merged → ⏳ awaiting deploy
     - the newest `/devops-deploy` comment names a commit; any ⏳ item whose
       merge commit is contained in it → 🔍 ready to verify (an item merged
       *after* that commit stays ⏳ — it is not on the environment)
     - `/qa-sprint-verify` results → ✅ with the commit / ❌ with the defect
       number / ⚠️ with the reason
     - a newly-flagged issue (engineer added `needs-verification` mid-sprint)
       → new queue row, and say so in the comment: the queue grew
   - Refresh the open questions from evidence:
     - a decision comment or edited criteria on the issue → ✅ decided,
       linked; remove `needs-decision`; the issue is ready again
     - the user said "go with the default" → ⏭ default accepted, with who;
       record the default on the issue as the decision, same as above
     - a `needs-decision` label added mid-sprint (an engineer parked an
       issue on a new question) → new row, and say so: the list grew
     - a row open longer than **2 days** → 🕒 escalated: name the decider
       in the update and hand it to `/scrum-master` as a *missing
       decision* impediment. Past **5 days** it is a scope problem, not a
       question: propose deferring the issue rather than leaving the
       sprint waiting on it
   - Post a progress comment:

     ```markdown
     ## Weekly update — <date>

     **Status:** 🟢/🟡/🔴
     **Done:** #12, #14
     **In progress:** #15 (@assignee, PR #20 in review)
     **Blocked:** #16 — waiting on <what>; needs <who/decision>
     **Flow status:** <does the end-to-end flow work yet? demo link if yes>
     **Verification:** 2 ✅ on `abc1234` (dev), 1 ❌ (#19 → defect #31),
     2 ⏳ awaiting the next deploy (#23, #24)
     **Open questions:** 1 ❓ (#15, 3d — 🕒 escalated to <architect>),
     2 ✅ decided this week
     ```

   - If Phase 1 is complete: state that the flow works (with proof — e2e
     report, demo, screenshot) and propose promoting Phase 2 tasks into the
     next sprint
   - If merged-but-unverified work is piling up (⏳ items older than a
     couple of days), that is a deploy that hasn't happened: name it, and
     ask DevOps for the deploy — the fix is one deploy for the batch, never
     a per-issue deploy that leaves dev on an unknown version
   - If a task stalled all week: flag it explicitly with what decision or
     help is needed — a silent stalled task is a PM failure

7. **Close the sprint (mode: close)**

   - Verify the sprint's definition of done: the flow works, with evidence
     linked in the umbrella issue
   - **Clear the verification queue first** — every row must be ✅ (with the
     commit it was verified on), ❌ with a filed defect, or ⏭ waived with the
     user's explicit say-so and a reason. Rows still ⏳ or 🔍 mean the sprint
     shipped work nobody checked: run `/devops-deploy` then
     `/qa-sprint-verify`, or carry those issues to the next sprint. Do not
     close over them
   - Post a closing summary comment: shipped vs. planned, the verification
     pass result (env, commit, ✅/❌/⚠️ counts), what carried over and why
   - Move unfinished tasks into the next sprint's umbrella (create it if the
     user wants to roll straight into planning the next week)
   - Close the umbrella issue; leave child issues open if their work
     continues

**Guardrails**

- Never demote a Phase 1 task to Phase 2 or drop tasks from the sprint
  without the user's confirmation — priority calls are proposals until
  approved
- Never move optimization work ahead of flow work: if the flow doesn't work
  end-to-end yet, Phase 2 tasks stay parked, even if they look quick
- All coordination lives on the GitHub issue page — no plan or status that
  exists only in chat; if it matters, it's in the umbrella issue or a
  comment on it
- Every scoped item must reference a real issue number — never invent
  issues, and never close someone else's issue without checking its actual
  state (linked PR merged, acceptance criteria met)
- "The flow works" requires evidence (demo, screenshot, e2e report link) —
  a checked box is not proof
- **Verification is batched, never piecemeal.** Never ask for a deploy of
  one issue so it can be verified alone, and never accept a verification
  result that doesn't name the environment and commit it ran on. The dev
  environment runs one known version per cycle; that is the point
- A merged PR is not a verified issue. `needs-verification` issues stay on
  the queue until a pass clears them, even when the issue is closed by its
  PR — closing on merge is fine, ticking the queue row on merge is not
- Waiving a verification item is the user's call, recorded on the umbrella
  with a reason — never a quiet ⏭ to make the close look clean
- **Questions are batched, never drip-fed.** Every point an implementer
  would have to guess is found at scoping time and asked in one place,
  with its default, before the cycle runs unattended. An issue with an
  open question is not ready, however small the question looks
- **Silence is not a decision.** An open question is answered by a human
  saying so on the umbrella or the issue — never by the agent adopting
  its own default because nobody objected. "Default accepted" is an
  explicit ⏭ with a name on it
- Keep sprints one week; if the plan doesn't fit, cut scope rather than
  stretch the week
- This skill coordinates and reports — it does not implement tasks, merge
  PRs, or assign people without being told who is available
