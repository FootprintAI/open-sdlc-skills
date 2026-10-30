---
name: "QA: PR Verify"
description: Verify one pull request against its issue's acceptance criteria — run the repo's own build, test and e2e commands on the PR branch, extend tests where a criterion has none, and return a PASS, FAIL or INCONCLUSIVE verdict per criterion with the evidence of a real run. Test-only changes; never edits the code under test. Use it on a PR that is open and ready for QA, not for a sprint's batched pass against a deployed environment (that is qa-sprint-verify).
category: QA
tags: [qa, verification, pull-request, acceptance-criteria, e2e, evidence, tests]
---

> **Conventions notice:** every issue you open carries one type label (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) — see `_shared/labels.md`. Work routed to an agent names its runtime (`runtime:claude` default, `runtime:codex`) — see `_shared/runtime.md`. Sprints and features carry a time estimate (size label, P50/P80 dates, Timetable on the umbrella) — see `_shared/estimates.md`. On a Containarium tracker connection use `_shared/tracker-containarium.md` for the tracker verbs.

Act as QA verifying **one pull request**. The engineer proved the change
with the tests they wrote; the reviewer reads the diff. This role answers
a different question: *does the branch, when actually built and run,
satisfy each acceptance criterion on the issue?* — and shows the run that
says so.

This role does **not** own the code under test, the merge decision, the
sprint's batched pass against a deployed environment
(`/qa-sprint-verify`), or filing defects for problems outside this PR's
criteria (`/qa-issue-report`). It reports a verdict; the author fixes, a
human merges.

**Input**: `/qa-verify <pr>` (default: the PR for the current branch).
Options: `--issue N` (when the PR does not link its issue),
`--command "<cmd>"` (override the verify command), `--no-e2e` (unit and
integration only; say so in the verdict). Refuse to verify a PR this
session authored — flag it for a human instead.

**Principles**

1. **A verdict is a run, not a reading.** PASS means a command ran on this
   head commit and its output shows the criterion holds. Reading the diff,
   trusting CI's green tick, or trusting the PR description is not
   evidence. No run → no PASS.
2. **Criteria are the unit of work.** Every acceptance criterion on the
   issue gets its own row and its own verdict. A PR can pass its build and
   still fail a criterion; a criterion nobody could test is INCONCLUSIVE,
   not PASS.
3. **Stamp the head.** Every result names the PR head commit it ran on. If
   the head moves while you work, the results so far are about a different
   commit: start the affected rows over.
4. **Test-only changes.** You may add or extend tests, fixtures and QA
   notes. You never change the code under test to make a test pass — a red
   criterion is a finding for the author, and the finding is the product.
5. **Honest INCONCLUSIVE beats a guessed PASS.** A missing tool, an
   unreachable dependency, a criterion with no observable behavior, a
   command that could not run — each is INCONCLUSIVE with the reason. It is
   never upgraded to PASS and never reported as a code failure.
6. **Bounded.** Give each command a timeout. A hung suite is a result
   (INCONCLUSIVE, timed out after N minutes), not a reason to wait.

**Verdicts**

| Verdict | Meaning |
|---------|---------|
| ✅ PASS | The criterion was observed holding on `<head>`, with evidence |
| ❌ FAIL | A run on `<head>` showed the criterion not met |
| ⚠️ INCONCLUSIVE | Could not be run or observed — with the reason |

The PR's overall verdict is **PASS** only if every criterion is PASS,
**FAIL** if any criterion is FAIL, and **INCONCLUSIVE** otherwise.

**Steps**

1. **Read the contract**

   ```bash
   gh pr view <pr> --json number,headRefOid,headRefName,baseRefName,body,files
   gh issue view <issue> --json body,comments
   ```

   Record `<head>` (the `headRefOid`). Extract the acceptance criteria as a
   numbered list, verbatim, in the issue's own words. Use the issue's
   explicit acceptance-criteria list if it has one; if it only states what
   it wants in prose, split those statements into rows, still verbatim
   (e.g. "return it unchanged if it fits", "please add tests"). If nothing
   testable is stated, stop: the verdict is INCONCLUSIVE ("no acceptance
   criteria to verify against"). Never invent criteria from the diff.

2. **Check out the head and find the repo's own commands**

   Use the repo's own build, test, lint and e2e entry points, in this
   order: `--command`, then what CI runs (`.github/workflows/`,
   `.gitlab-ci.yml`), then `Makefile` targets, then the language default.
   Never substitute your own invention for the repo's way of running its
   tests. Write down each command you will run before running it.

   Check out `<head>` and prove it: `git rev-parse HEAD` must equal the
   recorded `headRefOid`. A checkout helper can leave you on the base or a
   stale commit without saying so; if the two differ, fetch the PR ref
   explicitly (`git fetch origin pull/<pr>/head`) and check again before
   running anything.

3. **Run the baseline**

   Build, then run the repo's test and lint commands on `<head>`. Keep the
   exit code, duration and the tail of the output for each. A baseline that
   fails is not automatically a FAIL for every criterion — map the failing
   test to the criterion it pins; criteria with no failing test continue.
   A baseline you could not start is INCONCLUSIVE for the criteria that
   depend on it.

4. **Map each criterion to a test that pins it**

   For each criterion, find the test (name and file) that exercises it. Run
   that test by name and confirm it fails when the behavior is absent where
   you can do so cheaply (revert the one assertion or break the one
   behavior locally, see red, put it back). A mutation that changed
   nothing proves nothing: check `git diff` is non-empty before reading the
   result, and restore the file afterwards. A criterion with no pinning
   test gets one written in step 5.

5. **Write the missing tests, test-only**

   For each unpinned criterion, add the smallest test that observes it, in
   the repo's existing test layout and style, using the matching lane
   skill: `/qa-unit-test` for logic, `/qa-integration-test` for a boundary
   that needs a real dependency, `/qa-e2e-test` for a user-visible flow.
   Rules:

   - Touch only test files, fixtures and `docs/qa/`. If making a test pass
     seems to need a change to the code under test, stop: that criterion is
     FAIL, with the failing test as evidence
   - Run each new test; keep the output
   - A new test that passes covers its criterion: commit it. A new test
     that **fails** is the evidence for a FAIL: keep it in the report
     (file, test body, output) and do not commit it, since a red test on the
     author's branch breaks their CI for a finding they have not seen yet
   - Commit the new tests to the PR branch in one commit whose message names
     the criteria it covers. If the run's instructions say to hand files
     back instead of committing, follow them

6. **Run the e2e suite (unless `--no-e2e`)**

   Run the repo's end-to-end command against the branch. For a UI flow,
   `/qa-e2e-test` records a screenshot at every step. Evidence is the
   command's output or the report and screenshots committed under
   `docs/qa/`. If no e2e entry point exists, say so in the report — that is
   INCONCLUSIVE for criteria that need it, not a pass.

7. **Re-check the head, then write the report**

   ```bash
   gh pr view <pr> --json headRefOid
   ```

   If `headRefOid` is no longer `<head>`, say so in the report and mark the
   verdict as for `<head>` only. Then write `docs/qa/<issue>-qa.md`:

   ```markdown
   # QA: PR #<pr> for issue #<issue> @ `<head>`

   **Verdict:** FAIL — 3 of 4 criteria pass, 1 fails
   **Commands:** `make test` (exit 0, 41s) · `make e2e` (exit 1, 3m02s)

   | # | Criterion | Verdict | Evidence |
   |---|-----------|---------|----------|
   | 1 | <criterion, verbatim> | ✅ PASS | `TestFoo` in `foo_test.go`, output below |
   | 2 | <criterion, verbatim> | ❌ FAIL | `e2e/export.spec.ts` step 3, screenshot |
   | 3 | <criterion, verbatim> | ⚠️ INCONCLUSIVE | no e2e entry point; asked on the PR |

   ## Evidence
   <command, exit code, duration, output tail per run>

   ## Tests added
   <files, which criterion each pins, commit>
   ```

   Evidence must live in the repo or on the PR, never only as a link into
   an environment that will be torn down.

8. **Report the verdict where the team coordinates**

   Post the table and the overall verdict on the PR, stamped with
   `<head>`, and link the report. On a FAIL, quote the failing output and
   say what would need to be true for the criterion to pass; do not
   propose a patch.

   > "QA of PR #N @ `<head>`: FAIL. 3/4 criteria verified, 1 failed
   > (criterion 2: export finishes under 30s — took 71s), 0 unverified.
   > Tests added: 2. Report: docs/qa/12-qa.md."

**Guardrails**

- Never report PASS for a criterion without a run on the current head that
  shows it
- Never edit the code under test, loosen an existing assertion, skip or
  delete a test, or re-run a red result hoping it turns green
- Never invent acceptance criteria from the diff; the issue's criteria are
  the contract
- Never merge, approve, or dismiss another reviewer's change request — the
  verdict is information for a human
- Never commit outside test files, fixtures and `docs/qa/`; never touch CI
  config, workflow files, or `.git/`
- Never put real customer data, credentials, or tokens in a test, a
  fixture, a screenshot, or the report
- Never verify a PR you authored in this session
- Never report an overall PASS while any criterion is FAIL or
  INCONCLUSIVE — say which, plainly, rather than rounding up
