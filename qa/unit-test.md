---
name: "QA: Unit Test"
description: Write or extend the unit-test suite for a component — fast, isolated, deterministic tests that mock the underlying dependencies (DB, HTTP, queue, clock, filesystem) only where a real object won't do. Covers Go, Python, and TypeScript; classifies every dependency as real/fake/stub/mock with a reason, and proves each test can actually fail before trusting it.
category: QA
tags: [qa, testing, unit-test, mocks, fakes, tdd, golang, python, typescript, pytest, vitest, coverage]
---

Act as a QA engineer writing the **unit-test suite** for a component. A unit
test pins one unit's behavior in milliseconds, with no network, no real
database, no wall clock, and no ordering dependency on any other test — which
means every underlying dependency is either injected as a real object, a
fake, or a mock.

The judgment this skill exists to apply is *which one*:

> **Mock only where a real object won't do — and mock the seam you own,
> never the library you don't.**

Every mock is a claim about how a dependency behaves, and claims drift from
reality. That drift is exactly what `/qa-integration-test` catches with real
containers; this suite's job is speed, isolation, and precise failure
localization.

**Input**: Optionally a package, module, or file to cover (e.g.
`/qa-unit-test internal/billing`, `/qa-unit-test src/parser.ts`,
`/qa-unit-test review`). If omitted, find the highest-value untested logic
in the repo and start there.

**Principles**

- **Real object > in-memory fake > stub > mock with expectations** —
  work down that list and stop at the first one that works. A pure
  function needs nothing; an interface with three methods deserves a
  15-line hand-written fake before a mocking framework
- **Mock at your own boundary** — mock the `Storage`/`PaymentGateway`
  interface *you* defined, not `*sql.DB`, `requests`, or `fetch`.
  Mocking a third party's internals couples your test to their
  implementation, so their patch release breaks your suite while your
  code is fine
- **Test behavior, not implementation** — assert on the returned value,
  the state change, the error. "The mock was called once with X" tests
  your own call graph, and it is the assertion that makes refactoring
  expensive. Use it only when the call *is* the behavior (an audit
  event was emitted, an email was sent)
- **Inject what you'd otherwise have to mock** — clock, UUID generator,
  randomness, and the filesystem root are constructor parameters. A
  test that needs `freezegun` or `vi.useFakeTimers()` is usually a
  design telling you `time.Now()` is being called too deep
- **Determinism is non-negotiable** — no `sleep`, no real network, no
  shared temp paths, no dependence on test order or on a map's
  iteration order. A flaky unit test is worse than no test: it trains
  the team to ignore red
- **A test you haven't seen fail proves nothing** — before trusting a
  new test, make it fail on purpose (break the assertion or the code).
  Tests that pass against broken code are the most expensive artifact
  in a repo
- **Coverage is a diagnostic, not a target** — read the report to find
  the untested *branch*, then decide if it matters. A team optimizing
  a percentage writes tests for getters and leaves the error path bare

**Steps**

1. **Scope the unit and read what exists**

   Identify the unit under test and its current coverage. Read the code
   before writing anything — its actual behavior, including the error
   paths, is the specification here (unless this is TDD for new code
   under `/engineer-implement`, where the acceptance criteria are).

   ```bash
   go test ./... -cover                     # Go
   pytest --cov=<pkg> --cov-report=term-missing -q   # Python
   npx vitest run --coverage                # TypeScript
   ```

   Note the language's existing test conventions and reuse them — do
   not introduce a second test framework alongside the one in place.

2. **Classify every dependency before writing a line**

   List what the unit touches and decide the treatment for each, with
   the reason. This table goes in the PR description:

   | Dependency | Treatment | Why |
   |------------|-----------|-----|
   | Pure formatter | real | no IO, no reason to fake |
   | `UserRepo` interface | in-memory fake | needs state across calls |
   | `PaymentGateway` | mock | asserting the charge call *is* the behavior |
   | `time.Now` | injected clock | determinism |
   | Postgres itself | **not here** | real SQL → `/qa-integration-test` |

   If the answer for a dependency is "I'd have to spin up Postgres",
   that test belongs in the integration suite, not this one.

3. **Make the code testable, or record why it isn't**

   Untestable shapes — package-level globals, `init()` side effects, a
   constructor that dials the network, business logic inside an HTTP
   handler — are design findings, not test problems. Prefer the small
   seam (extract an interface, take a parameter, move IO to the edge)
   and note it in the PR. If the refactor is larger than the test,
   stop and raise it via `/qa-issue-report` or `/architect-design
   review` rather than building an elaborate mock scaffold to test
   around a design flaw.

4. **Write the tests, in the language's own idiom**

   - **Go** — table-driven with named subtests and `t.Parallel()`;
     hand-written fakes implementing your interfaces (gomock/mockery
     only when the interface is genuinely wide); `httptest.Server` for
     an HTTP client's boundary, `testing/fstest.MapFS` for filesystem;
     `t.Cleanup` over manual teardown; assert with `cmp.Diff` for
     structs. Never `time.Sleep` — inject the clock
   - **Python** — `pytest` with `parametrize` for the table; fixtures
     for construction, `monkeypatch` for env; `unittest.mock.patch`
     **always with `autospec=True`** (without it, a typo'd method name
     silently passes and the test proves nothing), and patch **where
     the name is used**, not where it is defined; `pytest-mock`'s
     `mocker` for teardown safety; a fake clock injected over
     `freezegun` where the code allows
   - **TypeScript** — Vitest (or Jest, matching the repo); typed mocks
     via `vi.mocked()`/`jest.mocked()` so a signature change breaks the
     test at compile time; **MSW to intercept at the HTTP boundary**
     rather than stubbing `fetch` by hand; fake timers for scheduling;
     `@testing-library/react` for components, asserting what the user
     sees (roles, text) — never internal state or class names.
     Snapshots only for genuinely stable serialized output, never as
     the primary assertion
   - Name tests for the behavior, not the method:
     `TestTransfer_InsufficientFunds_ReturnsErrAndLeavesBalance`, not
     `TestTransfer2`. Arrange/act/assert, one behavior per test, and
     assert on the error *value or type*, not on the message string

5. **Prove the tests can fail**

   For each new test, either watch it fail first (TDD order) or mutate
   the production code afterwards and confirm red. Spot-check with a
   mutation run where the tooling exists (`go-mutesting`, `mutmut`,
   `stryker`). Then run the whole suite repeatedly and in random order
   to smoke out inter-test coupling:

   ```bash
   go test ./... -race -count=5 -shuffle=on
   pytest -q -p no:randomly --count=3   # or pytest-randomly enabled
   npx vitest run --sequence.shuffle
   ```

   `-race` is not optional for Go code with goroutines.

6. **Wire into CI as a gate**

   The unit lane runs on every PR, in seconds to a couple of minutes,
   with the type check beside it (`go vet`, `mypy --strict`,
   `tsc --noEmit` — per `/architect-design`). Integration tests run in
   a separate lane so this one stays fast: mark them (`testing.Short`,
   `@pytest.mark.integration`, a separate vitest project) and exclude
   them here.

7. **Report**

   > "Unit tests for `<scope>`: N tests added across M files, branch
   > coverage `<before>%` → `<after>%`. Dependencies: X real, Y fakes,
   > Z mocks (table in PR). All verified failing-then-passing; suite
   > green under `-race`/shuffle in `<duration>`. Findings: `<untestable
   > shapes or real defects, as issues>`."

   Any genuine defect the new tests uncover is a finding, not a test
   bug — file it via `/qa-issue-report` before fixing anything.

**Guardrails**

- Never mock the unit under test — if it needs mocking to be tested,
  the test is testing a mock
- Never write a test whose only assertion is that a mock was called,
  unless the call itself is the observable behavior
- Never use `sleep`, real network calls, real cloud credentials, or a
  real database in this suite — those belong to
  `/qa-integration-test`
- Never weaken, skip, or delete a failing test to make the suite green;
  a red test is a finding until proven otherwise. `@skip`/`t.Skip`
  needs a linked issue in the same commit
- Never chase a coverage percentage — cover branches that matter, and
  say plainly which risky paths remain uncovered
- Never patch without `autospec`/typed mocks; a mock that accepts any
  call signature will happily outlive the API it was pretending to be
- Do not restructure production code beyond the small seam a test
  needs — larger refactors go through `/engineer-implement` with the
  design's blessing
