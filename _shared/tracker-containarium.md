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
| `issue.create` | `tracker_create_issue` | **must name `parent_number`**; the platform may file it for approval first; labels must pass the connection's allow-list |
| `issue.label` | `tracker_set_labels` (`add_labels`, `remove_labels`) | allow-list applies |
| `issue.comment` | `tracker_comment` | the platform stamps the identity — don't add your own `${who}` line |
| `issue.claim` | `tracker_claim` | **replaces the claim comment**: stamped comment + assign; refused while another run holds the claim; idempotent for the same run |
| `change.create` | `tracker_submit_change` (`issue`, `title`, `description`, `draft`) | ships the box's *committed* workspace and opens the PR/MR through the platform; needs the run's recorded `git_source`. Commit first; uncommitted work is not shipped |
| `change.view` | `tracker_get_change` | state + normalized CI verdict |
| `route.list` | `tracker_route_list` | admin-scoped; lists which agent skill each `scope:<role>` label starts |

Verbs with **no Containarium tool** (releases, CI log fetch, label creation,
milestones): fall back to `gh`/`glab` from outside the box, or hand back.
Don't invent a tool.

## Labels — the allow-list is a hard gate

The connection rejects labels outside its allow-list *before* the tracker is
touched. The list is set per connection. The team's type labels, `runtime:*`,
`sprint`, `verification` and `needs-verification` may not be on it.

- Filing with a rejected label fails the whole create. So: create the issue
  with allowed labels only, then `issue.label` the rest, and **report each
  rejection** — never drop a label silently.
- Fix is on the connection, not the issue: ask the connection's admin to
  allow the labels the cycle needs (see `/devops-containarium-env`
  capability 9). Don't work around it by encoding the type in the title.

## Native automation: `scope:<role>` routes

On Containarium a `scope:<role>` label can *start an agent skill* on the
issue. Routes are set by an admin. Never add `scope:*` to an issue you were
not asked to dispatch.

### Platform-provided skills

The platform may ship its own skills for some roles. `list_agent_skills`
shows what your deployment runs. A route may start a platform skill instead
of ours. That skill follows its own prompt, so our contract (claim wording,
labels, identity rules, evidence) is only as enforced as that prompt. Prefer
our skill when the contract matters. Say which one ran in the claim.

## Open follow-ups are approval-gated

Follow-ups filed by a run carry `agent:needs-approval`. Treat that as the
platform's human gate — never remove it yourself.
