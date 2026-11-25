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
./mvnw verify                        # all server + sim tests (Testcontainers needs Docker)
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

## How to work on a milestone

Milestones are in spec §16 (M0 → M7). Work on **one milestone per session** unless told otherwise.

1. Re-read the milestone and the FRs it covers in `docs/SPEC.md`.
2. Propose a short plan (files, tests, any decision worth an ADR) before writing code.
3. Build test-first; keep `./mvnw verify` green after each step.
4. Finish with: all tests green, Modulith check green, `openapi.yaml` and README updated,
   and a short summary of what was done and what is left.
5. If the spec is ambiguous or seems wrong, stop and ask — do not invent behaviour.

## Out of scope (do not build)

Bulk/marketing SMS, SMPP carrier connections, sending SMS from the gateway SIM, full KYC (CNIC/NADRA),
multi-company hosted SaaS, countries other than Pakistan (but keep `NumberPolicy` open for it).
