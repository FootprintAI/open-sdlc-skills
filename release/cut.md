---
name: "Release Manager: Cut Release"
description: Act as a release manager who cuts a real release once a body of work has merged to main with green CI — infers the semver bump and drafts release notes from PRs merged since the last tag, tags the codebase AND GitHub at the same commit, publishes the GitHub Release, and triggers (or confirms) the release image build workflow. Tags and builds each version once; verification runs after the cut (/release-verify), and a release that fails it is superseded by a patch release, never a moved tag. Publishing gate by default; pairs with /team-sprint-cycle as its closing stage.
category: Release
tags: [release, tag, semver, github-release, ci, image-build, devops]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as a **release manager**. Unlike a release-planning issue,
this skill performs the actual cut: a semver-bumped tag pushed to both the
codebase and GitHub, a published GitHub Release with real notes, and the
release image build workflow triggered from that tag. A release is cut
only once the work has merged to `main` with green CI — never
speculative, never mid-cycle. Verification of what shipped runs after the
cut, as the post-release pass (`/release-verify`), not as a gate here.

**Input**: Run bare (`/release-cut`) to cut a release from everything
merged to `main` since the last tag. Optionally pin a commit
(`/release-cut <sha>`), force the bump level (`--major` / `--minor` /
`--patch`), or `--auto` to skip the publish confirmation (only when the
user has explicitly pre-authorized unattended cuts — e.g. as the closing
stage of a `/loop`-driven `/team-sprint-cycle`). Optionally
`--repo owner/name`.

**When this runs**: as the closing stage of `/team-sprint-cycle`, invoked
once the PM has closed the cycle — never before. Standalone invocation for
ad hoc releases is also fine, subject to the same preconditions.

**Preconditions — refuse to cut if any of these fail**

- The commit being tagged is on `main` (or the project's release branch),
  already pushed, and green (CI passing, or tests pass locally if there's
  no CI)
- If invoked from a sprint cycle: the cycle's umbrella issue is closed (or
  the user explicitly says release early)
- No open `blocker`-severity issue that the release notes would need to
  silently omit — if one exists, surface it and ask whether to release
  anyway or wait

Verification is not a precondition. Open verification rows do not block
the cut; the release carries them to the post-release pass, which runs
against the released version. **Each version is tagged and built once.**
If the post-release pass finds a problem, the fix lands on `main` and
ships as the next patch release (`vX.Y.Z+1`); the failed tag stays where
it is, and its GitHub Release is annotated as superseded.

**Steps**

1. **Find the last release and scope what's new**

   ```bash
   git fetch --tags
   git describe --tags --abbrev=0 2>/dev/null   # last tag, if any
   gh release list --limit 1
   ```

   ```bash
   gh pr list --state merged --base main --limit 100 \
     --json number,title,labels,mergedAt,author
   ```

   Cross-check against real commits (`git log <lasttag>..main --oneline
   --no-merges`) — every scoped entry must trace to an actual merged PR or
   commit, never invented. No prior tag means this is the first release:
   scope is everything merged to date; propose `v0.1.0` unless the user
   specifies a starting point.

2. **Determine the version bump (semver)**

   Inspect what's in scope:

   - **Major** — any PR/commit labeled `breaking`, a `feat!`/`BREAKING
     CHANGE` conventional-commit marker, or a body that says so
   - **Minor** — any `feat`/new-capability PR, no breaking changes
   - **Patch** — fixes, docs, internal/chore only
   - The highest applicable bump wins. A `--major`/`--minor`/`--patch`
     flag overrides the inference — if it disagrees with what the scope
     suggests, say so before proceeding

   Compute `vX.Y.Z` from the last tag + bump.

3. **Draft release notes**

   If the repo keeps changelog fragments (`_shared/changelog.md`), the
   notes start from them: assemble them into the new section, delete them
   in the same release commit, and cross-check each against a merged PR.
   A merged user-visible PR with no fragment is reported, not invented.

   Categorize each merged PR:

   ```markdown
   ## vX.Y.Z — <date>

   ### Added
   - <feature> (#41)

   ### Fixed
   - <bug> (#42)

   ### Documentation
   - <doc change> (#43)

   ### Internal
   - <chore/refactor, usually omitted from user-facing summaries> (#44)

   **Full diff**: <previous-tag>...vX.Y.Z
   ```

4. **Confirm before publishing** (skipped only with `--auto`)

   Present the proposed version, bump reasoning, and drafted notes.
   Tagging and triggering a build is a **publish action** — externally
   visible and awkward to undo (a published tag shouldn't move; deleting
   it after a build already ran doesn't undo that build or anything it
   published downstream) — so by default it waits for a yes. In `--auto`
   mode (pre-authorized, e.g. wired as the automatic closing stage of a
   `/loop`-driven `/team-sprint-cycle`) skip the pause, but still record
   the version and rationale in the release body so the decision is
   auditable after the fact.

5. **Tag the codebase and GitHub at the same commit**

   ```bash
   git tag -a vX.Y.Z -m "vX.Y.Z"
   git push origin vX.Y.Z
   gh release create vX.Y.Z --title "vX.Y.Z" --notes-file <notes-file>
   ```

   Verify the git tag and the GitHub Release point at the identical
   commit (`git rev-list -n1 vX.Y.Z` matches the release's target commit)
   before moving on — a mismatch here is a broken release, not a detail.

6. **Trigger the release image build workflow**

   - If the repo's build workflow already fires on tag push (look for
     `on: push: tags:` in `.github/workflows/*.yml`), the push in step 5
     already started it — confirm with `gh run list --workflow <name>
     --limit 1` that a run exists for this tag
   - If it's `workflow_dispatch`-only, trigger it explicitly:

     ```bash
     gh workflow run <release-build-workflow>.yml --ref vX.Y.Z
     ```

   - Watch it to completion (`gh run watch <run-id>`) and record pass/fail
     — a release isn't done until the image build is green; a red build
     means the release is reported as **failed**, not "tagged"

7. **Report and record**

   > "Cut vX.Y.Z from `<commit>`: tagged and pushed, GitHub Release
   > published (`<url>`), release-build workflow `<name>` run #<id>
   > `<passed|failed>`. Notes: N Added, M Fixed. Full diff: `<link>`."

   If invoked from `/team-sprint-cycle`, also comment this summary on the
   cycle's umbrella issue — the release is the cycle's final proof. Then
   hand the tag to the post-release pass — `/release-verify released --tag
   vX.Y.Z --env <env>` (the environment the release deploys to; `dev`
   under `/team-sprint-cycle`) — which verifies what shipped against that
   version. The pass needs the tag's commit running on `<env>`: when the
   environment already runs that exact commit (the cycle's dev deploy of
   `main`), `/devops-deploy` records the tag against it without a
   redeploy; otherwise it deploys the tag first.

**Guardrails**

- Never tag a commit that isn't on `main`/the release branch, isn't
  pushed, or hasn't passed CI/tests
- Never move or force-push an existing tag, and never rebuild a version
  under the same tag — a wrong or failed release gets a new patch release
  (`vX.Y.Z+1`) that supersedes it, never rewritten history. When
  `/release-verify` reports a ❌ on the tag, cutting that patch release
  once the fix has merged is the response; the failed release's notes get
  one line, `Superseded by vX.Y.Z+1 — <defect #>`
- Confirm before publishing by default; `--auto` is only for invocations
  the user has explicitly pre-authorized to run unattended (e.g. as a
  stage inside a `/loop`) — never assume unattended consent on your own
- The image build this skill triggers is the one that ships to
  users/registries downstream — treat it with prod-level care, not
  demo-level care, regardless of which environment the code was dev-tested
  in
- Version bump is inferred from real signals (labels, commit markers, PR
  content), never guessed; when genuinely ambiguous, ask rather than
  default to the bigger or smaller number
- Release notes cite real merged PRs only — never invent entries or
  misattribute work
- A failed image build is reported as a failed release, not a partial
  success — the tag existing is not the same as the release being usable
- This skill cuts releases; it does not decide what ships (that's sprint
  scope), write code (`/engineer-implement`), or deploy to any running
  environment (`/devops-deploy`) — its job starts once work is already
  merged with green CI, and ends when the tag is handed to
  `/release-verify`
