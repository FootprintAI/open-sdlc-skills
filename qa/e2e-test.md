---
name: "QA: E2E Test"
description: Run a happy-path end-to-end test against a web/UI project with a screenshot recorded at every step as proof of execution. Collects sample data (PDF/image/etc.) from the user before testing, exercises success paths only, and writes a flow-by-flow report with embedded screenshots to an md file.
category: QA
tags: [qa, e2e, testing, screenshots, web, ui, happy-path]
---

Act as a QA engineer and run an **end-to-end happy-path test** of the current
web (or UI-related) project. Every tested step must be captured as a screenshot
— screenshots are the proof that the test actually ran, not a nice-to-have.
Edge cases are explicitly out of scope: success paths only.

**Input**: Optionally specify flows to test (e.g., `/qa:e2e-test login,upload`)
or a base URL of an already-running instance (e.g.,
`/qa:e2e-test http://localhost:3000`). If omitted, discover flows and launch the
app yourself.

**Called from a sprint verification pass**: when `/qa:sprint-verify` hands
this skill a deployed URL and a set of flows, they are the sprint's UI
verification items and they run as **one session against that one deployed
commit** — not one run per issue. Record the commit in the report header
alongside the URL, skip the "launch the app" step entirely (the environment
is already the thing under test), and never redeploy or restart it. Its
test-data cleanup rules (step 6) still apply to everything the run creates.

**Applies to**: Web/UI projects only (SPA, SSR app, static site, dashboard,
Electron renderer, etc.). If the project has no UI surface, stop and tell the
user this skill does not apply — do not force a browser onto an API-only or
CLI project.

**Test environment**

Unless the user supplied a base URL of an already-running instance, the app
under test runs in a clean, disposable environment:

1. **Default: a fresh, disposable box.** Check what this environment can
   actually provision before settling for the weakest option — in
   preference order: a **connected container-platform MCP server or CLI**
   that can create a box and expose a port on demand, a preview deployment,
   a container built from the project's compose file or `Dockerfile`, then a
   local VM. Create a *fresh* one, deploy the project into it, and point the
   browser at the exposed port. A fresh box guarantees the run isn't
   polluted by leftover state from earlier runs
2. **If the box is not working, investigate — don't silently route around
   it.** Find out why (image pull failure, resource limits, network, host
   down) and report what you found; an environment that won't come up is
   itself a finding
3. **Fallback: a locally launched VM or container.** If the usual
   environment is genuinely unavailable, launch the app in a locally
   started VM or container and test against that. Record in the report that
   the fallback was used and why

**Steps**

1. **Confirm this is a UI project and map the happy-path flows**

   Inspect the repo to confirm a UI exists and identify the core user flows:

   - `package.json` scripts (`dev`, `start`, `serve`), framework markers
     (React/Vue/Svelte/Next/Nuxt/Angular/vanilla), templates, or an
     `index.html`
   - Routes/pages directory, navigation components, and the README's stated
     purpose

   From this, list the **happy-path flows** worth testing — the 2–6 journeys a
   successful user takes (e.g., "sign in → land on dashboard",
   "upload a PDF → see parsed result", "fill form → submit → see confirmation").
   Present the flow list to the user before executing so they can add or drop
   flows.

2. **Request sample data from the user BEFORE testing**

   Determine what input data each flow needs — PDF, image, CSV, credentials,
   or other domain files. Then **ask the user to provide samples**:

   > "To run these flows end-to-end I need sample data. Please drop the files
   > into `e2e/fixtures/` (or give me paths):
   > - <flow A>: a sample PDF
   > - <flow B>: a sample image (png/jpg)
   > - <flow C>: test credentials (never production credentials)"

   Rules for sample data:
   - Wait for the user's response — do not fabricate binary fixtures for
     flows whose whole point is processing a real file
   - If the user says "generate it yourself" or the data is trivial (plain
     text, form field values), create it under `e2e/fixtures/` and note in the
     report that the fixture was auto-generated
   - Never use production data or real personal information; if the user
     hands over something that looks like production data, flag it and ask
     for a sanitized sample

3. **Set up browser automation with screenshot capture**

   - If the repo already has an e2e tool configured (Playwright, Cypress,
     WebdriverIO), use it
   - Otherwise use Playwright as the default: install it as a dev-only
     concern (prefer `npx playwright` / a throwaway script under `e2e/`;
     do not restructure the project around it)
   - Create the artifact layout:

     ```
     e2e/
       fixtures/                      # sample data from step 2
       screenshots/<YYYYMMDD-HHmmss>/ # one dir per run
         <flow-name>/01-<step>.png
         <flow-name>/02-<step>.png
       E2E-REPORT.md                  # written in step 6
     ```

4. **Launch the app**

   If the user supplied a base URL, use it. Otherwise launch the app in a
   fresh environment per **Test environment** above (disposable box first, a
   locally launched VM or container as fallback), start it the way the
   project intends (`npm run dev`, `docker compose up`, etc.), wait for it
   to become reachable, and record the environment, exact command, and URL
   for the report. If the app fails to start, stop and report the failure —
   do not pretend to test against a dead server.

5. **Execute each flow — happy path only, screenshot every step**

   For each flow:

   - Perform only the success path: valid inputs, expected navigation,
     expected result. **Do not test edge cases** — no invalid input, no
     error-state probing, no boundary values. If you notice a bug while
     passing through, note it in the report as an observation but do not
     chase it
   - Capture a screenshot at every meaningful step: initial page, after each
     significant interaction (form filled, file selected, button clicked),
     and the final success state. Name them in order:
     `01-landing.png`, `02-form-filled.png`, `03-upload-selected.png`,
     `04-success.png`
   - Assert the success condition from what is actually on screen (visible
     text, element state), and keep the screenshot that shows it
   - A flow with zero screenshots is a flow that was not tested — if
     screenshots failed to write, the flow result is FAILED (infrastructure),
     not PASSED
   - **Track everything the test creates** — uploaded files, created records,
     registered accounts, submitted forms. Keep a running list of entity
     type + identifier (id, name, URL); step 6 uses it to clean up

6. **Clean up test data via the app's API or web UI**

   Once testing is done, remove the data the run created so the app is left
   the way you found it:

   - **Prefer the app's API** (REST/GraphQL endpoint discovered from the
     code or docs) to delete each tracked entity — it is scriptable and
     verifiable
   - **Fall back to the web UI** if there is no API for the operation: drive
     the same browser session through the app's own delete/remove action,
     and screenshot the confirmation as proof of cleanup
   - Verify deletion (entity no longer listed / GET returns 404) and record
     per-entity cleanup status for the report
   - Delete only what this run created — never bulk-delete or wipe data that
     existed before the test
   - If an entity has no delete path at all (API or UI), leave it, and list
     it in the report under "Residual test data" so the user can remove it
     manually

7. **Write the report to `e2e/E2E-REPORT.md`**

   ```markdown
   # E2E Happy-Path Test Report

   **Date:** <today>
   **App:** <name> @ <url>
   **Version under test:** `<commit>` (required when testing a deployed environment)
   **Environment:** disposable box `<id>` | local VM/container (fallback: <why>) | user-provided URL | deployed `<env>`
   **Launch command:** `<command>`
   **Tool:** Playwright <version> (chromium)
   **Run artifacts:** e2e/screenshots/<run-id>/
   **Scope:** Happy path only — edge cases intentionally excluded

   ## Summary

   | Flow | Result | Steps | Screenshots |
   |------|--------|-------|-------------|
   | Login | ✅ PASS | 4 | 4 |
   | PDF upload | ✅ PASS | 5 | 5 |

   ## Flow: <name>

   **Sample data:** e2e/fixtures/<file> (provided by user | auto-generated)

   | # | Step | Expected | Observed | Screenshot |
   |---|------|----------|----------|------------|
   | 1 | Open /login | Login form visible | Form rendered | ![](screenshots/<run-id>/login/01-landing.png) |
   | 2 | Submit valid credentials | Redirect to /dashboard | Redirected, greeting shown | ![](screenshots/<run-id>/login/02-dashboard.png) |

   ## Test data cleanup

   | Entity | Created by | Removed via | Verified |
   |--------|-----------|-------------|----------|
   | user `qa-e2e-20260720@example.com` | Login flow | API `DELETE /api/users/:id` | ✅ GET returns 404 |
   | document `sample.pdf` (id 42) | PDF upload flow | Web UI delete button | ✅ no longer listed ([screenshot](screenshots/<run-id>/cleanup/01-deleted.png)) |

   ### Residual test data (no delete path — remove manually)
   - <entity + where it lives>, or "none"

   ## Observations (not tested, noted in passing)
   - <anything odd seen while walking the happy path>

   ## Not covered
   - Edge cases, error states, and invalid inputs — out of scope by design
   - <flows skipped and why, e.g., missing sample data>
   ```

   Every flow section must reference its actual screenshot files; a claim
   without a screenshot does not belong in the report.

8. **Present summary and shut down**

   - Stop any server you started, and tear down the test environment you
     created (delete the box / stop the VM or container) — a fresh box per
     run means no box outlives its run
   - Summarize to the user: flows passed/failed, where the report and
     screenshots live, which flows (if any) were skipped waiting on sample
     data, and whether all test data was removed
   - If the run produced failed flows or observations, remind the user to
     file them as GitHub issues with `/qa:issue-report e2e/E2E-REPORT.md` —
     a defect that lives only in the report isn't tracked

   > "E2E happy-path run complete: N/M flows passed. Report:
   > `e2e/E2E-REPORT.md`, screenshots: `e2e/screenshots/<run-id>/`.
   > Test data cleaned up: N/N entities removed (or: 1 residual, see report).
   > Skipped: <flow> (waiting on sample PDF)."

**Guardrails**

- Happy path ONLY — never spend time on edge cases, error states, or invalid
  input; the deliverable is proof that the success paths work
- Every reported step needs a real screenshot on disk; never report a step as
  passed without one, and never fabricate or reuse screenshots across runs
- Ask for sample data before testing; do not invent stand-in PDFs/images for
  flows that exist to process real files unless the user tells you to
- Never use production credentials or real personal data
- Always clean up the data the run created, via the app's API or its own web
  UI — and delete ONLY what this run created. Never bulk-delete, never
  truncate tables, never touch pre-existing data. If you cannot tell whether
  an entity came from this run, leave it and report it as residual
- Do not modify the application source code; only add files under `e2e/`
- Report results faithfully — a failed or skipped flow is reported as such,
  with the failing screenshot attached
- Run against a locally launched instance or a URL the user provided — never
  point the test at a production environment unless the user explicitly asks
