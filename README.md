# open-sdlc-skills

A software team as [Claude Code agent skills](https://docs.claude.com/en/docs/claude-code)
— Product Manager, Software Architect, Project Manager, Scrum Master,
Software Engineer, QA Tester, Code Reviewer, DevOps, and Release Manager.

Each skill is one role with its own principles, deliverables, and
guardrails. Together they cover the whole software delivery lifecycle —
from "someone has a problem" to a tagged, deployed, e2e-proven release —
and the whole **software quality management** flow inside it: unit tests,
integration tests, e2e proof, defect reporting, and code review, each with
an explicit evidence gate.

The roles coordinate the way a real team does: through **GitHub issues and
PRs**, and **markdown docs committed in your repo**. There is no hidden
state, no database, no service to run. If you delete every skill tomorrow,
the artifacts they produced are still there and still readable.

Built and used daily by [FootprintAI](https://github.com/FootprintAI) —
these are the skills our own team ships with, not a demo. Licensed under
[Apache 2.0](LICENSE); no account, service, or vendor required to run any
of them.

## Roles

| Role | Skill | What it does |
|------|-------|--------------|
| **Product Manager** | [`/product:define`](product/define.md) | Turns a raw idea or user problem into a PRD: problem with evidence, target user, success metrics, MVP scope, and P0 user stories with testable acceptance criteria. Ruthlessly cuts to the smallest thing worth shipping, then hands the stories off as GitHub issues. |
| **Software Architect** | [`/architect:design`](architect/design.md) | Designs (or reviews) the technical solution. Opinionated stack — Docker, protobuf/gRPC + grpc-gateway, Next.js/React/Tailwind — across three sanctioned languages picked per component and justified in the doc: **Go** for services and CLIs, **Python** for ML/data (where the ecosystem *is* the reason), **TypeScript** for the browser and IO-bound backends. Test-driven design, with every boundary typed *and type-checked in CI* (`go vet` / `mypy --strict` / `tsc --noEmit`; Pydantic and zod parsing external input). Writes the design doc to `docs/architecture/`. |
| **Project Manager** | [`/pm:sprint-delivery`](pm/sprint-delivery.md) | Plans and coordinates weekly sprints. Prioritizes "make the flow work first, optimize next phase"; coordinates everything on a GitHub umbrella issue with linked child issues and weekly progress updates. |
| **Scrum Master** | [`/scrum:master`](scrum/master.md) | Facilitates a sprint that's already running. Rebuilds the board from *evidence* — commits, PRs, CI runs, claim timestamps — rather than status labels, then sorts it by how long each thing has been still. Names every impediment with a type, an owner, and an age (review starvation, stalled claim, broken CI, missing decision), enforces WIP limits, posts the standup on the umbrella issue, and runs the retro at cycle close. Owns *momentum* — never scope, code, or review verdicts. |
| **Team (automated cycle)** | [`/team:sprint-cycle`](team/sprint-cycle.md) | Runs a whole cycle end to end and enforces every hand-off: the PM stage scopes (umbrella + flow-first child issues), engineers implement each scoped issue in its own git worktree under a routed model tier (`model:*` label + rationale), DevOps releases merged work to dev, and once the cycle closes the release manager cuts the release. Escalates the model tier on stalls, never downgrades. Pair with `/loop` for a self-running cycle. |
| **Software Engineer** | [`/engineer:implement`](engineer/implement.md) | Claims the issue first — posts `${who}-${model} is starting processing it`, where `${who}` is the agent instance's own id, not the shared account it authenticates as, so parallel engineers stay distinguishable — then implements it test-first on the team stack, following the architecture design doc. Delivers a small, green, reviewable PR linked to its issue. No scope creep, no drive-by refactors. |
| **QA Tester** | [`/qa:unit-test`](qa/unit-test.md) | The fast lane: isolated, deterministic unit tests in Go/Python/TypeScript that mock underlying dependencies **only where a real object won't do** — real object > fake > stub > mock, and always at a seam you own, never a third party's internals. Classifies every dependency with a reason, proves each test can actually fail before trusting it, and runs green under `-race`/shuffle. |
| **QA Tester** | [`/qa:integration-test`](qa/integration-test.md) | The real-dependency lane: Postgres, Redis, Kafka, MinIO launched in throwaway containers via **Docker or rootless Podman** (Testcontainers or compose), pinned to the versions production runs. Random ports, readiness polling instead of sleeps, real migrations, per-test isolation, guaranteed teardown — and container logs as CI artifacts when it goes red. Its own CI lane, so the unit suite stays fast. |
| **QA Tester** | [`/qa:e2e-test`](qa/e2e-test.md) | Happy-path e2e run of a web/UI project with **a screenshot at every step as proof of execution**. Requests sample data (PDF/image/etc.) up front, cleans up test data afterward via the app's own API or UI, and writes a flow-by-flow report to `e2e/E2E-REPORT.md`. |
| **QA Tester** | [`/qa:issue-report`](qa/issue-report.md) | Files every defect found as a GitHub issue — one issue per finding, created *before* any fix is discussed. UI findings are verified live in a browser first (fresh defect screenshot + console errors), every issue records the app version/commit it was seen on, dedupes against existing issues, and cross-links related issues in both directions. |
| **Code Reviewer** | [`/reviewer:pr`](reviewer/pr.md) | Reviews a PR against the team contract: acceptance criteria covered by tests (verified by *running* them), no weakened tests, proto-as-source-of-truth, typed boundaries, scope matching the issue. Verdict posted on the PR page. |
| **DevOps** | [`/devops:deploy`](devops/deploy.md) | Deploys git-natively: commit, push, converge the environment to exactly that commit, verify health on the real route. Platform-agnostic — uses whatever the repo already has (CI deploy job, Kubernetes, Compose over SSH, a PaaS). Dev before prod; prod is confirmation-gated with a backup. |
| **Release Manager** | [`/release:cut`](release/cut.md) | Cuts a real release once work has shipped and been dev-verified: infers the semver bump and drafts categorized notes from PRs merged since the last tag, tags the codebase and GitHub at the same commit, publishes the GitHub Release, and triggers the release image build. Confirmation-gated; never moves or force-pushes a tag. |
| **Coordinator** | [`/coordinator:status`](coordinator/status.md) | Answers "where are we?" by reading each role's artifacts, gates stage advancement on evidence rather than claims, and when a cycle completes, closes it with a retro and re-triggers the next one from the PRD. |

## How the roles chain

Each cycle is a **waterfall** — stages in strict order, each gated on
evidence — and **cycles iterate**: when one closes,
`/coordinator:status cycle` re-triggers the next from the PRD (last cycle's
"optimize" phase becomes a candidate for this cycle's "make it work").

```
/coordinator:status    where are we?            → stage gates, next action, cycle re-trigger
      ↓ (oversees every stage below)
/product:define        what to build & why      → PRD + P0 stories as issues
      ↓
/architect:design      how to build it          → design doc + typed contracts
      ↓
/pm:sprint-delivery    when & who               → umbrella issue, flow-first sprint
      ↓
/scrum:master          why isn't it moving?     → standup, impediments, WIP, retro
      ↓ (runs daily alongside every stage below)
/engineer:implement    build it                 → test-first PRs, one per issue
      ↓
/qa:unit-test          pin the logic            → fast mocked lane, green under -race
/qa:integration-test   pin the boundaries       → real Postgres/Redis in containers
      ↓
/reviewer:pr           check it                 → verified review on the PR page
      ↓
/devops:deploy         ship it                  → git-native deploy, health verified
      ↓
/qa:e2e-test           prove it works           → screenshots + e2e report
/qa:issue-report       file what broke          → one GitHub issue per defect
      ↓
/release:cut           cut it                   → semver tag + GitHub Release
      ↺
cycle closes → retro → next PRD (waterfall in the small, iterative in the large)
```

`/team:sprint-cycle` automates the PM → engineer → DevOps → release lane of
that chain in one command. `/scrum:master` is its counterpart: the cycle
skill makes work *happen*, the scrum master finds out why it *stopped*.

The chain is deliberate: the PRD's MVP journey becomes the architect's core
flow, the sprint's Phase 1, and finally QA's happy path — **one journey,
verified at every stage**.

## Quality management, specifically

Quality here is not a review stage bolted on at the end; it's four
independent evidence gates, each producing an artifact someone else can
check:

| Gate | Skill | Evidence it produces |
|------|-------|----------------------|
| Logic is pinned | `/qa:unit-test` | A fast suite where every test has been *proven able to fail* |
| Boundaries are pinned | `/qa:integration-test` | A suite run against real Postgres/Redis/Kafka at production versions |
| The contract is met | `/reviewer:pr` | A review that ran the tests, not one that read them |
| The product works | `/qa:e2e-test` | A screenshot per step, in a report, against a deployed URL |
| Defects are tracked | `/qa:issue-report` | One GitHub issue per finding, deduped and cross-linked |

The recurring rule across all five: **claims don't pass gates, artifacts
do.** A skill that cannot produce the evidence reports the gate as *not
passed* rather than asserting success.

## Installation

Each skill is a single markdown file at `<namespace>/<name>.md`, invoked as
`/<namespace>:<name>`. To install for Claude Code:

```bash
git clone https://github.com/FootprintAI/open-sdlc-skills.git
cd open-sdlc-skills
for f in */*.md; do
  ns=$(dirname "$f"); name=$(basename "$f" .md)
  mkdir -p ~/.claude/skills/"$ns-$name"
  cp "$f" ~/.claude/skills/"$ns-$name"/SKILL.md
done
```

New Claude Code sessions pick the skills up automatically. To install only
some roles, copy just the files you want — the skills reference each other
by name but degrade gracefully when one isn't installed.

## How to use — a worked cycle

Install the skills, open a Claude Code session **in your product's repo**
(the skills read and write *that* repo's docs, issues, and PRs — not this
one), and drive one cycle:

```
# 1. Not sure where things stand? Always safe to start here:
/coordinator:status

# 2. Define what to build (writes docs/product/, files P0 issues)
/product:define let users bulk-import invoices

# 3. Design it (writes docs/architecture/)
/architect:design bulk-import

# 4. Plan the week (creates the sprint umbrella issue)
/pm:sprint-delivery ship bulk-import MVP

# 5. Build one issue at a time (test-first branch + PR per issue)
/engineer:implement #12

# 5b. Back the change with tests (fast lane, then the real-dependency lane)
/qa:unit-test internal/billing
/qa:integration-test repository layer

# 6. Review the PR (runs the tests itself, verdict on the PR page)
/reviewer:pr 20

# 7. Deploy the merged commit (dev first; prod asks for confirmation)
/devops:deploy api to dev

# 8. Prove it works (screenshots + e2e/E2E-REPORT.md; asks for sample data first)
/qa:e2e-test

# 8b. File everything QA found as linked GitHub issues (dedupes first)
/qa:issue-report e2e/E2E-REPORT.md

# 9. Cut the release — semver tag + GitHub Release
/release:cut

# 10. Close the cycle — retro, then the waterfall restarts at the PRD
/coordinator:status cycle
```

Run the scrum master alongside it, daily, for as long as the sprint is open:

```
/scrum:master                  # standup on the umbrella: what moved, what's stuck
/scrum:master impediments      # just the blocked register, routed to owners
/scrum:master retro            # at cycle close
```

Or hand the middle of the cycle to the automation:

```
/team:sprint-cycle ship bulk-import MVP     # PM → engineers → dev deploy → release
/loop 15m /team:sprint-cycle                # ...and keep it running
```

### Tips

- **Lost? Run `/coordinator:status`.** It reads the artifacts, tells you
  the current stage, and names the one next command to run.
- **Don't skip stages.** Each skill consumes the previous one's artifact
  (the engineer reads the design doc; QA tests the PRD's MVP journey). Ask
  `/coordinator:status advance` to check a gate instead of guessing.
- **Most role skills have a `review` mode** (`/product:define review`,
  `/architect:design review`) to critique an existing artifact instead of
  creating a new one.
- **Weekly rhythm:** `/pm:sprint-delivery update` posts the progress
  comment on the umbrella issue — run it every week, or wire it to `/loop`.
- **Prod is never automatic.** `/team:sprint-cycle` stops at dev. Ship to
  prod yourself with `/devops:deploy <service> to prod`, which names the
  exact commit and waits for your confirmation.
- **Skills propose before they act on anything hard to undo** — expect to
  be asked before issues are filed in bulk, scope is cut, or prod is
  touched.

## Adapting them to your team

These skills encode *one* team's opinions — a specific stack, a specific
definition of done, a specific prioritization rule. That's deliberate:
vague skills produce vague work. But the opinions are meant to be edited.

The most common changes:

- **Different stack?** Edit `architect/design.md` — the sanctioned
  languages and default components live there, and every downstream skill
  reads the design doc rather than hardcoding the stack.
- **Different deploy target?** `devops/deploy.md` already dispatches on
  what your repo contains; add your platform to step 2.
- **Different sprint length or ceremony?** `pm/sprint-delivery.md` assumes
  one week and an umbrella issue.

## Contributing

Issues and PRs welcome — see [CONTRIBUTING.md](CONTRIBUTING.md). The most
useful contributions are concrete: a role that's missing, a guardrail that
misfired on a real repo, or a stack the architect skill should know about.

## About

These skills are maintained by **[FootprintAI](https://github.com/FootprintAI)**,
where they run the actual sprint cycle for our own products. That's the
only claim we'd make for them: they're load-bearing somewhere.

We also build **[Containarium](https://containarium.dev)** — an
open-source agent runtime (SSH-native isolation, eBPF egress policy,
Kubernetes and LXC backends, GPU passthrough, MCP-native CLI). It's the
disposable-box layer these skills keep referring to: the fresh environment
`/engineer:implement` runs tests in, the box `/qa:e2e-test` deploys into,
the target `/devops:deploy` ships to.

**You do not need it.** Every skill here dispatches on what your repo
already has, and Containarium is one option among Kubernetes, Compose over
SSH, CI jobs, and PaaS targets — never a default and never assumed. If you
happen to want a purpose-built sandbox for agent workloads, it's Apache 2.0
too.

## License

[Apache License 2.0](LICENSE) — Copyright FootprintAI, Inc.
