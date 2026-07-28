---
name: "QA: Issue Report"
description: File every defect found while running as QA as a GitHub issue — one issue per finding, created before any fix or discussion happens elsewhere. UI findings are verified live in a browser via MCP before filing, with a fresh defect screenshot and the app's version/commit recorded in every issue. Dedupes against existing issues, attaches evidence (screenshot/log/repro), and cross-links related issues bidirectionally so the defect graph is navigable from any node.
category: QA
tags: [qa, issues, bug-report, github-issues, triage, linking]
---

Act as a **QA engineer filing defect reports**. The rule this skill enforces:
**an issue found is an issue filed** — every defect, regression, or suspicious
behavior discovered while running as QA gets its own GitHub issue *first*,
before it is discussed, worked around, or fixed. A bug that lives only in
chat, a test report, or someone's memory does not exist to the team.

**Input**: One or more findings to file (e.g.,
`/qa:issue-report upload fails for PDFs over 10MB`), or a source artifact to
sweep for unfiled findings (e.g., `/qa:issue-report e2e/E2E-REPORT.md`).
Run with no arguments after a QA session to be walked through everything
observed but not yet filed. Optionally `--repo owner/name` (defaults to the
current repo).

**Principles**

- **Issue first, everything else second.** Filing is not the last step of QA
  — it happens the moment a finding is confirmed. Fixing, triaging, and
  prioritizing all start from the issue, never precede it.
- **One issue per finding.** Never bundle unrelated defects into one issue;
  a bundled issue can't be assigned, closed, or linked independently. If two
  symptoms might share a root cause, file both and link them.
- **Linked, not orphaned.** Every filed issue is connected to what it
  relates to — the e2e report or test run that surfaced it, the sprint
  umbrella issue, the story whose acceptance criteria it violates, and any
  sibling issues that look related.

**Steps**

1. **Collect the findings**

   Gather every candidate defect from the source at hand:

   - Findings the user names directly
   - The **Observations** and failed/skipped flows of an e2e report
     (`e2e/E2E-REPORT.md` from `/qa:e2e-test`)
   - Failing test output, console/server errors, or broken acceptance
     criteria noticed during any QA activity

   For each candidate, confirm it is a real finding: what was done, what was
   expected, what actually happened, and what evidence exists (screenshot,
   log excerpt, failing command output). A "finding" without evidence or a
   reproduction path is a question, not an issue — park it and ask the user.

2. **Verify UI findings live in the browser (via MCP) before filing**

   A UI finding is not filed from hearsay or from a stale report — reproduce
   it yourself first:

   - Drive a real browser through the browser MCP (the Claude-in-Chrome MCP
     tools, or the project's Playwright setup if that's what the repo uses)
     against the environment where the finding was observed — the test box,
     the deployed URL, or a locally launched instance
   - Walk the exact reproduction steps and capture a **fresh screenshot of
     the defect state** — this screenshot, taken during verification, is the
     evidence that goes in the issue (plus console/network errors surfaced
     by the browser session, if any)
   - **While the browser is open, capture the app's versioning** for the
     issue's Environment section: the version/commit shown in the UI
     (footer, about page) or exposed by the app (`/health`, `/version`,
     `/api/version` endpoint); if the app exposes none, record the deployed
     git commit (`git rev-parse HEAD` of the synced branch on the box) and
     flag "app exposes no version info" as its own minor finding
   - Outcomes: **reproduces** → file it with the fresh evidence;
     **does not reproduce** → do not file; note it back to the source
     ("observed in e2e run X, not reproducible on <version> — possibly
     fixed or environment-specific") and ask the user; **cannot verify**
     (environment down, no URL) → say so explicitly and let the user decide
     whether to file as unverified — never file it silently as if verified
   - Non-UI findings (API errors, failing tests, log-only defects) skip the
     browser but still get their versioning captured the same way (version
     endpoint or deployed commit) before filing

3. **Dedupe against existing issues**

   Before filing anything:

   ```bash
   gh issue list --state all --limit 200 --search "<keywords>" \
     --json number,title,state,labels
   ```

   For each finding, decide:

   - **No match** → file a new issue (step 4)
   - **Same defect, issue open** → don't file; add a comment with the new
     evidence/occurrence ("Also reproduced: <context>, see <evidence>")
   - **Same defect, issue closed** → reopen-or-refile is the user's call;
     propose one with the evidence attached
   - **Similar but distinct** → file a new issue and link it (step 5)

4. **File one issue per finding**

   ```markdown
   Title: <symptom, specific and searchable — "PDF upload >10MB returns 500",
          not "upload broken">

   ## What happened
   <observed behavior, 1–3 sentences>

   ## Expected
   <what should have happened, citing the acceptance criterion / PRD story
   if one exists>

   ## Steps to reproduce
   1. <exact steps, including data used — reference fixture files, not
      production data>
   2. ...

   ## Evidence
   - <fresh screenshot from the browser verification (UI issues) / log
     excerpt / failing command + output>
   - <console or network errors captured during verification, if any>
   - Found during: <e2e run `e2e/screenshots/<run-id>/` | manual QA session
     <date> | reviewing #N>
   - Verified: <reproduced via browser MCP on <date> | not UI — verified via
     <command/log> | UNVERIFIED (env unavailable, filed on user's call)>

   ## Environment
   - **App version:** <version string or deployed commit SHA — from the UI,
     `/version`-style endpoint, or the synced branch; never "unknown"
     without saying why>
   - **Where:** <environment + URL tested against>
   - **Browser:** <browser + version, for UI issues>

   ## Severity (proposed)
   <blocker — flow cannot complete | major — flow completes with wrong
   result | minor — cosmetic/annoyance> — final call is the team's
   ```

   Create with `gh issue create`, applying labels that exist in the repo
   (`bug` plus severity/area labels if present — check with
   `gh label list`; never invent new labels without asking).

5. **Link related issues — both directions**

   Linking is what makes the pile of issues a map instead of a heap. For
   every issue filed, add the links that apply:

   - **Sibling defects** (similar symptom, shared suspect area):
     `Related to #N` in the body — and comment on #N with
     `Related: #<new>` so the link is visible from both ends
   - **Suspected same root cause**: file both, mark the weaker one
     `Possible duplicate of #N` and let triage decide
   - **Dependency**: `Blocked by #N` / comment `Blocks #<new>` on #N when
     one defect cannot be verified fixed until another is
   - **Provenance**: link the source artifact — the e2e report path + run
     id, the umbrella sprint issue (`Found during Sprint #N`), or the story
     issue whose acceptance criteria the defect violates
     (`Violates acceptance criteria of #N`)
   - **Fix tracking**: when a fix PR appears later, it references the issue
     (`Fixes #N`) — the issue is the anchor, so never let a fix land
     without one

   GitHub only auto-links one direction; the comment on the other issue is
   what makes the relationship discoverable from either side. Do both,
   every time.

6. **Report back**

   Summarize what was filed, skipped, and linked:

   > "Filed 3 issues: #41 (blocker, PDF >10MB 500s), #42 (major, wrong
   > total on dashboard, related to #41), #43 (minor, tooltip typo).
   > Skipped 1: duplicate of open #37 — added new evidence as a comment.
   > All linked to e2e run `e2e/screenshots/20260722-.../` and sprint
   > umbrella #30."

   If a sprint umbrella issue is open, post a short comment there listing
   the new defect issues so the PM sees them in the coordination point.

**Guardrails**

- Never let a finding exist only in chat or a report — if it's real, it's
  an issue; if it's not worth an issue, say so explicitly and why
- One finding, one issue — no bundling; split rather than lump
- Always dedupe before filing; a duplicate issue costs the team more than a
  missed one, and commenting on the existing issue is the better move
- Filing more than ~5 issues at once is bulk filing — present the list to
  the user for confirmation before creating them
- Every issue needs evidence and a reproduction path; never file from
  memory or speculation, and never fabricate screenshots or logs
- UI findings are verified live in a browser (via MCP) before filing; a
  finding filed without verification is explicitly marked UNVERIFIED and
  only filed on the user's call
- Every issue records the app's versioning (version string or deployed
  commit) — a bug report that can't say what version it was seen on can't
  be confirmed fixed later
- Links go both directions — an issue that mentions #N gets a reciprocal
  comment on #N
- Propose severity, don't decree it — priority calls belong to the team
- Never include production data, credentials, or real personal information
  in an issue body or its attachments; sanitize logs before pasting
- This skill files and links issues — it does not fix bugs, close issues it
  didn't file, or reprioritize the sprint
