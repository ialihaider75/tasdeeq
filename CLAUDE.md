# CLAUDE.md — Tasdeeq

Tasdeeq is an open-source, self-hosted phone-number verification service for Pakistani startups.
Its main method is **reverse OTP**: the end user *sends* a code (by SMS to an Android gateway phone,
or by WhatsApp to a business number), and the carrier- or Meta-stamped sender number proves ownership.
A paid outbound SMS OTP exists only as a fallback.

**The full specification is `docs/SPEC.md`. Read it before starting any milestone.** If this file and
the spec disagree, the spec wins; flag the conflict instead of guessing.

## Repository layout

```
tasdeeq/
├── CLAUDE.md
├── docs/
│   ├── SPEC.md              # software specification (source of truth)
│   ├── runbook.md
│   └── adr/                 # one ADR per significant decision
├── openapi.yaml             # API contract; implementation must match it
├── pom.xml                  # Maven parent for the Java modules
├── tasdeeq-server/          # Spring Boot service (Maven)
├── tasdeeq-sim/             # CLI that simulates a gateway device (Maven)
├── tasdeeq-client/          # plain Java 17 client source → Maven Central
├── tasdeeq-spring-boot-starter/ # auto-configuration source → Maven Central
├── tasdeeq-gateway-android/ # Kotlin Android app (Gradle, own wrapper — NOT a Maven module)
├── tasdeeq-web-widget/      # browser widget, plain TypeScript (npm)
├── examples/spring-boot-integration/  # sample startup backend using the starter
├── docker-compose.yml       # server, postgres, redis, caddy
└── .github/workflows/       # path-filtered: a server change doesn't rebuild the APK
```

Open the repo root in IntelliJ for the Java modules; open `tasdeeq-gateway-android/` on its own in
Android Studio.

Folders hold **source only**. Never commit jars, APKs, images or other build outputs; the release
workflow builds and publishes them.

## Publishing

- Maven group `com.intellermatrix` (already verified on Maven Central). Published artifacts:
  `tasdeeq-client` and `tasdeeq-spring-boot-starter` only. `tasdeeq-server` and `tasdeeq-sim` are
  not published (`maven.deploy.skip` / no publishing plugin).
- One version for everything, released together from one git tag `vX.Y.Z`: Maven Central (signed,
  sources + Javadoc jars), jars + SHA-256 checksums attached to the GitHub Release, server image to
  GHCR, APK to the GitHub Release, widget to npm.
- Client libraries keep backward compatibility within a major version.
## Tech stack (do not substitute without an ADR)

- Server: Java 21, Spring Boot 3.x, Spring Modulith, Spring Security, Web MVC, Spring Data JPA (Hibernate),
  PostgreSQL 16 + Flyway, Redis 7, libphonenumber, springdoc-openapi, Maven wrapper.
- Client libraries: **Java 17 baseline** (`maven.compiler.release=17`) so integrators on 17 can use them.
  `tasdeeq-client` has no Spring dependency (JDK `HttpClient` + Jackson); the starter adds
  `spring-boot-autoconfigure` and `tasdeeq.client.*` properties. Test with WireMock.
- Tests: JUnit 5, Mockito, AssertJ, Testcontainers, Cucumber, ArchUnit / Modulith verification.
- Optional events: spring-kafka, auto-configured only when `tasdeeq.events.kafka.*` is set.
- Admin page: Thymeleaf inside `tasdeeq-server`. Web widget: plain TypeScript, no framework, < 15 KB gz.
- Android: Kotlin, minSdk 26, Hilt, WorkManager, Room, DataStore, EncryptedSharedPreferences, OkHttp.
## Commands

```bash
./mvnw test                          # unit tests only (*Test), no Docker needed
./mvnw verify                        # unit + integration tests (*IT, Testcontainers needs Docker)
./mvnw -pl tasdeeq-server verify -Dit.test=VerificationApiIT   # a single IT
./mvnw -pl tasdeeq-server spring-boot:run -Dspring-boot.run.profiles=local
docker compose up -d                 # full local stack
./mvnw -pl tasdeeq-sim exec:java -Dexec.args="pair --token <t>"   # simulated gateway
cd tasdeeq-gateway-android && ./gradlew test connectedCheck
adb emu sms send 03001234567 "TSQ 482193"   # inject SMS into the emulator
```

Keep this section accurate: update it when you add or change a command.

## Architecture rules

- **Package root `com.intellermatrix.tasdeeq`** in every Java module (client:
  `com.intellermatrix.tasdeeq.client`, starter: `com.intellermatrix.tasdeeq.autoconfigure`).
- **Modular monolith.** Server packages under `com.intellermatrix.tasdeeq.server.<module>`: `verification`, `channel`, `gateway`,
  `whatsapp`, `outbound`, `webhook`, `tenant`, `ratelimit`, `alerting`, `shared`.
- Modules communicate **only** via application events or the types in a module's root (API) package.
  Never import another module's `internal` package. `ModularityTests` (`ApplicationModules.verify()`)
  must stay green.
- `verification` is the only module that changes a verification's state.
- **Outbox → inbox at every hop.** Gateway: save to Room first, forward later, delete only on a
  per-message `ACCEPTED`/`DUPLICATE`/`REJECTED` ack (spec §8.3). Server: commit `inbound_message`
  (unique device + message ID) before acking; match codes *after commit* via Spring Modulith's event
  publication registry, never inside the inbound request. Webhooks: `webhook_delivery` is an outbox.
- **Event-sourced status.** Every change is appended to `verification_event`; `verification` is a
  projection. Never update status without appending an event in the same transaction.
- Extension points are SPIs: `VerificationChannel`, `OutboundSmsProvider`, `EventPublisher`
  (webhook default, Kafka optional — same event JSON), `AlertSink`, `NumberPolicy`.
  New providers or channels are new beans selected by config — never `if (provider == "x")` branches.
- Integrators' apps never call Tasdeeq; only their backend does (API key server-side). The API returns
  ready-made deep links (`links` always, `link` when `platform` is sent) — clients must never build links.
- `tasdeeq-web-widget` holds no API key and polls the integrator's own backend, never Tasdeeq.
- Nothing operator-specific is hard-coded (brand, numbers, provider, prefix). Everything goes through
  `tasdeeq.*` config (spec §12.1).
## Coding conventions

- TDD: write the failing test first, then the code. Every FR-x in the spec needs at least one test;
  name the test or scenario after it (e.g. `FR6_rejectsMismatchedSender`).
- Records for DTOs and events; constructor injection; no field injection; no Lombok.
- IDs: prefixed ULIDs (`ver_`, `app_`, `dev_`, `evt_`). Time: `Instant` in UTC, inject `Clock`.
- Phone numbers: always normalise through `shared.PhoneNumbers` to E.164 before compare/store.
  Never infer carrier from prefix (number portability).
- Errors: RFC 9457 problem details with the stable `code` values in spec §8.6.
- Flyway migrations are append-only; never edit a released migration.
- Update `openapi.yaml` in the same change as any API change; the contract test enforces it.
## Testing conventions (Java modules)

Every behaviour gets **both** unit tests and, where it touches Spring wiring, the database, Redis or
HTTP, an integration test.

| Kind | Class name | Runs with | Uses |
| --- | --- | --- | --- |
| Unit | `*Test` (e.g. `PhoneNumbersTest`) | Surefire, `./mvnw test` | JUnit 5, Mockito, AssertJ; no Spring context, no containers |
| Integration | `*IT` (e.g. `VerificationApiIT`) | Failsafe, `./mvnw verify` | **Must extend `IntegrationTestSupport`**; real Postgres + Redis via Testcontainers |

- Configure `maven-surefire-plugin` for `**/*Test.java` and `maven-failsafe-plugin` for `**/*IT.java`
  (goals `integration-test` + `verify`) in the parent `pom.xml`. `./mvnw verify` must run both.
- `IntegrationTestSupport` lives in `tasdeeq-server/src/test/java/com/intellermatrix/tasdeeq/server/support/`.
  Never start containers in individual IT classes; add any new container to the base class.
- ITs call the API through **RestAssured** (black-box, random port) or **MockMvc** (when inspecting
  Spring MVC details), and check state through `DbOperations`, not through production repositories.
- Each IT starts from a clean state: `@BeforeEach` truncates all tables (except Flyway history) and
  flushes Redis.
- BDD (Cucumber) scenarios run as ITs on top of the same base class.
- Client libraries (`tasdeeq-client`, starter) have no server: their ITs use WireMock and do not
  extend `IntegrationTestSupport`.
- Use `Clock` injection for time-dependent tests; never `Thread.sleep` (use Awaitility for async).
  Base class template (follow this structure):

```java
package com.intellermatrix.tasdeeq.server.support;
 
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Import(TestAsyncConfig.class)                 // runs async/event listeners synchronously in tests
public abstract class IntegrationTestSupport {
 
    // Started once per JVM and shared by every IT class (singleton container pattern).
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("tasdeeq").withUsername("tasdeeq").withPassword("tasdeeq");
 
    static final GenericContainer<?> REDIS =
            new GenericContainer<>("redis:7-alpine").withExposedPorts(6379)
                    .waitingFor(Wait.forListeningPort());
 
    static {
        POSTGRES.start();
        REDIS.start();
    }
 
    @DynamicPropertySource
    static void containerProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.data.redis.host", REDIS::getHost);
        registry.add("spring.data.redis.port", () -> REDIS.getMappedPort(6379));
    }
 
    @Autowired protected MockMvc mockMvc;
    @Autowired protected DbOperations dbOperations;   // test helper: truncate tables, flush Redis, seed data
    @LocalServerPort protected int port;
 
    @BeforeEach
    void resetState() {
        RestAssured.baseURI = "http://localhost";
        RestAssured.port = port;
        dbOperations.clearDatabase();
        dbOperations.flushRedis();
    }
}
```

`DbOperations` template (same package; a test-only `@Component`, no Lombok):

```java
@Component
public class DbOperations {
 
    private final JdbcTemplate jdbc;
    private final StringRedisTemplate redis;
 
    public DbOperations(JdbcTemplate jdbc, StringRedisTemplate redis) {
        this.jdbc = jdbc;
        this.redis = redis;
    }
 
    /** Empties every application table (incl. event_publication); keeps Flyway history. */
    public void clearDatabase() {
        List<String> tables = jdbc.queryForList("""
                SELECT tablename FROM pg_tables
                WHERE schemaname = 'public' AND tablename <> 'flyway_schema_history'
                """, String.class);
        if (!tables.isEmpty()) {
            jdbc.execute("TRUNCATE TABLE " + String.join(", ", tables) + " RESTART IDENTITY CASCADE");
        }
    }
 
    public void flushRedis() {
        redis.execute((RedisCallback<Void>) connection -> {
            connection.serverCommands().flushDb();
            return null;
        });
    }
 
    // Seed and read helpers for ITs go here, written as plain SQL, e.g.
    // insertClientApp(...), countVerificationEvents(String verificationId).
}
```

- Never inject production repositories or entities into `DbOperations`. Use SQL, so test setup
  doesn't depend on any module's internals.
- New tables need no change here: they are found from `pg_tables`.
  Rules for the base class:
- Pin image versions (`postgres:16-alpine`, `redis:7-alpine`), never `latest`.
- Use readiness waits (`PostgreSQLContainer`'s built-in wait, `Wait.forListeningPort()`), not
  `IsRunningStartupCheckStrategy`, which only checks the container is running, not ready.
- No `@DirtiesContext`: one shared Spring context across all ITs keeps the suite fast. Use it only
  on a specific IT that truly changes context state, with a comment explaining why.
- Test credentials are dummy values defined only in the test code; never real passwords.
- When WhatsApp, Kafka or the fallback provider are tested, add their containers or WireMock stubs
  here (e.g. a `KafkaContainer`, started lazily only when needed).
## Security and privacy — non-negotiable

- The gateway app forwards **only** SMS on the bound SIM whose body matches the configured pattern.
  Non-matching messages are never stored, logged or transmitted.
- The gateway **never sends SMS**. Outbound SMS only via an `OutboundSmsProvider`.
- Never reply on WhatsApp (service replies are billable since Oct 2026).
- Never log full phone numbers, codes, message bodies, API keys or secrets. Use `PhoneNumbers.mask()`.
- API keys and device credentials are stored hashed (SHA-256), shown once. Codes are stored
  AES-GCM encrypted in Postgres (`TSQ_CODE_KEY`), plain only in Redis with TTL.
- Verify Meta's `X-Hub-Signature-256` before parsing WhatsApp webhooks. Sign outbound webhooks
  with HMAC-SHA256 (`Tasdeeq-Signature: t=…,v1=…`).
- No secrets in the repo. Use `.env` (git-ignored) and provide `.env.example`.
- `.gitignore` must include `local/`, `.env`, `target/`, `build/`, `node_modules/` and IDE folders.
## Development workflow (mandatory for every milestone)

Milestones are in spec §16 (M0 → M7), done strictly in order. This workflow uses the **Superpowers**
skills. Every step marked **⏸ STOP** means: stop, report, and wait for the user's explicit instruction.
Never skip a gate, never combine gates, never assume approval.

### Where documents live

- Design docs: `local/superpowers/specs/<milestone>-design.md` (e.g. `M0-design.md`)
- Plans: `local/superpowers/plans/<milestone>-plan.md`
- `local/` is git-ignored: these are working documents, never committed, never pushed.
  This overrides the Superpowers default location (`docs/superpowers/...`).
### Phase 1 — Design (once per milestone)

1. Read the milestone and the FRs it covers in `docs/SPEC.md`.
2. Use the `superpowers:brainstorming` skill to produce the milestone design doc. It must stay within
   the spec; anything the spec leaves open is listed as a question, not decided silently.
3. **⏸ STOP — design review.** The user reviews and may request changes. Apply them and stop again,
   until the user explicitly approves the design.
### Phase 2 — Plan (once per milestone)

4. Use the `superpowers:writing-plans` skill to split the approved design into **tasks**, each with
   checkbox **steps**. Each task = one feature branch = one PR, small enough to review in one sitting.
   Give every task an ID and a kebab-case name, e.g. `M0-T1 maven-parent`.
5. **⏸ STOP — plan review.** Apply requested changes until the user explicitly approves the plan.
### Phase 3 — Per task (repeat for each task in plan order)

6. Sync and branch: `git checkout main && git pull`, then
   `git checkout -b feature/<task-id>-<task-name>` (e.g. `feature/m0-t1-maven-parent`, lowercase).
7. Implement with the `superpowers:test-driven-development` skill: red → green → refactor for every
   step. Run `./mvnw verify` (and Gradle/npm where relevant) before reporting.
8. **⏸ STOP — code review 1.** When the task's steps are green, summarise the changes and the test
   results. Wait for the user's review and approval.
9. **Commit only when the user explicitly says "commit".** Conventional message, e.g.
   `feat(gateway): add device pairing endpoint [M2-T3]`, with the attribution lines required by the session.
10. **⏸ STOP** after committing. Wait for the user to ask for the independent review.
11. When asked, use the `superpowers:requesting-code-review` skill to launch an **independent code
    reviewer subagent** on the branch's diff against `main`. Report its findings in full.
12. Fix the findings using `superpowers:receiving-code-review` (verify each finding; push back with
    reasons on any you believe is wrong instead of blindly applying it). Keep tests green.
13. **⏸ STOP — code review 2.** Wait for the user's review. Commit only on an explicit "commit".
14. **Push only when the user explicitly says "push"**: `git push -u origin <branch>`.
    Never open, merge or close PRs; the user does that on GitHub.
15. **⏸ STOP** until the user says the PR is merged.
16. Then: `git checkout main && git pull`, delete the local feature branch, and update
    `local/superpowers/plans/<milestone>-plan.md`, ticking the task and all of its steps as complete.
17. Move to the next task (back to step 6). After the last task, report that the milestone is
    complete and **⏸ STOP** before starting the next milestone's Phase 1.
### Always

- Never `git commit`, `git push`, merge, rebase onto main, force-push or delete remote branches
  without an explicit instruction for that exact action.
- One task per branch; never mix work from two tasks in one branch.
- Each task's changes also update, where relevant: `openapi.yaml`, README, `docs/runbook.md`,
  `docs/integration-guide.md`, and an ADR in `docs/adr/` for any significant decision.
- If the spec is ambiguous or seems wrong, stop and ask. Do not invent behaviour.
## Out of scope (do not build)

Bulk/marketing SMS, SMPP carrier connections, sending SMS from the gateway SIM, full KYC (CNIC/NADRA),
multi-company hosted SaaS, countries other than Pakistan (but keep `NumberPolicy` open for it).
