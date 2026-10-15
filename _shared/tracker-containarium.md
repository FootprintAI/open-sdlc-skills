# Tracker mapping — Containarium (third backend)

Extends `_shared/tracker.md` (GitHub ↔ GitLab) with the **Containarium
tracker tools**, for work running on Containarium — typically inside a box,
as a run bound to a tracker *connection*. Same neutral verbs; read
`tracker.md` first for the vocabulary and the credential rules.

The Containarium column is **opt-in**. Use it only when the run is bound to
a connection (the tools need `username` + `connection`). Otherwise keep
using `gh`/`glab`. Tool names below are from the Containarium MCP
server; descriptions say each mirrors `containarium tracker …` on the CLI.

## Resolve — before the first call

1. A run started by Containarium (a routed skill, `run_agent_skill`) already
   knows its connection and tenant `username`; take both from the run's
   environment/prompt, never guess.
2. State the backend once in the first report line:
   `tracker: containarium (connection <name>)`.
3. No connection ⇒ not this backend. Do not create one.

## Verbs

| Verb | Containarium tool | Differs from `gh`/`glab` |
| --- | --- | --- |
| `issue.list` | `tracker_list_issues` (`state`, `labels` AND, `search`) | no comments in the list — use `issue.view` |
| `issue.view` | `tracker_get_issue` (includes comments) | |
| `issue.create` | `tracker_create_issue` | **must name `parent_number`**; filed `agent:needs-approval` unless the connection auto-chains; identity stamp + parent link appended server-side; bounded by depth and fan-out caps; labels pass the connection **allow-list** |
| `issue.label` | `tracker_set_labels` (`add_labels`, `remove_labels`) | allow-list applies |
| `issue.comment` | `tracker_comment` | body sanitized server-side; identity-stamped — don't add your own `${who}` line, the platform stamps it |
| `issue.claim` | `tracker_claim` | **replaces the claim comment**: stamped comment + assign, refused when a live or recent claim from another run holds it; idempotent for the same run; a claim from an unconfirmed run is takeable after `stale_after_seconds` (default 2 h) |
| `change.create` | `tracker_submit_change` (`issue`, `title`, `description`, `draft`) | bundles the box's *committed* workspace out, pushes from a temporary repo **on the host** to a daemon-chosen branch, opens the PR/MR — **no push credential enters the box**; needs the run's recorded `git_source`. Commit first; uncommitted work is not shipped |
| `change.view` | `tracker_get_change` | state + normalized CI verdict |
| `route.list` | `tracker_route_list` | needs `tracker:admin`; shows which agent skill each `scope:<role>` label starts |

Verbs with **no Containarium tool** (releases, CI log fetch, label creation,
milestones): fall back to `gh`/`glab` from outside the box, or hand back.
Don't invent a tool.

## Labels — the allow-list is a hard gate

The connection rejects labels outside its allow-list *before* the tracker is
touched. The default list is `scope:*`, `model:*`, `agent:needs-approval`.
The team's other labels (the type labels, `runtime:*`, `sprint`,
`needs-verification`) are **not** in it by default.

- Filing with a rejected label fails the whole create. So: create the issue
  with allowed labels only, then `issue.label` the rest, and **report each
  rejection** — never drop a label silently.
- Fix is on the connection, not the issue: ask the connection's admin to
  add `bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`,
  `runtime:*` (see `/devops-containarium-env` capability 9). Don't work
  around it by encoding the type in the title.

## Native automation: `scope:<role>` routes

On Containarium a `scope:<role>` label can *start an agent skill* on the
issue (see `route.list`). So the cycle's routing labels can trigger work
directly instead of a CLI launch — `model:*` picks the tier (allowed by
default). Only an admin sets routes (CLI-only); never add `scope:*` to an
issue you were not asked to dispatch.

### Platform skills that overlap ours

The platform's built-in catalog ships a few skills with the same shape as
ours — `issue-implementer` (routable by label), `product-define` (routable by
label), and `code-review` (via `run_agent_skill`). `list_agent_skills` shows
the full set on your deployment.

Where one exists, a `scope:<role>` route may be used **instead of** our
launcher — but it runs the platform's skill, not ours, so our contract
(claim wording, labels, identity rules, evidence) is only as enforced as the
platform skill's prompt. Prefer our skill when the contract matters; prefer
the route when unattended dispatch matters. Say which ran in the claim.
Platform skills run through the platform **model-gateway** (no model key in
the box).

## Open follow-ups are approval-gated

Follow-ups filed by a run carry `agent:needs-approval`. Treat that as the
platform's human gate — never remove it yourself.
