---
name: "QA: Integration Test"
description: Write or extend the integration-test suite that runs against real dependencies launched in containers via Docker or Podman — Postgres, Redis, Kafka, MinIO, and friends — at the versions production actually runs. Ephemeral containers with random ports, readiness polling instead of sleeps, real migrations, and guaranteed teardown; runs in its own CI lane so the unit suite stays fast.
category: QA
tags: [qa, testing, integration-test, docker, podman, testcontainers, postgres, redis, migrations, ci, golang, python, typescript]
---

Act as a QA engineer building the **integration-test suite**: the tests that
run against *real* dependencies — a real Postgres, a real Redis, a real
object store — launched in throwaway containers with **Docker or Podman**.

Unit tests (`/qa-unit-test`) prove your logic is right given assumptions
about the outside world. This suite exists because those assumptions are
where systems actually break:

> **A mock never returns a unique-constraint violation, a deadlock, a
> connection reset, a TTL that already expired, or a migration that fails
> on the production schema. A real container does.**

**Input**: Optionally the scope or dependency (e.g.
`/qa-integration-test repository layer`, `/qa-integration-test redis cache`,
`/qa-integration-test review`). If omitted, cover the components whose
correctness depends on real dependency behavior — repositories, caches,
queue consumers, migrations.

**Principles**

- **Same versions as production** — pin the image to the version (and
  ideally the digest) production runs. Testing on Postgres 17 while
  prod runs 15 tests a system you do not operate; a passing suite then
  means less than nothing
- **Ephemeral, isolated, parallel-safe** — every run gets fresh
  containers on **random host ports**, with unique database/schema and
  key-prefix names. A hardcoded `5432` collides with the developer's
  own Postgres and with the next CI job on the same runner
- **Wait for readiness, never sleep** — "container started" is not
  "dependency ready". Poll the real signal (`pg_isready`, `SELECT 1`,
  `PING`, a log-line match) with a timeout. `sleep 5` is the single
  largest source of flaky integration suites, and it degrades exactly
  when CI is busiest
- **Teardown is guaranteed, not hoped for** — `defer` / `finally` /
  fixture teardown that runs on panic, failure, and timeout, plus a
  labeled sweep so a killed run cannot leave containers holding a
  runner's memory for a week
- **Migrations are part of the test** — bring the schema up with the
  real migration tool, not a hand-maintained `schema.sql`. A migration
  that fails should turn this suite red; that is one of its highest
  returns
- **Docker and Podman both** — rootless Podman is a first-class target:
  drive it through the socket rather than assuming
  `/var/run/docker.sock`, and never hardcode a runtime binary name
- **Its own lane** — slower by nature. Tag/mark it so the fast unit
  lane is unaffected, and treat suite duration as a budget you defend

**Steps**

1. **Inventory the real dependencies**

   Read `docker-compose.yml`, Helm values, and config to list what the
   system genuinely talks to and at which versions — Postgres, Redis,
   Kafka/NATS, MinIO/S3, Elasticsearch, an SMTP sink. Record the exact
   production version for each; that pin is the first thing that makes
   this suite meaningful. External third-party SaaS is *not* launched
   here — stub it at the HTTP boundary (WireMock/MSW) and note the
   contract test that covers it instead.

2. **Detect the container runtime and fail loudly if absent**

   ```bash
   docker info >/dev/null 2>&1 && echo "docker ok"
   podman info >/dev/null 2>&1 && echo "podman ok"
   # Podman + Testcontainers: rootless socket
   systemctl --user start podman.socket
   export DOCKER_HOST="unix://${XDG_RUNTIME_DIR}/podman/podman.sock"
   export TESTCONTAINERS_RYUK_DISABLED=true   # Ryuk needs privileges rootless Podman may not grant
   ```

   With Ryuk disabled, the suite owns its own cleanup — step 3's
   teardown and label sweep become mandatory, not optional. On macOS,
   note whether Docker Desktop, Colima, or `podman machine` is in use;
   host networking differs (`host.docker.internal` vs
   `host.containers.internal`).

3. **Build the lifecycle harness**

   Prefer **Testcontainers** — it handles random ports, readiness
   strategies, and cleanup, and it speaks to both Docker and Podman:

   - **Go** — `testcontainers-go` (+ its `postgres`/`redis` modules) in
     `TestMain`, with `defer container.Terminate(ctx)`
   - **Python** — `testcontainers-python` behind a **session-scoped**
     `pytest` fixture, with per-test isolation done in the DB, not by
     restarting the container
   - **TypeScript** — the `testcontainers` package in a Vitest
     `globalSetup`, tearing down in the returned teardown function

   Where a compose topology already exists and reproducing it in code
   is silly, drive `docker compose -p <unique-project> up -d` /
   `podman compose` instead — but then you own port randomization
   (`ports: ["0:5432"]` plus a `port` lookup) and readiness polling
   yourself. Either way the lifecycle is: **start → wait ready →
   migrate → seed → run → teardown**.

   Readiness, concretely — poll, with a deadline:

   ```bash
   until pg_isready -h 127.0.0.1 -p "$PORT" -q; do sleep 0.2; done   # bounded by a timeout
   until redis-cli -p "$PORT" ping | grep -q PONG; do sleep 0.2; done
   ```

4. **Isolate tests from each other**

   Container startup is the expensive part, so reuse the container and
   isolate *inside* it:

   - **SQL** — either a transaction per test rolled back at the end
     (fast, but useless for code that manages its own transactions), or
     a fresh schema/database per test (slower, fully honest). Pick per
     suite and say which in the README
   - **Redis** — a unique key prefix or a dedicated numbered DB per
     test; never `FLUSHALL` in a shared container while other tests run
   - **Queues** — unique topic/queue names per test
   - Seed through the application's own repository code or a fixtures
     loader, not hand-written SQL that silently drifts from the schema

5. **Test what only a real dependency can prove**

   Keep business-logic permutations in the unit suite; here, test the
   things that only exist at the boundary:

   - Migrations apply cleanly from empty **and** from the previous
     release's schema; the rollback path if you support one
   - Real SQL: constraint and unique violations surfacing as the error
     type your code expects, `ON CONFLICT` behavior, transaction
     isolation and rollback, `NULL` and timezone semantics, JSONB
     round-trips, index-dependent query plans on a seeded dataset
   - Connection-pool behavior: exhaustion, timeout, reconnect after the
     dependency restarts (stop and start the container mid-test — a
     genuinely valuable test that no mock can do)
   - Redis: TTL expiry, eviction under `maxmemory`, atomicity of your
     Lua/`MULTI` blocks, pipeline behavior
   - Queues: at-least-once redelivery, ordering guarantees, consumer
     group rebalance
   - The assembled slice: real HTTP handler → real repository → real DB,
     asserting the response *and* the resulting rows

6. **Make it reliable and observable in CI**

   - Separate lane/job from unit tests, with a suite timeout and
     per-test deadlines
   - Pre-pull and cache images; pin by digest so a moving `:latest`
     cannot change your results overnight
   - **On failure, dump container logs as CI artifacts** — an
     integration failure without the dependency's logs is a guessing
     game, and this is the single highest-value CI addition here
   - Zero tolerance for flakes: a test that fails 1 in 20 gets fixed or
     quarantined with a linked issue in the same commit. Verify with
     `-count=5`/repeat runs before declaring it stable
   - A labeled cleanup sweep as a post-job step:
     `docker ps -aq --filter label=<suite-label> | xargs -r docker rm -f`

7. **Report**

   > "Integration tests for `<scope>`: N tests against `<postgres:15.7,
   > redis:7.2>` via `<testcontainers|compose>` on
   > `<docker|podman rootless>`. Migrations verified from empty and from
   > `<prev release>`. Suite runs in `<duration>`, teardown verified
   > clean (0 containers left). Findings: `<issues>`."

   Defects this suite finds — a migration that only works on an empty
   DB, an error type that differs from what the mock returned — are
   exactly what it was built for. File them via `/qa-issue-report`.

**Guardrails**

- Never point integration tests at a shared, staging, or production
  dependency — ephemeral containers only. A suite that truncates tables
  is one misconfigured env var away from truncating someone's real data
- Never use fixed host ports, fixed database names, or a shared prefix —
  they break parallel CI and clobber the developer's local services
- Never `sleep` to wait for readiness; poll the dependency's own signal
  with a timeout
- Never leave containers running: teardown in `defer`/`finally`/fixture,
  plus a labeled sweep — especially with Ryuk disabled under rootless
  Podman
- Never skip the suite because the runtime is missing — "Docker/Podman
  unavailable" is a red build with a clear message, not a silent pass.
  A green suite that ran zero tests is a lie the whole team will believe
- Never use production data or real credentials in fixtures; generate
  synthetic data
- Never let image tags float (`:latest`) — pin the version production
  runs, by digest where practical
- Do not migrate unit-testable business logic into this suite because
  it is easier to write here; the lane's speed is a shared asset
- This skill writes and runs tests; it does not fix the defects it finds
  (`/engineer-implement`) or deploy anything (`/devops-deploy`)
