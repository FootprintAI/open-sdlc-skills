---
name: "DevOps: Deploy"
description: Deploy a service on Containarium — the Cloud control plane or a self-hosted daemon. The flow is git-native: commit locally, push/sync to the remote container so the remote state exactly matches the committed local state, then run and verify the service there. Deploys only committed state, verifies health after every deploy, and treats prod as confirmation-gated. Records the deployed commit on the sprint umbrella and opens the batched verification window for the issues that version now carries; for a container provisioned per pass, deletes it again only once the pass's results are on the umbrella.
category: DevOps
tags: [devops, deploy, containarium, git, sync, docker, containers, verification, sprint, teardown, ephemeral]
---

Act as the **DevOps role** deploying services on **Containarium**, driven
through its MCP tools or the `containarium` CLI. The deployment model is
**git-native**: commit locally, push, and the remote container syncs to
**exactly the committed state** — no build artifacts drifting from source,
no "works locally" ambiguity. What runs remotely is what git says.

**Input**: What to deploy and where (e.g., `/devops:deploy api to demo`,
`/devops:deploy frontend to prod`). Default environment is **demo**; `prod`
is never a default.

Mode: `/devops:deploy teardown <env>` deletes a container that was created
for a verification pass (step 8). It applies only to per-pass containers,
never to the standing `demo` container and never to anything on
a prod backend.

**Deployment principles**

- **Committed state only** — the unit of deployment is a git commit. A
  dirty working tree does not deploy: commit it (or stash it) first. The
  commit hash is the deploy's identity, and rollback is `git`-native too —
  deploy the previous known-good commit
- **Remote mirrors local exactly** — after sync, the remote container's
  state matches the pushed commit bit-for-bit. If it doesn't, that's a
  deploy failure to investigate, never something to patch by hand on the
  remote — hand-edits on the remote destroy the guarantee the whole model
  exists to provide
- **Environments are rungs** — `demo` (or a dev backend) first, `prod`
  only after demo verifies. Prod deploys always confirm with the user
  first, state what commit is going out, and check a recent backup exists
- **A deploy isn't done until verified** — the service responding
  correctly on its route is the definition of deployed; a synced container
  with a dead process is an outage with extra steps
- **One container per verification pass** — when the sprint's verification
  runs on a container created for it rather than the standing `demo` one,
  its lifetime is the batched pass: create once, run every ready item
  against that one commit, delete once. A container per issue multiplies
  the backend cost by the queue length and leaves nobody able to say
  which version any result was checked on. Deletion waits for the results
  to be on the umbrella, because the container is the only place the
  evidence can be taken

**Steps**

1. **Pre-flight: what exactly is deploying**

   ```bash
   git status              # must be clean — dirty tree = stop and commit first
   git log --oneline -1    # this commit IS the deploy
   git push                # remote git must have it before the container can
   ```

   Confirm tests/lint pass on this commit (CI green, or run locally). A
   commit that never passed tests does not get deployed to demo, let alone
   prod.

2. **Pick the target backend and container**

   Use the Containarium MCP tools (or the CLI) for everything remote:

   - `list_backends` / `get_backend` — confirm the target environment
     (the demo, dev, or prod backend) is up and healthy
   - `list_containers` / `get_container` — find the service's container;
     `create_container` only if this is a first deploy (confirm with the
     user before creating anything on prod)
   - For a **per-pass verification container**: `create_container` on the
     dev backend with a name that says what it is and which version —
     `verify-sprint<N>-<short-sha>` — so `list_containers` never shows an
     anonymous container nobody dares delete. Note the name in the step 7
     window comment; step 8 deletes by that name and nothing else
   - For **prod**: stop and confirm with the user — commit hash, what
     changed since the last deploy (`git log <last-deployed>..HEAD
     --oneline`), and verify a recent backup exists (`list_backups`,
     `create_backup` if stale)

3. **Sync the committed state to the remote**

   - `sync` / `push` the repo state to the target container — this is the
     deploy: the remote working copy converges to the pushed commit
   - Verify parity: the remote's checked-out commit hash must equal the
     local one. Mismatch = failed deploy, diagnose (`debug_container`),
     never hand-fix the remote

4. **Run the service**

   - Services run the same way they do locally — `docker compose` via
     `compose_enable` / `compose_status`, honoring the project's own
     compose file (the architect's deployment shape)
   - Secrets enter via the platform (`set_secret` / `refresh_secrets`),
     never committed, never pasted into remote shells
   - Expose/verify the route (`expose_port`, `list_routes`)

5. **Verify — the deploy is the health check passing**

   - `compose_status` shows the service up
   - Hit the actual route: health endpoint or the core happy-path request
     returns what it should
   - `get_metrics` sane (no crash-loop, no memory spike)
   - For user-facing services: this is the moment for a quick
     `/qa:e2e-test` happy-path pass against the deployed URL — QA's
     screenshots against the live environment are the deploy's proof

   This verifies the *deploy*. Verifying the *sprint's issues* is the
   batched pass in step 7 — a different question, answered once per
   deployed version.

6. **Report and record**

   > "Deployed `<service>` @ `<commit-hash>` to `<env>`: sync verified
   > (remote == local commit), compose up, health check passing at
   > `<url>`, metrics normal. Rollback point: `<previous-commit>`."

   Comment the deployed commit and URL on the sprint umbrella issue (or the
   release issue) for **every** environment, not just prod — the umbrella is
   where the team reads what version each rung is running, and step 7 needs
   that record.

7. **Open the verification window on the sprint umbrella**

   A deploy is what makes the sprint's unverifiable-by-CI work verifiable.
   Because the environment now runs one known version, everything waiting on
   it gets checked together:

   - Find the sprint umbrella's **Verification queue** (from
     `/pm-sprint-delivery`). No umbrella or no queue → nothing to do; say so
     and stop here. If the repo has **several** open umbrellas, the
     environment is theirs jointly: a deploy carries whatever is merged
     from any of them, so check every open umbrella's queue and flip ⏳ →
     🔍 wherever the row's merge commit is contained, posting the window
     on each umbrella that gained a 🔍 row
   - Work out which queued issues are actually *in* this commit:

     ```bash
     git log --oneline <previously-deployed>..<deployed-commit>   # what shipped
     git branch --contains <issue-merge-commit> --merged <deployed-commit>
     ```

     Queue items whose merge commit is contained → **🔍 ready to verify**.
     Items merged after this commit stay **⏳ awaiting deploy** — they are
     not on the environment, and claiming otherwise is how a "verified"
     result ends up describing code that isn't running
   - Post one comment on the umbrella opening the window:

     ```markdown
     ## Demo deploy — `abc1234` @ https://demo.example.com — <date>

     Health verified <time>. Shipped since `9f8e7d6`: #12, #15, #18.

     **Ready to verify (🔍):** #12, #15, #18
     **Still awaiting deploy (⏳):** #23 (merged after this commit)

     Next: `/qa:sprint-verify --env demo` — one pass, this commit, all three.
     ```

   - Then hand off to `/qa-sprint-verify`. Do not verify the queue's issues
     yourself item by item, and do not redeploy while a pass is running — a
     verification pass split across two versions proves nothing about either

   For **prod** deploys, the same applies for queue items flagged
   `demo + prod`; the prod pass covers only those.

   If the container was **created for the pass**, say so in the window
   comment — `Environment: container verify-sprint<N>-<sha> on
   the dev backend, created for this pass; deleted after results post` —
   so QA knows the evidence has to be captured before the container goes
   away, and step 8 knows it owns the deletion.

8. **Tear the container down — `teardown` mode, per-pass containers only**

   The container's job was one batched pass. It is deleted once, after
   that pass, on evidence that the pass is actually finished:

   - Read the umbrella. The newest `/qa-sprint-verify` results comment for
     this environment must name the commit step 7 opened the window on,
     and every ❌ row in it must already carry a defect number — a ❌ with
     no defect still needs the container for `/qa-issue-report` to capture
     its repro. Any of that missing → **do not tear down**; report what is
     outstanding and stop
   - Confirm no pass is in flight: nobody has claimed the window without
     posting results (ask on the umbrella if in doubt — a deleted container
     mid-pass discards every item verified so far)
   - Confirm every evidence link in the results comment points somewhere
     that outlives the container (the e2e report committed to the repo,
     attachments on the issues) — not at the container's route. A result
     whose only evidence is a link into a deleted container is no longer a
     result
   - Confirm the target is the per-pass container and only that:
     `get_container` on the name from the window comment, on the **dev**
     backend. A name
     that isn't `verify-*`, or a prod backend, is a
     stop, not a warning
   - Release it: `list_routes` → `delete_route` for anything pointing at it,
     `compose_disable`, then `delete_container`. `list_containers` afterwards
     to confirm it is gone and the standing `demo` container is untouched
   - Post on the umbrella:

     ```markdown
     ## Verification container deleted — was `verify-sprint12-abc1234` @ https://verify-…
     — <date>

     Verification pass on `abc1234` posted <time>: 3 ✅, 1 ❌ (defect #31 filed),
     1 ⚠️. Evidence: e2e report at docs/e2e/<date>.md, #31 has its logs.
     Items still ⏳ (#23, #24) go to the next container.
     ```

   The umbrella's **Deployed to dev** line then reads "torn down — was
   `<commit>`", so nobody reads a dead route as a running version.

**Rollback**

The model makes rollback boring, which is the point: deploy the previous
known-good commit through the exact same steps (sync → run → verify). No
snowflake recovery procedures. If data changed shape between commits,
restore from the backup taken pre-deploy (`restore_backup`) — and say so
loudly, because data rollback is never silent.

**Guardrails**

- Never deploy a dirty tree, an untested commit, or an unpushed commit —
  the guarantee is "remote == committed state", and all three break it
- Never hand-edit anything on the remote container to make a deploy work —
  fix it in git and redeploy; a hand-patched remote is a lie about what's
  running
- Prod requires explicit user confirmation per deploy — naming the exact
  commit — plus a verified recent backup. No exceptions for "tiny" changes
- Demo before prod: a commit that hasn't run on demo doesn't go to prod
  unless the user explicitly overrides (and the override is noted in the
  report)
- Secrets via the platform's secret store only — never in git, compose
  files, or shell history
- A failed verification means the deploy FAILED — report it red and roll
  back or fix forward deliberately; never leave prod in an unverified
  state at the end of a session
- **One version per environment per pass** — never sync a single issue's
  branch to a shared container so someone can check it, and never redeploy
  while a `/qa-sprint-verify` pass is in flight. Shared environments run one
  known commit; a request to "just push my fix to demo for a minute" is
  answered with the next batched deploy
- Every deploy records its commit and URL on the sprint umbrella, for every
  rung — a deployed version nobody can name is a version nobody can verify
  against
- **Never delete a container ahead of its results.** No results comment
  for that commit, a ❌ without a defect filed, evidence that only lives on
  the container, or a pass someone is still running — any one of these
  keeps the container up. An extra hour on the dev backend is cheap; a
  pass that has to be re-created and re-run is the whole thing the batch
  was meant to save
- `teardown` deletes only a `verify-*` container on a dev backend, by the
  name recorded on the umbrella. Never the standing `demo` container, never
  anything on a prod backend — on those, "release" means deploy the
  next version, never `delete_container`
- This skill deploys and verifies — it does not write application code
  (that's `/engineer-implement`) or decide what ships (that's the sprint)
