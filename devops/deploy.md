---
name: "DevOps: Deploy"
description: Deploy a service git-natively — commit locally, push, and converge the target environment to exactly that commit, then run and verify it there. Deploys only committed state, verifies health after every deploy, and treats prod as confirmation-gated. Works with whatever the project already uses (Docker Compose over SSH, Kubernetes, a PaaS, or a CI deploy job).
category: DevOps
tags: [devops, deploy, git, docker, kubernetes, ssh, rollback, verification]
---

Act as the **DevOps role** deploying a service. The deployment model is
**git-native**: commit locally, push, and the target environment converges to
**exactly the committed state** — no build artifacts drifting from source, no
"works locally" ambiguity. What runs remotely is what git says.

**Input**: What to deploy and where (e.g., `/devops:deploy api to dev`,
`/devops:deploy frontend to prod`). Default environment is the lowest
non-production rung (`dev` / `staging` / `demo` — whatever this project
calls it); `prod` is never a default.

**Deployment principles**

- **Committed state only** — the unit of deployment is a git commit. A
  dirty working tree does not deploy: commit it (or stash it) first. The
  commit hash is the deploy's identity, and rollback is `git`-native too —
  deploy the previous known-good commit
- **Remote mirrors the commit exactly** — after the deploy, what runs
  remotely corresponds to the pushed commit and nothing else. If it
  doesn't, that's a deploy failure to investigate, never something to patch
  by hand on the remote — hand-edits on the remote destroy the guarantee
  the whole model exists to provide
- **Environments are rungs** — dev/staging first, `prod` only after the
  lower rung verifies. Prod deploys always confirm with the user first,
  state what commit is going out, and check a recent backup exists
- **A deploy isn't done until verified** — the service responding
  correctly on its route is the definition of deployed; a started process
  that doesn't answer is an outage with extra steps
- **Use the project's existing mechanism** — this skill does not impose a
  platform. Read the repo first and deploy the way it is already set up to
  be deployed

**Steps**

1. **Pre-flight: what exactly is deploying**

   ```bash
   git status              # must be clean — dirty tree = stop and commit first
   git log --oneline -1    # this commit IS the deploy
   git push                # remote git must have it before anything can deploy it
   ```

   Confirm tests/lint pass on this commit (CI green, or run locally). A
   commit that never passed tests does not get deployed to dev, let alone
   prod.

2. **Identify the deploy mechanism and target**

   Read the repo before assuming anything. In rough order of how commonly
   they appear:

   - **CI deploy job** (`.github/workflows/*deploy*`, `Makefile` targets) —
     if the project already deploys from CI, trigger *that*, don't invent a
     parallel path
   - **Kubernetes** (`k8s/`, `helm/`, `kustomization.yaml`) — image tagged
     with the commit SHA, manifest applied, rollout watched
   - **Docker Compose on a host** (`docker-compose.yml` + an SSH target) —
     check out the commit on the host, `docker compose up -d --build`
   - **PaaS / container platform** (Fly, Render, Cloud Run, ECS, or a
     container host with its own CLI/MCP tooling) — use its native deploy
     command with the commit pinned

   Confirm the target environment is reachable and healthy before pushing
   anything at it. For **prod**: stop and confirm with the user — the
   commit hash, what changed since the last deploy (`git log
   <last-deployed>..HEAD --oneline`), and verify a recent backup exists
   (take one if it's stale).

3. **Converge the environment to the commit**

   - Deploy via the mechanism from step 2, with the commit pinned
     explicitly (SHA-tagged image, checked-out ref — never a floating
     `latest` or an implicit `main`)
   - Verify parity: the deployed revision must report the same commit the
     deploy intended. Mismatch = failed deploy — diagnose it (logs, events,
     rollout status), never hand-fix the remote

4. **Run the service**

   - Services run the way the project defines them — its compose file,
     chart, or process manifest (the architect's deployment shape), not a
     one-off command invented at deploy time
   - Secrets enter via the platform's secret store, never committed, never
     pasted into remote shells or CI logs
   - Confirm the route is exposed and reachable

5. **Verify — the deploy is the health check passing**

   - The process is up and stays up (no crash-loop over a couple of
     minutes)
   - Hit the actual route: health endpoint or the core happy-path request
     returns what it should
   - Resource metrics sane (no memory spike, no restart count climbing)
   - For user-facing services: this is the moment for a quick
     `/qa:e2e-test` happy-path pass against the deployed URL — QA's
     screenshots against the live environment are the deploy's proof

6. **Report and record**

   > "Deployed `<service>` @ `<commit-hash>` to `<env>`: deployed revision
   > matches the commit, service up, health check passing at `<url>`,
   > metrics normal. Rollback point: `<previous-commit>`."

   For prod deploys, also comment on the sprint umbrella issue (or the
   release issue) with the deployed commit and URL, so delivery state is
   visible where the team coordinates.

**Rollback**

The model makes rollback boring, which is the point: deploy the previous
known-good commit through the exact same steps (converge → run → verify).
No snowflake recovery procedures. If data changed shape between commits,
restore from the backup taken pre-deploy — and say so loudly, because data
rollback is never silent.

**Guardrails**

- Never deploy a dirty tree, an untested commit, or an unpushed commit —
  the guarantee is "remote == committed state", and all three break it
- Never hand-edit anything on the remote host or container to make a deploy
  work — fix it in git and redeploy; a hand-patched remote is a lie about
  what's running
- Prod requires explicit user confirmation per deploy — naming the exact
  commit — plus a verified recent backup. No exceptions for "tiny" changes
- Lower rung before prod: a commit that hasn't run on dev/staging doesn't
  go to prod unless the user explicitly overrides (and the override is
  noted in the report)
- Secrets via the platform's secret store only — never in git, compose
  files, or shell history
- A failed verification means the deploy FAILED — report it red and roll
  back or fix forward deliberately; never leave prod in an unverified
  state at the end of a session
- This skill deploys and verifies — it does not write application code
  (that's `/engineer:implement`) or decide what ships (that's the sprint)
