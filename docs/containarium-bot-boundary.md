# Containarium bot — what goes where

Status: **draft**. `Containarium-cloud` is private, so the split below holds.

The bot is `@containarium <verb> <target>` — for example `start #123` or
`review #45`. It is wired in `Containarium-cloud`. The skills it runs live in
this repo. This doc sets the line between the two.

## 1. This repo (public, Apache 2.0)

Allowed:

- Skill contracts: what a role does, its inputs, its outputs, its evidence
  gates. Generic process.
- The user-facing command grammar (verb names, what each verb takes).
- Adapter docs that call **Containarium MCP tool names** as a backend, such
  as `_shared/tracker-containarium.md`. Tool names are already public via the
  MCP server.
- Synthetic examples: `acme-demo`, `example.com`, placeholder IDs.

Not allowed:

- Any real or working credential, token, key path, or secret name.
- Hostnames, IPs, daemon addresses, tenant or user names, or customer names.
- How the hosted control plane authorizes, caps, times out, or rejects a
  request. That is the map section 2 keeps private.
- Roadmap, pricing, competitor notes, or product strategy from real planning
  (PRDs, sprint plans, `product-define` output for real work).
- Output of a real run that contains any of the above.

## 2. Containarium-cloud (private)

Owns:

- Dispatch: which verb starts which skill, and under which model tier.
- Who may trigger a verb (collaborator check, write access, bot identity).
- Server-side enforcement: label allow-lists, depth and fan-out caps, claim
  takeover rules, sanitization of comment bodies, identity stamping.
- Rate limits, timeouts, and budget caps.
- Secret storage and the rule that the hosted plane never holds tenant
  plaintext.

Enforcement must hold even if someone has read these rules. Keeping them
private is defense in depth, not the defense itself.

## 3. Cleanup before the bot ships

Review these against the rules above. Do not move them without a decision:

- `_shared/tracker-containarium.md` — the "Resolve", "Verbs", and "Labels"
  sections describe enforcement details (allow-list gate, depth and fan-out
  caps, the 2-hour stale-claim window, the 501 on hosted secrets). Candidate
  to trim to tool names and the caller's contract; the enforcement detail
  moves to the cloud repo.
- `devops/containarium-env.md` — the secrets section names where tenant
  tokens live (`containarium secrets list <tenant> --server <daemon>`). Keep
  the check procedure; drop anything that reads like a real layout.

## 4. Rules for anyone writing to either repo

- Write the contract, not the mechanism. "Only collaborators may start a
  run" is fine here. The check itself belongs in the cloud repo.
- Before committing, scan the diff for hostnames, IPs, tokens, tenant names
  and customer names. Use synthetic values.
- If a doc describes a cloud behavior, mark which repo owns it.
- When unsure whether a detail is safe to publish, leave it out and ask.

## 5. When the environment cannot use Containarium

Containarium is optional for the planning, build, review and QA stages. The
skills only need `gh` or `glab`. Use the fallback below when a runtime has no
Containarium MCP server, no tracker connection, or no daemon.

**Detection.** The runtime has no Containarium if `/devops-containarium-env`
check reports capability 2 (control-plane MCP) or capability 3 (CLI) as
missing, or if the run has no tracker connection (`tracker_*` tools are not
bound). Do not create a connection to get around this.

**What still runs, and how**

| Stage | Skill | Without Containarium |
|-------|-------|----------------------|
| Implement | `/engineer-implement` | Local or CI run; tracker via `gh`/`glab`; claim comment posted by the run |
| Review | `/reviewer-pr` | Verdict posted on the PR with `gh`/`glab` |
| Verify | `/qa-verify` | Runs in a sandbox with no credentials; report posted on the PR |
| Merge | `/merge-steward` | Scheduled run (`/loop`) or CI job; same merge rules |
| Test | `/qa-unit-test`, `/qa-integration-test`, `/qa-e2e-test` | CI lanes or a local container runtime |
| Release | `/release-cut` | Tag and GitHub release via `gh`/`glab` |
| Deploy | `/devops-deploy` | **No fallback yet.** Needs a decision on the target (see below) |

**Triggering without the bot.** Two options, in order of preference:

1. **Run by hand or from CI.** Invoke the skill directly. No trigger layer is
   needed, and the output is the same.
2. **A GitHub Action that calls the same skills.** A comment such as
   `/engineer start #123` or `/reviewer review #45` starts a job that runs the
   matching skill. The job must use the same contracts and the same
   permission rules as the bot: only collaborators with write access may
   trigger it, and it never runs untrusted PR code with secrets. The
   Action's own config lives in the public repo; the rules it enforces must
   match section 2.

**Open decision: deploy without Containarium.** `/devops-deploy` assumes
Containarium. Before a fallback exists, pick the target: a plain SSH host, a
managed platform, or a container registry plus orchestrator. Until then,
deploys on a non-Containarium runtime are manual and must be recorded on the
sprint umbrella the same way `/devops-deploy` records them.
