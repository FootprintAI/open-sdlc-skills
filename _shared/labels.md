# Notice: every issue carries one type label

**Applies to every skill that opens an issue** (GitHub or GitLab) — product
stories, sprint umbrellas and child issues, QA defects, release candidates,
docs and CI work. The label says at a glance what an issue *is*.

## Type labels (exactly one per issue)

| Label | Use for | Typical author |
|---|---|---|
| `bug` | a defect — behavior differs from what was specified | `/qa-issue-report` |
| `feature-request` | new or changed user-visible capability | `/product-define`, `/pm-sprint-delivery` |
| `epic` | an umbrella that groups child issues (sprint umbrella, release candidate) | `/pm-sprint-delivery`, `/release-cut` |
| `ci` | pipeline, build, test-infra, release-workflow work | any |
| `docs` | documentation only | any |
| `security` | vulnerability, hardening, compliance gap | any |
| `chore` | maintenance that fits none of the above (deps, refactor, cleanup) | any |

Existing workflow labels (`sprint`, `release`, `product`, `needs-verification`,
`model:*`, `runtime:*`, `size:*`) are **additional**, not replacements — an umbrella is
`epic` + `sprint`; a release candidate is `epic` + `release`.

## Rules

- Pick the type at creation time; never file an issue without one. If it is
  genuinely ambiguous, use the closest and say why in the body — don't skip.
- Create a missing label rather than dropping it (idempotent):
  `gh label create <name> -c "<hex>" -d "<desc>" 2>/dev/null` /
  `glab label create`. Suggested colors: bug `d73a4a`, feature-request
  `a2eeef`, epic `3e4b9e`, ci `fbca04`, docs `0075ca`, security `b60205`,
  chore `cfd3d7`.
- A repo's own labels win when they already cover a type (e.g. `enhancement`
  for `feature-request`): reuse them, don't duplicate.
- Re-type with `issue.label` (add the new, remove the old) when triage
  changes the answer — never leave two type labels.
- **Labels are public on a public repo** — and so are label descriptions,
  issue titles and comments. Never encode a customer, tenant, deployment,
  host, internal project or repo name, an IP, or anything credential-like in
  a label name or description. Use the generic vocabulary above; if a label
  you need would name something private, describe it generically (`blocked`,
  `needs-decision`) and keep the specifics in a private tracker. When
  unsure whether a repo is public, treat it as public.

## On Containarium trackers

A Containarium tracker connection rejects labels outside its allow-list
(default `scope:*`, `model:*`, `agent:needs-approval`), so the type labels
above must be added to the connection first. File with allowed labels, then
label, and report any rejection — see `_shared/tracker-containarium.md` and
`/devops-containarium-env` capability 9.
