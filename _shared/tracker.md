# Tracker mapping — GitHub ↔ GitLab

Shared reference for every team skill that reads or writes issues, change
requests, CI, labels, or releases. Skills speak the **neutral verbs** below;
this file says what each verb is on the tracker the repo actually uses.

Scope of the GitLab column: **self-managed GitLab, Free tier**. Anything that
needs Premium is listed under "Not available" with the Free-tier substitute —
don't reach for it.

If a `glab` flag here disagrees with `glab <command> --help` on the installed
version, the installed help wins; fix this file afterwards.

## 1. Resolve the tracker — once, before the first tracker call

```bash
host=$(git remote get-url origin | sed -E 's#^([a-z+]+://)?([^@/]+@)?([^/:]+).*#\3#')
```

| `host` | Provider | CLI |
| --- | --- | --- |
| `github.com` | GitHub | `gh` |
| anything else, and `glab auth status --hostname "$host"` succeeds | GitLab | `glab` |
| anything else, and `gh auth status --hostname "$host"` succeeds | GitHub Enterprise | `gh` |
| none of the above | **unknown — stop and report** | — |

Decide from the auth probe, never from the hostname's spelling (a host named
`git.<company>` can be either). State the resolved provider once in your
first report line so a wrong guess is visible.

**Credentials.** Use whatever the environment already provides (`gh auth`,
`glab auth`, `GITLAB_TOKEN` + `GITLAB_HOST`). If no credential is available,
stop and say so. Never ask for a token to be pasted into a sandbox/box, never
write one into a repo, a seed file, a box image, `.git/config`, or a comment,
and never echo one into a log.

## 2. Vocabulary

| Concept | GitHub | GitLab |
| --- | --- | --- |
| Change request | pull request, `#M` | merge request, **`!M`** |
| Issue reference | `#N` | `#N` (other project: `group/project#N`) |
| Issue number | `number` | **`iid`** (project-scoped; `id` is the global one — never use it in a URL or reference) |
| CI | Actions workflow / run | pipeline / job |
| Auto-close keyword | `Closes #N` in the PR body | `Closes #N` in the MR description — fires only when merged into the **default branch** |
| Sprint container | umbrella issue with a `- [ ] #N` task list | the same umbrella issue; optionally also a **milestone** for a free progress bar |

Write `!M` when linking a merge request in a comment — `#M` would link an
unrelated issue with the same number.

## 3. Verbs

`N` = issue number, `M` = change-request number.

### Issues

| Verb | GitHub | GitLab |
| --- | --- | --- |
| `issue.list` | `gh issue list --state open --label L --json number,title,labels,assignees,updatedAt` | `glab issue list --label L --output json` (add `--closed` / `--all` for other states) |
| `issue.search` | `gh issue list --state all --search "<kw>" --json number,title,state,labels` | `glab issue list --all --search "<kw>" --output json` |
| `issue.view` (with comments) | `gh issue view N --json number,title,state,body,assignees,labels,updatedAt,comments` | `glab issue view N --comments`; for parseable evidence: `glab api "projects/:id/issues/N"` and `glab api "projects/:id/issues/N/notes?sort=asc&per_page=100"` |
| `issue.comment` | `gh issue comment N --body "<text>"` | `glab issue note N -m "<text>"` |
| `issue.create` | `gh issue create --title T --body B --label L` | `glab issue create -t T -d B -l L --yes` |
| `issue.assign-self` | `gh issue edit N --add-assignee @me` | a note whose own line is `/assign me` (see §4) |
| `issue.label` | `gh issue edit N --add-label A --remove-label B` | `glab issue update N --label A --unlabel B` |
| `issue.edit-body` | `gh issue edit N --body-file F` | `glab issue update N -d "$(cat F)"` |
| `issue.close` | `gh issue close N` | `glab issue close N` |
| `issue.linked-changes` | `gh pr list --state all --search "N" --json number,title,isDraft,updatedAt,mergedAt` (text search — verify each hit really references N) | `glab api "projects/:id/issues/N/related_merge_requests"` and `.../issues/N/closed_by` (exact, no verification needed) |

### Labels

| Verb | GitHub | GitLab |
| --- | --- | --- |
| `label.list` | `gh label list` | `glab label list` |
| `label.ensure` | `gh label create NAME -c "#hex" -d "<desc>" 2>/dev/null` | `glab label create --name NAME --color "#hex" --description "<desc>" 2>/dev/null` |

Label names carry over unchanged, including `model:sonnet` / `model:opus` /
`model:fable` — a single colon is a plain label on GitLab.

### Change requests

| Verb | GitHub | GitLab |
| --- | --- | --- |
| `change.create` | `gh pr create --title T --body B [--draft]` | `glab mr create --title T --description B --target-branch main --remove-source-branch --yes [--draft]` |
| `change.view` | `gh pr view M --json title,body,files,additions,deletions` | `glab mr view M --output json` |
| `change.diff` | `gh pr diff M` | `glab mr diff M` |
| `change.list` | `gh pr list --state all --limit 30` | `glab mr list --all --output json` |
| `change.comment` | `gh pr comment M --body "<text>"` | `glab mr note M -m "<text>"` |
| `change.review` | `gh pr review M --approve \| --request-changes --body "<text>"` | approve: `glab mr approve M`; request changes: `glab mr note M -m "<text>"` opening with `Verdict: changes requested`, and `glab mr revoke M` if previously approved |
| `change.inline-comment` | `gh api` review comments | `glab api --method POST "projects/:id/merge_requests/M/discussions"` with a `position[...]` payload; if the position is fiddly, quote `path:line` in a normal note instead |

### CI

| Verb | GitHub | GitLab |
| --- | --- | --- |
| `ci.baseline` (is the default branch green?) | `gh run list --branch main --limit 1` | `glab api "projects/:id/pipelines?ref=main&per_page=1"` → `.[0].status` |
| `ci.watch` (gate a change request) | `gh pr checks M --watch` or `gh run watch <run-id>` | `glab ci status --live --branch <branch>`; the verdict of record is `head_pipeline.status` from `glab mr view M --output json` |
| `ci.logs` | `gh run view <run-id> --log-failed` | `glab ci trace <job-id>` |
| `ci.trigger` | `gh workflow run <wf> --ref <ref>` | `glab ci run --branch <ref> [--variables K:V]` |

Green means `status == "success"`. `manual`, `skipped`, and `canceled` are
**not** green; an MR with no pipeline at all is "no CI" — a finding, per the
engineer skill's CI rule.

### Releases

| Verb | GitHub | GitLab |
| --- | --- | --- |
| `release.list` | `gh release list` | `glab release list` |
| `release.create` | `gh release create TAG --title T --notes-file F` | `glab release create TAG --name T --notes-file F` |

### Evidence attachments (screenshots, logs)

| GitHub | GitLab |
| --- | --- |
| No upload API for issue attachments — commit the file under the repo's evidence path and link it | `curl -sf -H "PRIVATE-TOKEN: $GITLAB_TOKEN" -F "file=@shot.png" "https://$host/api/v4/projects/<url-encoded group%2Fproject>/uploads"` → paste the returned `.markdown` into the issue or note |

## 4. GitLab quick actions — status changes inside a comment

A note can carry state changes on their own lines. The text is posted and the
actions are applied in the same call, so a claim is one write, not three:

```
agent-1a2b3c4d-opus is starting processing it
/assign me
/label ~"model:opus"
```

Rules:

- Each quick action sits alone on its line; label names with `:` or spaces
  need the `~"..."` form.
- **`/assign me` only when the issue is unassigned.** Free tier allows one
  assignee per issue, so assigning *replaces* whoever is there — check
  `assignees` first, and leave the line out if someone already holds it.
- A note containing *only* quick actions posts no visible comment. Status
  that people need to see always has a text line.
- Tier escalation is a swap, not an add — plain labels are not exclusive:
  `/unlabel ~"model:opus"` then `/label ~"model:fable"`.
- `/relate #12` records a "relates to" link (Free). It does not express
  direction, so keep the `Blocked by #12` line in the issue body as well.

## 5. Not available on self-managed Free — and what to do instead

| Premium feature | Free-tier substitute |
| --- | --- |
| Scoped labels (`model::opus`, mutually exclusive) | plain `model:opus`; remove the old tier label explicitly when escalating |
| "Blocks / is blocked by" issue links | `Blocked by #N` text in the body + `/relate #N` |
| Epics | the umbrella issue; optionally a milestone per sprint |
| Multiple assignees | one assignee = the live claim holder; everyone else is named in comments |
| Required approvals / merge blocked by a "changes requested" review | approval is advisory — the reviewer's verdict note is the record, and nobody merges over an unresolved `Verdict: changes requested` |
| Multiple issue boards, iterations | one board with label lists; milestones for time-boxing |

Task-list checkboxes (`- [ ] #12`) render with the issue's state but do
**not** tick themselves on close, on either tracker. Whoever owns the
umbrella body (`/pm:sprint-delivery update`) ticks them from evidence.

## 6. Reading evidence — field names differ

Skills that judge state from JSON (`/scrum-master`, `/coordinator:status`)
need these translations:

| Meaning | GitHub (`--json`) | GitLab (API / `--output json`) |
| --- | --- | --- |
| Issue number | `number` | `iid` |
| Last activity | `updatedAt` | `updated_at` |
| Open / closed | `state`: `OPEN` / `CLOSED` | `state`: `opened` / `closed` |
| Assignee | `assignees[].login` | `assignees[].username` |
| Comment author / time / text | `comments[].author.login` / `.createdAt` / `.body` | notes: `[].author.username` / `.created_at` / `.body`; skip entries with `system: true` when looking for human or agent comments |
| Draft change request | `isDraft` | `draft` |
| Merged at | `mergedAt` | `merged_at` |
| CI verdict | `statusCheckRollup` | `head_pipeline.status` |
| Review verdict | `reviewDecision` | `glab api "projects/:id/merge_requests/M/approvals"` → `approved_by[]`, plus the latest `Verdict:` note |

System notes (`system: true`) are useful as timestamps — "assigned to",
"added label", "mentioned in merge request !M" — and are harder to fake than
a status label.

## 7. Identity

`${who}-${model}` in a claim comment stays the way agents are told apart on
**both** trackers: concurrent agents normally share one tracker credential,
and the account name therefore identifies nobody. Never substitute the
authenticated username for `${who}`.
