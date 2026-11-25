# 18 · Backend: make the Kotlin test suite faster and parallel

> **Solves:** Cuts the backend test run (~3.5 min on CI and longer locally) toward ~1 min by resetting the database through JDBC instead of `docker exec psql`, reusing Spring contexts, and running test classes in parallel forks, each with its own database.

| Field | Value |
|---|---|
| Issue | none yet; open one with the measurements from step 0 |
| Component | backend (tests), CI |
| PR titles | `test(backend): reset test database via JDBC instead of docker exec`, `test(backend): reduce distinct Spring test contexts`, `test(backend): run test classes in parallel forks with per-fork databases` |
| Effort | M (3 incremental PRs) |
| Risk | Low–medium: test-only changes, but parallelism can expose hidden coupling between tests |
| Status | todo · 🟢 free (no related issue or PR found) |

## Current state (verified 2026-11-24)
- **CI:** `backend-tests` → job `test` → step **"Run tests" = 210 s** on the latest green `main` run. Over ~90 days, the workflow had a median of 4.3 min and a p90 of 6.0 min.
- **Suite size:** ~81 test files and roughly 1,000+ test methods (rough count).
- **Everything runs serially in one JVM:**
  - `build.gradle` `tasks.named('test')` sets no `maxParallelForks`, and JUnit parallel execution isn't enabled;
  - one Postgres and one MinIO (Testcontainers) are shared by the whole run (`EndpointTestExtension.kt`, `TestInfrastructure`).
- **The database reset runs before every test, through Docker:**
  - `EndpointTestExtension.beforeEach` calls `env.postgres.exec(clearDatabaseStatement())`;
  - `DockerPostgres.exec` (`testutil/docker/DockerPostgres.kt:29`) uses `container.execInContainer("psql", …, "-c", "truncate table …")`, which spawns a **`docker exec` + `psql` process per test**. That's typically 100–300 ms each, so with ~1,000 tests it can account for **minutes** (measure in step 0).
- **Many Spring contexts:**
  - **12 test files use `@MockkBean`** (each distinct set of mocked beans creates a new application context);
  - ~27 files set custom properties or config paths (`@EndpointTest(properties = …)`: S3 config, debug mode, data-use-terms-disabled, single-segment config…);
  - each distinct context means a new Spring Boot start-up plus Flyway checks.
- **The 210 s include compilation:** CI runs a single `./gradlew test` (`.github/workflows/backend-tests.yml:42`), so the step covers `compileKotlin` + `compileTestKotlin` + test execution. `backend/gradle.properties` only sets `org.gradle.jvmargs=-Xshare:off`: **no `org.gradle.caching`, no `org.gradle.parallel`, no configuration cache**. `setup-gradle` restores dependencies, but compiled output isn't reused between runs.
- **The truncate list is hard-coded** in `clearDatabaseStatement()`. Every new table must be added by hand, or tests leak data into each other.
- **JVM flags:** `-Xshare:off` (a workaround for gradle#19989) disables class data sharing, which makes every context JVM start a bit slower.

## Plan

### Step 0: measure (½ day)
- Split **compile vs. test time** on CI: run `./gradlew compileTestKotlin` as its own step before `./gradlew test`, or read it from a build scan. If compiling is a large share, step 4 (build cache) comes first.
- Context count: set `logging.level.org.springframework.test.context.cache=DEBUG` in `src/test/resources/logback-test.xml`/properties. At the end of the run Spring logs *"Spring test ApplicationContext cache statistics: size = N, maxSize = 32, hitCount = …, missCount = …"*. `missCount` is the number of contexts built.
1. Run `./gradlew test --scan` (or with `--profile`) locally and on CI. Get per-class durations from `build/test-results/test/*.xml`.
2. Time `clearDatabaseStatement()` with a stopwatch log, then multiply by the number of tests.
3. Count the Spring contexts that get created: enable `logging.level.org.springframework.test.context.cache=DEBUG` and use the context-cache statistics logged at the end of the run.

### Step 1: reset the database through JDBC (quick win)
- In `EndpointTestExtension.beforeEach`, run the `TRUNCATE … RESTART IDENTITY CASCADE` through a JDBC connection that is kept open (or through the Spring `DataSource`), instead of `execInContainer`.
- Keep `LocalPostgres` (`USE_NONDOCKER_INFRA`) working.
- While at it, **build the truncate list dynamically** (`select tablename from pg_tables where schemaname = 'public' and tablename not in ('flyway_schema_history', 'compression_dictionaries', …)`), so new tables are cleaned automatically.
- **Expected:** removes one `docker exec` per test; likely the biggest single gain.

### Step 2: fewer distinct Spring contexts
- Replace per-class `@MockkBean` sets with a few shared test configurations, for example one `@TestConfiguration` providing mocks for the commonly mocked beans, or `@MockkBean` declared on shared base annotations. This lets Spring reuse cached contexts.
- Group tests by config variant (S3, debug, single-segment, data-use-terms-disabled) and keep the number of variants small.
- Make sure the context cache is big enough (`spring.test.context.cache.maxSize`, default 32), so contexts aren't evicted and rebuilt.

### Step 3: parallel forks with one database per fork
- `build.gradle`: `maxParallelForks = (Runtime.runtime.availableProcessors() / 2).clamp(1, 4)` (try 2–4 on CI).
- Keep **one Postgres container**, but give each fork its own database. Use `org.gradle.test.worker` to name it (`loculus_test_${worker}`), created with `CREATE DATABASE … TEMPLATE loculus_template` after Flyway migrates the template once. That is fast and gives full isolation.
- MinIO: one bucket per fork (`testbucket-${worker}`) or a per-fork key prefix.
- Run Flyway **once** on the template database. Each fork's database is a copy, so the Spring contexts in that fork only validate the migrations and don't run them.
- Use `System.getProperty("org.gradle.test.worker")` as the fork ID. `TestInfrastructure` already sets S3 system properties per JVM, which is safe with forks because each fork is its own JVM.
- Alternative to forks: JUnit 5 class-level parallelism within one JVM. It needs per-thread databases and is riskier with Spring's static state, so forks are preferred.

### Step 4: Gradle and CI tuning (can go first if compiling dominates)
- `backend/gradle.properties`: add `org.gradle.caching=true` (local and remote/Actions cache via `setup-gradle`), `org.gradle.parallel=true`, and try `org.gradle.configuration-cache=true` (check that the custom `doFirst` in the test task is compatible).
- Locally: enable Testcontainers reuse (`testcontainers.reuse.enable=true` in `~/.testcontainers.properties` plus `.withReuse(true)`), so repeated runs don't start Postgres and MinIO from scratch.
- If needed, shard test classes across 2 runners (`--tests` filters by package, or a test-distribution plugin).

## Acceptance criteria
- Before/after numbers for each step in the PR descriptions (local and CI).
- CI "Run tests" step at **≤ 90 s** (from 210 s), with no new flakiness over 2 weeks. Compile and test time are reported separately.
- The truncate list is derived from the schema (no hand-maintained list).
- `USE_NONDOCKER_INFRA=true ./gradlew test` still works (see `backend/AGENTS.md`).

## Files
- `backend/build.gradle` (test task)
- `backend/src/test/kotlin/org/loculus/backend/testutil/EndpointTestExtension.kt`, `TestEnvironment.kt`, `docker/DockerPostgres.kt`, `binary/LocalPostgres.kt`
- Test classes using `@MockkBean` (12 files) and `@EndpointTest(properties = …)`
- `.github/workflows/backend-tests.yml`, `backend/gradle.properties`
- `backend/src/test/resources/logback-test.xml` (context-cache logging)
