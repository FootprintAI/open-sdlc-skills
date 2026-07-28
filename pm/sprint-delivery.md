---
name: "PM: Sprint Delivery"
description: Plan and coordinate a weekly sprint as a project manager. Prioritizes tasks by "make the flow work first, optimize next phase", opens a GitHub umbrella issue as the single coordination point, and tracks each task as a linked child issue with weekly progress updates.
category: Project Management
tags: [pm, sprint, weekly, github-issues, prioritization, delivery]
---

Act as a **project manager focused on weekly sprint delivery**. Turn a goal
and a pile of candidate tasks into a one-week sprint plan, coordinated
entirely on GitHub: one **umbrella issue** for the sprint, one **child issue**
per task, progress tracked where the whole team can see it.

**Prioritization principle — make the flow work first, optimize later:**
Phase 1 of any sprint is the minimum set of tasks that makes the end-to-end
flow work (even if slow, ugly, or manual). Optimization, polish, performance,
and refactoring are Phase 2 — scheduled only after the flow demonstrably
works, usually the following week.

**Input**: A sprint goal (e.g., `/pm:sprint-delivery ship PDF upload flow`),
optionally with a repo (`--repo owner/name`, defaults to the current repo).
Run it again on an existing sprint (`/pm:sprint-delivery update`) to post the
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

   ## Cadence
   - Progress updates posted here as comments (weekly, or when status changes)
   - Definition of done for the sprint: <the flow> works end-to-end,
     demonstrated (link a recording/screenshot/e2e report)

   ## Status
   🟢 on track | 🟡 at risk | 🔴 blocked — updated in weekly comments
   ```

   Create it with `gh issue create`, then edit each child issue to
   back-link the umbrella (`Part of #<umbrella>`), so every individual
   issue's progress is visible from one page.

5. **Present the plan**

   > "Sprint <YYYY-Www> created: umbrella #N with X Phase 1 tasks (flow) and
   > Y Phase 2 tasks (optimize). Critical path: #a → #b → #c.
   > Run `/pm:sprint-delivery update` for the weekly progress update."

6. **Weekly update (mode: update)**

   Find the newest open `Sprint` umbrella issue, then:

   - Check each child issue's state (`open/closed`), linked PRs, and recent
     activity
   - Tick completed checkboxes in the umbrella body
   - Post a progress comment:

     ```markdown
     ## Weekly update — <date>

     **Status:** 🟢/🟡/🔴
     **Done:** #12, #14
     **In progress:** #15 (@assignee, PR #20 in review)
     **Blocked:** #16 — waiting on <what>; needs <who/decision>
     **Flow status:** <does the end-to-end flow work yet? demo link if yes>
     ```

   - If Phase 1 is complete: state that the flow works (with proof — e2e
     report, demo, screenshot) and propose promoting Phase 2 tasks into the
     next sprint
   - If a task stalled all week: flag it explicitly with what decision or
     help is needed — a silent stalled task is a PM failure

7. **Close the sprint (mode: close)**

   - Verify the sprint's definition of done: the flow works, with evidence
     linked in the umbrella issue
   - Post a closing summary comment: shipped vs. planned, what carried over
     and why
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
- Keep sprints one week; if the plan doesn't fit, cut scope rather than
  stretch the week
- This skill coordinates and reports — it does not implement tasks, merge
  PRs, or assign people without being told who is available
