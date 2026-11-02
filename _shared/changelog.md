# Changelog fragments

Shared reference for `/engineer-implement`, `/release-cut` and
`/merge-steward`. It exists because of one failure: when every PR edits
the top of `## [Unreleased]` in one `CHANGELOG.md`, every pair of open PRs
conflicts on the same lines, and each merge makes the rest of the queue
stale.

## Detect which convention the repo uses

- A `changelog.d/` (or `changes/`, `newsfragments/`) directory, or a
  towncrier / changie config → **fragments**. Follow this file.
- No such directory → **single file**. Edit `CHANGELOG.md` under
  `## [Unreleased]` as the repo's PR template says, and do not introduce
  fragments on your own: switching convention is a repo decision, raised
  as an issue, not slipped into a feature PR.

## Fragments

One file per PR, named for the change, never for a shared line:

```
changelog.d/<pr-or-issue-number>.<category>.md
```

`<category>` is the repo's own heading, lower-cased: `added`, `changed`,
`fixed`, `security`, `upgrading`, `docs`, `internal`. The body is exactly
the entry that would have gone in the changelog, written for a user of
the product, with the PR or issue reference in it.

Two PRs never touch the same file, so they cannot conflict. A PR with no
user-visible change adds no fragment and says so (a `no-changelog` label
or a line in the PR body), so a reviewer can tell "forgot" from "none".

## Assembling at release

The release step, not a feature PR, builds the section:

1. Read every fragment, group by category, order by number.
2. Write them under the new `## [x.y.z] - <date>` heading, together with
   anything still under `## [Unreleased]`.
3. Delete the fragments in the same commit.

The result is a release-only change (`CHANGELOG.md`, the version
constant, deleted fragments) and nothing else. If the repo has a gate
that allows a release PR to touch only the changelog and the version,
it must also allow deleted fragment files, or the release PR is refused.

## Never

- Rewrite or reorder existing released sections to "tidy" them.
- Assemble fragments in a feature PR — that brings back the conflict.
- Invent an entry for a PR with no fragment; release notes cite what
  merged (`/release-cut`).
