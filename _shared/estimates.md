# Notice: every sprint and feature carries a time estimate

**Applies to `/pm-sprint-delivery`, `/team-sprint-cycle`, `/scrum-master`,
`/coordinator-status`, `/coordinator-sync`, and `/release-cut`** — anything
that scopes, tracks or reports on shipping. The goal is a clear timetable:
for each feature, *when will it ship, how sure are we, and how did we do*.

## What gets estimated

| Level | Estimate | Where it lives |
|---|---|---|
| Child issue (feature/defect) | size + elapsed range | `size:*` label + `**Estimate:**` line in the issue body |
| Sprint | start date, target ship date, confidence | umbrella issue, **Timetable** section |
| Release | target date = last verified feature + release lag | release-candidate issue |

All dates are absolute (`2026-10-14`), never "next week". Time is **elapsed
wall-clock**, not effort: the estimate must include waiting, because waiting
is where sprints actually slip.

## Size labels (exactly one per child issue)

| Label | Agent build time | Typical |
|---|---|---|
| `size:S` | under ~1 h | one file, fully specified |
| `size:M` | ~1–4 h | multi-file, clear design |
| `size:L` | ~4 h–1 day | cross-cutting or needs a design decision |
| `size:XL` | over 1 day | **split it** before the sprint starts |

Starting values only — replace them with the team's measured numbers (see
Calibration). `size:XL` is never accepted into a sprint.

## Per-issue estimate line

```
**Estimate:** build 2h · CI+review 4h · deploy+verify wait 1d → ships 2026-10-15 (P50) / 2026-10-17 (P80)
**Waiting on:** none | Q3 (open question) | #41 (dependency)
```

Break an estimate into **lanes**, because they have different owners:

1. **Build** — the agent's work (routed `model:*` / `runtime:*`)
2. **CI + review** — pipeline and reviewer queue
3. **Deploy + verify** — the sprint's single batched deploy and QA pass
4. **Human wait** — open questions, confirmation gates (prod), sign-ins

An issue blocked on an unanswered open question has **no start date**: show
`waiting on Q<n>` instead of inventing one. The clock starts when the
question is answered.

## Sprint Timetable (on the umbrella)

```
## Timetable   (forecast as of 2026-10-07)
Sprint start 2026-10-07 · target ship 2026-10-16 · confidence P50 10-15 / P80 10-17

| # | Feature | Size | Starts | Ships (P50/P80) | Waiting on | Actual | Δ |
|---|---------|------|--------|-----------------|------------|--------|---|
| #41 | Login flow | M | 10-07 | 10-09 / 10-10 | — | | |
| #44 | Billing export | L | after #41 | 10-13 / 10-15 | Q2 | | |

Critical path: #41 → #44 → deploy → verify. Buffer: 1d (deploy+verify).
```

Rules:
- Order by dependency; the **critical path** is named, the rest runs in
  parallel under the WIP cap
- Give a **range** (P50 / P80), never a single date; a single date claims
  precision nobody has
- Add an explicit **buffer** for the batched deploy + verification pass —
  it is one event for the whole sprint, not per issue
- Cross-repo dependencies and open questions are *waiting* rows, not dates

## Calibration

- Source of truth is evidence, not memory: median and P80 **claim-to-merge**
  days per size from the last N sprints (`/scrum-master` already reports
  cycle time) plus deploy-to-verified lag
- First sprint, or fewer than 3 closed issues of a size: mark the forecast
  **uncalibrated** and widen the range — say so on the umbrella, don't hide it
- Do not convert agent speed into human-hours; estimate elapsed time the
  team will actually observe

## Keeping it honest

- **Re-forecast** at every standup (`/scrum-master`): update the Timetable,
  and record the new date next to the old one — never overwrite history
- **Slip rule:** a feature's P80 moving later by more than its buffer, or the
  sprint's target ship date moving at all, is an *impediment* with an owner,
  not a quiet edit
- **Retro** (cycle close): fill Actual and Δ for every shipped feature,
  compute per-size accuracy, and feed it to the next sprint's calibration
- `/coordinator-sync` surfaces the soonest and latest target ship date per
  project, and flags any past its P80 as **stalled**
- `/release-cut` quotes the verified ship dates; it never forecasts a
  release date earlier than the last unverified feature's P80

## Containarium trackers

`size:*` is not in a connection's default label allow-list
(`scope:*`, `model:*`, `agent:needs-approval`). Add `size:*` with the other
labels (`/devops-containarium-env` capability 9), or keep the size only in
the issue body's `**Estimate:**` line and report the rejected label.
