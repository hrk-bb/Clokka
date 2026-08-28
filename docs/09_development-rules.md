# Clokka Development Rules (Phase 4)

| Item | Value |
| --- | --- |
| Purpose | Define how AI and human developers write, review, and test code consistently with the approved Phase 3 contracts. |
| Audience | Developers, AI agents, reviewers |
| Status | REVIEW — awaiting Q-04 |
| Last updated | 2026-08-28 |
| Depends on | `08_directory.md`, `01_requirements.md`–`07_security.md`, `03_tech-stack.md` |

## 1. General principles

1. **Single Source of Truth is Git.** No design or requirement lives only in chat or a local file.
2. **Approval gate is mandatory:** `design → review → PO approval → implementation → review → fix → test → commit → next`. Do not start Phase 6 code before Q-04.
3. **No silent contract changes.** If a requirement, ADR, API, or DDL invariant must change, file a `Q-xx` and update `decisions.md` with an ADR.
4. **Small, single-purpose commits.** One fix or one feature per commit; no mixed doc+code commits except for `chore: normalize structure` style.
5. **No secrets in Git.** See `07_security.md:4` and `.gitignore` rules below.

## 2. Languages and tooling

| Area | Rule |
| --- | --- |
| Java | **Java 21 LTS** (`eclipse-temurin:21`). No preview features. |
| Build | Gradle Wrapper (`./gradlew`). No Maven. `gradle-wrapper.jar` is committed. |
| Backend | Spring Boot 3.x (latest 3.2.x at Phase 6 start, pinned in `backend/gradle/libs.versions.toml` or `build.gradle`). |
| Frontend | Vanilla JS (ES Modules), HTML, CSS. No React/Vue, no bundler, no `node_modules` in the MVP. |
| DB migrations | Flyway OSS, SQL files only: `backend/src/main/resources/db/migration/V{NN}__{description}.sql`. |
| Formatting | Java: Google Java Format via Spotless (or `gradle spotlessCheck`). JS/CSS: no formatter in MVP; keep `editorconfig` simple if added. |
| Lint / static analysis | Backend: SpotBugs or ErrorProne (one, not both) plus Checkstyle for imports. Frontend: manual review only in MVP. |
| Editor | `.editorconfig` with `charset=utf-8`, `end_of_line=lf`, `insert_final_newline=true`, `trim_trailing_whitespace=true`. |

## 3. Backend rules

* **Package structure** follows `08_directory.md:2` ( `identity`, `attendance`, `submission`, `admin`, `notification`, `audit`, `config` ). No new top-level packages without ADR.
* **Controllers** are thin: validate, call service, return DTO. Business rules live in `Service` classes, not controllers.
* **Validation**
  * Request DTOs use `jakarta.validation` (`@NotBlank`, `@Pattern`, etc.) plus explicit JST work-date checks.
  * Database invariants from `04_database.md:4` (`CHECK` constraints) are **not duplicated** as application `CHECK` SQL; they are enforced by the DB and covered by integration tests.
* **Transactions:** Use `@Transactional` at the service layer. Attendance write + audit + `DRAFT` creation (for `PUT /attendance`) and submission + audit are single transactions. Notification reservation is a separate transaction committed before Push calls (at-most-once per `04:5`).
* **Time:** All persisted instants are `Instant`/`TIMESTAMPTZ`. Business dates are `LocalDate` interpreted as `Asia/Tokyo`. Never use `LocalDateTime` for storage. Convert via `clock_in_at.atZone(ZoneId.of("Asia/Tokyo")).toLocalDate()`.
* **Error handling:** Use `ResponseStatusException` or a small `@ControllerAdvice` that returns `{code, message, fieldErrors}` per `05_api.md:1` and never leaks secrets or stack traces.
* **Logging:** Log `X-Request-Id` on every request. Never log `password`, `subscription_ciphertext`, `subscription_iv`, `PUSH_SUBSCRIPTION_ENCRYPTION_KEY`, or `VAPID` private key. Push endpoint is never logged at `INFO`.

## 4. Frontend rules

* **No framework.** Use `fetch` + `async/await`, ES Modules, and plain DOM. Keep `js/api.js` as the single `fetch` wrapper that adds `X-Requested-With`/`X-CSRF-Token` and handles `401` → redirect to `S-01`.
* **CSRF:** On app start and before every mutating call, `GET /api/v1/csrf` and send `X-CSRF-Token`. Store the token in memory, not in `localStorage` beyond the session (the token itself is not a secret but must not be cached across logouts).
* **JST display:** The API returns UTC; the UI formats with `Intl.DateTimeFormat("ja-JP", {timeZone:"Asia/Tokyo"})`. Never use the browser's local time zone for `work_date` or `due_at`.
* **Accessibility:** Follow `06_screen-design.md:6` (360px, labels, focus, 44px targets, color + text).
* **State:** No global store. Each screen fetches its own data via `GET /me`, `GET /attendance?month=`, etc., and re-fetches after mutations.

## 5. Database and migrations

* **Migrations are SQL only** and once merged to `main` are immutable. Never edit a migrated `V1__` file; add a new `V{NN}__`.
* **First migrations (Phase 6 skeleton):**
  * `V1__extensions_and_roles.sql` — `pgcrypto` for `gen_random_uuid()`, roles `clokka_owner` / `clokka_app`.
  * `V2__core_tables.sql` — `departments`, `employees`, `attendance_records`, `monthly_submissions`, `company_calendar`, `submission_deadlines`, `push_subscriptions`, `notification_deliveries`, `notification_delivery_attempts`, `audit_logs` plus all `CHECK`, `UNIQUE`, and `FK ON DELETE RESTRICT`.
  * `V3__indexes_and_triggers.sql` — indexes from `04:7` and the `reject_audit_log_mutation` trigger.
* **`application.yml` (checked in) must not contain secrets.** Use `${DATABASE_URL}`, `${PUSH_SUBSCRIPTION_ENCRYPTION_KEY}`, `${VAPID_PRIVATE_KEY}` etc., injected via Render env or local `.env`. `.env` is gitignored.

## 6. Security rules (from `07_security.md`)

* Passwords: Argon2id (`m=19MiB`, `t=2`, `p=1` minimum, tuned at Phase 6) via the approved library; verify with a single test vector.
* Sessions: `HttpOnly; Secure; SameSite=Lax`, `SESSION_COOKIE` name, `server.servlet.session.cookie.secure=true`, session fixation protection (`migrateSession`), logout invalidates server session.
* Rate limiting: login and `POST /internal/jobs/monthly-reminders` are rate-limited (in-memory for MVP, not DB).
* `audit_logs` trigger and `GRANT` per `04:7` / `07:4` are mandatory before any audit-relevant feature is merged.
* Push subscription `subscription_ciphertext`/`iv`/`key_version` handling per `07:5` is mandatory before any Push feature is merged.

## 7. Git hygiene

* See `10_branch-strategy.md` for branches, commits, and PRs.
* Commit message prefix: `feat:`, `fix:`, `docs:`, ` chore:`, `test:`, `refactor:` (conventional, lower-case).
* Every commit must pass `./gradlew check` locally before push (Spotless + tests).

## 8. Definition of Done for a Phase 6 task

A task is done only when: code follows this file and `08_directory.md`, Flyway migrations apply on a fresh DB, unit + integration tests pass, no secret is committed, `X-Request-Id` is present on the changed endpoints, and the related `known-issues.md` entry is updated.

## 9. What is not allowed in Phase 4

* No Java/JS code, no `Dockerfile`, no `docker-compose.yml` beyond the design, no `.env` with real values.
* No changing of `01`–`07` contracts without a new `Q-xx` and ADR.
