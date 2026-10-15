# Notice: every routed issue names its runtime

**Applies to `/team-sprint-cycle` and `/engineer-implement`** — and anything
else that launches an agent to do an issue's work. The *runtime* is the agent
CLI that does the work; the *model* is the tier inside it. They are routed
independently.

## Labels

| Label | Meaning |
|---|---|
| `runtime:claude` | work runs under the Claude Code CLI (**default**) |
| `runtime:codex` | work runs under the Codex CLI |

Model tier labels (`model:sonnet|opus|fable`) apply to `runtime:claude`. For
`runtime:codex` the model is whatever the Codex CLI is configured with, or an
explicit `model:<name>` label if one is given.

## Rules

- Every issue the cycle implements carries exactly one `runtime:*` label,
  set when it is scoped and stated with a one-line reason next to the
  `model:*` rationale. No label ⇒ `runtime:claude`, and the scoping comment
  says so.
- **Never fall back silently.** If the labelled runtime's CLI is not on
  PATH or not authenticated (`command -v claude` / `command -v codex`), park
  the issue as an open question on the umbrella ("`runtime:codex` requested,
  CLI unavailable — run on claude instead? default: wait"). Do not swap.
- Escalation (stall → stronger model) stays inside the runtime. Switching
  runtime on a stall is a deliberate re-route: remove the old `runtime:*`,
  add the new, comment why.
- On a connection-bound Containarium run the claim is `tracker_claim`
  (platform-stamped identity); the runtime still goes in the claim's
  accompanying comment.
- **Box id on the issue.** While an agent works an issue, the issue says
  where: the claim comment carries a `Box:` line, and the same line is
  reposted if the box changes.

  ```
  agent-7f3a1c-codex is starting processing it
  Box: issue-41-myrepo
  ```

  - Local worktree run: `Box: none` — still stated, so "no box" is a fact
    and not an omission
  - The value is the **box id only**, taken from the `create_container`
    result, never guessed. Nothing else goes on the line — no run name,
    backend, **IP, ssh host, key path, or token** (credential rules apply
    to comments too)
  - On teardown, post `Box: issue-41-myrepo deleted · PR <url>` *after* the
    evidence is posted, so the issue always shows the box's last state
  - `/scrum-master` uses it: a claim whose `Box:` is missing, or whose box has no
    live run (`code_runs` on that box id), is a stalled claim to escalate —
    evidence, not label
- The claim comment uses the runtime too: `${who}-${runtime}-${model}`
  (e.g. `agent-7f3a1c-codex`, `agent-7f3a1c-claude-opus`), so the issue shows
  who is working it and on what.
- Same contract either way: claim first, test-first, one worktree = one
  branch = one PR, `Closes #<N>`, tracker-agnostic. Only the launcher differs.

## Launchers

```bash
git worktree add ../<repo>-issue-<N> -b feat/<N>-<slug> main
cd ../<repo>-issue-<N>

# runtime:claude
claude --model <sonnet|opus|fable> -p "/engineer-implement #<N>"

# runtime:codex — no slash skills; point it at the same contract
codex exec "Implement issue #<N> following the contract in \
~/.claude/skills/engineer-implement/SKILL.md (also read _shared/labels.md, \
_shared/runtime.md, _shared/estimates.md)."
```

> Verify the `codex exec` flags against the installed Codex CLI version
> before first use — they are the part most likely to drift. When the
> Agent tool spawns the work instead of the CLI, `runtime:codex` has no
> equivalent: use the CLI launcher.

## Launcher: in a Containarium box (preferred when on Containarium)

Instead of a local worktree, run the issue in its own box. The agent keeps
running if the session drops, the box is disposable, and no credential has
to be placed in it.

1. **Box** — `create_container` named `issue-<N>-<repo>` (sizing as
   `/devops-containarium-env` capability 8). Verify the runtime's CLI is on
   it before dispatching; the box needs the coding toolchain installed
   (`code_run` requires it). If the labelled runtime isn't installed and
   can't be, park the issue — no silent swap (rules above).
2. **Claim** — `tracker_claim` (see `_shared/tracker-containarium.md`), or
   the claim comment on `gh`/`glab` repos. Name the runtime and the `Box:`
   line in it (rules above).
3. **Dispatch** — `code_run` with `box` + `name: issue-<N>` and a prompt
   that points at the engineer-implement contract and the shared notices
   (labels, runtime, estimates). A distinct `name` lets several issues share
   one box. Follow with `code_attach` (carry `next_offset` forward — lossless
   reconnect) or `code_status` for the exit code; `code_stop` to abort.
4. **Deliver** — commit inside the box, then `tracker_submit_change`
   (`issue: <N>`): the PR opens from the host, `Closes #<N>` in the
   description, no push token ever in the box. `tracker_get_change` for the
   CI verdict.
5. **Tear down** — `delete_container` only after the PR URL and the run log
   (`code_logs` to the end) are posted on the issue. Evidence first, then
   delete.

> **Note:** the model-gateway's provider list depends on the deployment and
> may not include Codex's provider — so `runtime:codex` in a box can mean
> installing the Codex CLI on the box (toolchain), not a gateway option.
> Check `list_agent_engines` / the gateway's model list on your deployment.
>
> **Unverified:** `code_run`'s own parameters take no runtime selector, so
> which agent it starts is decided by what is installed on the box. Confirm
> that `claude` and `codex` can each be installed and selected on a box
> before relying on `runtime:codex` here; until confirmed, `runtime:codex`
> uses the local launcher above.

## Choosing a runtime

Default to `runtime:claude`. Choose `runtime:codex` when the issue's author
or the team has a reason (a task shape it handles better, a second opinion,
quota). Record that reason in the scoping comment, and note outcomes in the
retro so the routing improves from evidence rather than habit.
