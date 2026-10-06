---
name: "DevOps: Containarium Env"
description: Set up and prove the Containarium environment on the runtime an agent runs in — control-plane MCP server, the CLI and ssh path, a build-credential path for private modules, committed dev config, and the permission rules — so /devops-deploy and /qa-sprint-verify can run a batched verification pass unattended instead of handing it back to a human. Checks every capability with a command and reports what is missing; credential-bearing steps stay with the human.
category: DevOps
tags: [devops, containarium, mcp, agent-runtime, bootstrap, ssh, secrets, permissions, verification, sprint]
---

Act as the **DevOps role** provisioning the *agent's own runtime* — the
laptop, CI runner, or agent box a Claude Code session runs in — so that the
sprint's deploy and verification stages can execute there end to end. The
role owns **capability**: that every tool the chain needs is present,
authenticated, allow-listed, and proven by a command. It does not own the
deploy (`/devops-deploy`), the pass (`/qa-sprint-verify`), or the target
repo's code.

**Why this exists.** An agent asked to run a verification pass often cannot:
no control-plane access, no build token for a private module, a dev-only
daemon flag nobody committed, a box whose user services die when the ssh
session closes. Each is a runtime gap, not a deploy problem — and each ends
with the agent handing steps back to a human mid-deploy. A runtime that has
been through this skill hands none of them back.

**Input**: `/devops-containarium-env` (default mode `check`). Modes:

- `check` — prove each capability below with a command; report the
  matrix and stop. Never changes anything
- `setup` — walk the gaps `check` found: agent-side steps run, human-side
  steps (anything that creates or moves a credential) are handed over as
  exact commands, then `check` re-runs
- `box <name>` — apply the per-pass box defaults (capability 8) to a box
  `/devops-deploy` just created

Options: `--profile dev|prod` (default `dev`; `prod` runs read-only checks
only), `--repo owner/name` (the target repo whose deploy this runtime must
be able to perform; defaults to the current repo), `--issue N` (where the
readiness report is posted; defaults to the newest open sprint umbrella).

**Principles**

- **Prove, don't assume.** A capability is present when its check command
  succeeded in this session, with output. "The MCP server is configured"
  is a claim; `list_backends` returning a healthy backend is evidence
- **Credentials never pass through the agent.** The agent references
  where a credential lives (a token file, a tenant secret name); a human
  creates it and puts it there. The harness refuses credential moves for
  a reason — design the path so none is needed, instead of asking for the
  refusal to be lifted
- **Allow-list the exact commands, not the shell.** Unattended runs need
  permission rules for the specific ssh, curl, `gh`, and git invocations
  the chain uses. `Bash(*)` or skipping permissions is not a setup step
- **Dev configuration is committed in the target repo.** A dev-only flag
  the daemon needs (a stub backend, a placeholder value a compose guard
  demands) belongs in a committed, reviewed dev override in that repo —
  never authored on a box by the agent at deploy time. Missing → file an
  issue on the target repo; it is their gap
- **Per-pass boxes get the same defaults every time.** Linger, registry
  search, tooling, naming. A box that dies when the ssh session closes is
  not a deploy target
- **Prod is checked, never touched.** The `prod` profile proves the
  control plane answers; it creates nothing and changes nothing

**Capabilities and their checks**

| # | Capability | Check (must succeed in-session) |
|---|------------|---------------------------------|
| 1 | Tracker CLI | `gh auth status` (or `glab`, per `_shared/tracker.md`), can comment on the target repo's issues |
| 2 | Control-plane MCP | The Containarium MCP server is registered; `list_backends` → ≥1 healthy backend; `list_containers` answers |
| 3 | CLI + ssh config | `containarium version --server <url>` reports a release; `containarium ssh-config sync` writes `~/.containarium/ssh_config` |
| 4 | Sentinel ssh path | `~/.containarium/keys/` exists (0700); TCP 22 to the sentinel opens; every ssh the chain runs carries `-o IdentitiesOnly=yes` |
| 5 | Build credential path | The **daemon that owns the box** holds `GH_TOKEN` for the tenant (`containarium secrets list <tenant> --server <daemon>` — names only; a hosted control plane does not hold tenant plaintext and answers 501 for secrets by design); a fresh box sees it as env; the build recipe reads it from env into a 0600 file it deletes after the build |
| 6 | Committed dev config | The target repo carries a documented dev override (compose override / `.env.dev.example`) covering every dev-only flag its daemon needs; `git ls-files` finds it |
| 7 | Permission rules | Claude Code settings allow the chain's commands (list below); a dry `gh issue comment --help` and an `ssh -G` with the keys dir run without a prompt |
| 8 | Per-pass box defaults | On a fresh box: `loginctl show-user $USER -p Linger` → `yes`; `podman info` rootless with `docker.io` in unqualified search; `tmux -V` present |
| 9 | Tracker connection allow-list | When work is dispatched through a Containarium tracker connection: the connection's label allow-list contains every label the cycle applies — the type labels (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`), `runtime:*`, `size:*`, `model:*` and `scope:*` — and `_shared/tracker-containarium.md` is installed. Labels outside the list are rejected *before* the tracker is touched, so the failure is a refused issue, not a warning. Read the allow-list from the connection config (admin-scoped; `containarium tracker` CLI); if it cannot be read, report `unknown`, never `ok`. Not applicable when the repo uses plain `gh`/`glab` |

**Steps**

1. **Resolve targets**

   Target repo (`--repo` or `git remote get-url origin`), tracker via
   `_shared/tracker.md`, control-plane profile, and the sentinel host —
   read it from the MCP server's `CONTAINARIUM_SERVER_URL` and from a
   container's `sshHost` (`get_container`), never guessed.

2. **Run every check (mode `check`)**

   Run them all before reporting — a partial matrix hides the gap that
   matters. Concretely:

   ```bash
   # 1 tracker
   gh auth status && gh api repos/<owner>/<repo> --jq .full_name

   # 3 CLI (binary from the OSS release: containarium-<os>-<arch>)
   containarium version --server "$CONTAINARIUM_SERVER_URL"
   containarium ssh-config sync && grep -c '^Host ' ~/.containarium/ssh_config

   # 4 sentinel path (python: same answer on macOS/BSD/GNU stat)
   python3 -c 'import os,sys;print(oct(os.stat(sys.argv[1]).st_mode&0o777))' ~/.containarium/keys   # 0o700
   nc -z -w 5 <sentinel-host> 22 && echo "sentinel: reachable"

   # 6 committed dev config in the target repo
   git ls-files 'deploy/*dev*' 'deploy/*.override*' '.env*dev*'
   ```

   For 2, use the MCP tools (`list_backends`, `list_containers`). For 5,
   use the CLI against the daemon that owns the box — a hosted control
   plane's `list_secrets` / `set_secret` return 501 (secrets live with the
   daemon, never the control plane), which also makes capability 3 a
   prerequisite of 5. For 7, read the settings files
   (`~/.claude/settings.json`, project `.claude/settings.json`) and match
   the rule list below. For 8, only when a box name was given.

   Never read a token's value to "check" it. The check for a credential is
   that the tool using it works.

3. **Report the readiness matrix — the artifact**

   Post one comment on `--issue` (default: the open sprint umbrella), and
   keep a copy at `~/.containarium/agent-env-<date>.md`:

   ```markdown
   ## Agent runtime readiness — <host-or-runner> — <date>

   Profile: dev (`<server-url>`), target repo: owner/repo

   | # | Capability | State | Evidence |
   |---|------------|-------|----------|
   | 1 | Tracker CLI | ✅ | `gh auth status`: logged in, repo scope |
   | 2 | Control-plane MCP | ✅ | `list_backends`: backend-1 healthy |
   | 3 | CLI + ssh config | ❌ | `containarium: command not found` |
   | 4 | Sentinel ssh path | ✅ | keys dir 700; sentinel:22 reachable |
   | 5 | Build credential path | ❌ | `secrets list`: no `GH_TOKEN` |
   | 6 | Committed dev config | ❌ | no dev override in deploy/ — issue #N filed |
   | 7 | Permission rules | ⚠️ | ssh rule present; `gh issue edit` rule missing |
   | 8 | Per-pass box defaults | – | no box given |
   | 9 | Tracker connection allow-list | – | repo uses plain `gh` |

   **Verdict:** NOT ready — a deploy from this runtime hands back 3, 5, 6.
   Next: `/devops:containarium-env setup`
   ```

   A ✅ without its evidence column filled is not a ✅. The verdict names
   exactly which capabilities a deploy would hand back to a human.

4. **Close the gaps (mode `setup`) — split by who may do it**

   **Agent-side** (runs now, no credential involved):

   - Install the CLI from the OSS release matching the control plane:
     `gh release download <tag> --repo FootprintAI/Containarium
     -p 'containarium-<os>-<arch>' -O ~/.local/bin/containarium && chmod
     +x ~/.local/bin/containarium`, then `ssh-config sync`
   - Create `~/.containarium/keys` (0700) if missing
   - Add the permission rules (via `/update-config`), scoped to the
     chain's real invocations — start from:

     ```
     Bash(ssh -i ~/.containarium/keys/*:*)
     Bash(scp -i ~/.containarium/keys/*:*)
     Bash(nc -z:*)
     Bash(curl -s -m * https://*.containarium.dev/*)
     Bash(containarium *)
     Bash(gh issue view:*)   Bash(gh issue comment:*)   Bash(gh issue edit:*)
     Bash(gh pr view:*)      Bash(gh api repos/*)        Bash(gh release view:*)
     Bash(git push containarium-*:*)
     ```

     These cover what `/devops-deploy` and `/qa-sprint-verify` actually
     run. They do not cover remote `sudo`, piping a token over ssh, or
     writing a daemon "allow stub" flag — the harness refuses those as
     outcomes, and the fix is capabilities 5, 6, and 8, not a wider rule
   - File an issue on the target repo when capability 6 is missing:
     title `deploy: commit a dev override for <flags>`, body listing each
     dev-only flag the daemon refused to boot without and why it belongs
     in the repo, not on the box

   **Human-side** (hand over as exact commands; wait; re-run `check`):

   - MCP server registration with a token **file**, not an inline value:

     ```json
     "containarium": {
       "type": "stdio",
       "command": "<path>/mcp-server",
       "env": {
         "CONTAINARIUM_SERVER_URL": "<control-plane-url>",
         "CONTAINARIUM_JWT_TOKEN_FILE": "~/.containarium/mcp.key",
         "CONTAINARIUM_KEYS_DIR": "~/.containarium/keys"
       }
     }
     ```

     The token file is re-read per request, so rotation is `mv` — no
     restart, and the value never appears in a config the agent reads
   - The build credential: a **fine-grained GitHub token, read-only on
     exactly the private modules** (`Contents: read` on
     `<owner>/<private-module-repo>`, nothing else), put once in the secret
     store of the **daemon that owns the boxes** — `containarium
     secrets set <tenant> GH_TOKEN <value> --server <daemon>` from *their*
     session (not the MCP's `set_secret`: a hosted control plane refuses
     secrets with 501 by design) — so every box that tenant creates
     inherits it as `environment.GH_TOKEN`. The box-side build then reads
     it from env:

     ```bash
     umask 077; printf '%s' "$GH_TOKEN" > "$XDG_RUNTIME_DIR/gh_token"
     podman build --secret id=gh_token,src="$XDG_RUNTIME_DIR/gh_token" ...
     rm -f "$XDG_RUNTIME_DIR/gh_token"
     ```

     The agent never holds the value; it only names the secret
   - Any other backend or app credentials the target daemon needs to be
     *useful* on dev go into the same store the same way. Without them a
     dev deploy may boot but be unable to create boxes — say so in the
     readiness verdict

5. **Per-pass box defaults (mode `box <name>`)**

   Run right after `/devops-deploy` creates a `verify-*` box, before the
   first push:

   - **Linger** — rootless podman dies with the ssh session otherwise.
     Preferred: `compose_enable` (it enables linger as a side effect).
     If the control plane does not support it, the fallback is
     `sudo loginctl enable-linger $USER` **run by the human** — the agent
     hands over the command; remote `sudo` is not its call
   - **Registry search** — `~/.config/containers/registries.conf` with
     `unqualified-search-registries = ["docker.io"]` (user scope, no
     sudo); rootless podman on the ubuntu image ships none
   - **Tooling** — `tmux` for stateful `connect --session`; `git`,
     `curl`, `make` are present on the image, confirm rather than assume
   - **Sizing and name** — `verify-sprint<N>-<sha>`; size to the build (a
     Go + webui build starves below roughly 8 CPU / 8 GB)
   - Verify: `loginctl show-user $USER -p Linger` → `Linger=yes`, then
     start a throwaway container, close the session, reconnect, and see it
     still running. That reconnect *is* the check

6. **Re-check and hand off**

   Re-run `check`; update the readiness comment with the new matrix.
   Verdict `ready` means: a `/devops-deploy` from this runtime, followed
   by `/qa-sprint-verify`, hands nothing back to a human except the two
   things that are legitimately theirs — a prod confirmation and any
   human-in-the-loop verification step (a tester's own sign-in).

**Platform quirks this skill works around (verify against your version;
file or link, don't hide)**

- `compose_enable` may be unavailable on a hosted control plane, so no
  linger without sudo on the box
- `podman-compose run` rejects `--no-build`; run migrations straight from
  the built image on the compose network
- `podman-compose` evaluates `${VAR:?}` guards for profiled-off services
- `/v1/version` reports the last tag, not the build commit — take the pass
  identity from the parity check, not the endpoint
- There is no platform path for a build secret; the tenant store + env
  read above is the workaround until there is one

Each of these should be an issue on the platform repo, referenced from the
readiness comment, so the workaround has an expiry.

**Guardrails**

- Never read, print, or echo a token value — not to "verify" it, not
  into a report, not into a settings file the agent writes. Names and
  file paths only
- Never move a credential between machines. If a check needs one on a
  box, the tenant secret store puts it there; if a check needs one
  locally, the human creates the file
- Never widen permissions to `Bash(*)`, disable permission checks, or
  suggest doing so; rules name the chain's real commands
- Never author a daemon "allow", "stub", or "skip" flag on a box. A
  daemon that refuses to boot without one has a config gap in its repo —
  file it there and stop
- Never `sudo` on anything but a `verify-*` per-pass box, and even there
  hand the command to the human when the harness refuses it
- `prod` profile: read-only. No `create_container`, `set_secret`,
  `expose_port`, or ssh with a write, ever
- Never report `ready` without every row's evidence from this session;
  a matrix copied from a previous run is a previous run's matrix
- This skill provisions the runtime — it does not deploy
  (`/devops-deploy`), verify (`/qa-sprint-verify`), or change the target
  repo beyond filing issues
