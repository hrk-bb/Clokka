# Clokka Test Plan (Phase 4 — levels and traceability)

| Item | Value |
| --- | --- |
| Purpose | Define what to test, at which level, and how to trace a requirement to a passing test without inventing implementation details. |
| Audience | Developers, AI agents, QA, Product Owner |
| Status | REVIEW — awaiting Q-04 |
| Last updated | 2026-08-28 |
| Depends on | `01_requirements.md`, `04_database.md`, `05_api.md`, `06_screen-design.md`, `07_security.md`, `08_directory.md`, `09_development-rules.md` |

## 1. Test levels

| Level | Scope | Technology | When it runs | Who owns |
| --- | --- | --- | --- | --- |
| Unit | Single class / function (e.g., JST work-date calc, break validation) | JUnit 5, Mockito | `./gradlew test` (CI and local) | Developer |
| Integration (slice) | Spring slice + real PostgreSQL via Testcontainers | Spring Boot Test, Testcontainers `postgres:16`, Flyway migrations, RestAssured/MockMvc | `./gradlew test` (CI) | Developer |
| API (contract) | Full `backend` against `05_api.md` contracts (auth, attendance, submission, admin, push, deadlines, reminders, audit) | Testcontainers, MockMvc, AssertJ | `./gradlew test` and nightly CI | Developer |
| E2E (browser) | `frontend` + `backend` + DB via Playwright | Playwright (Phase 9) | `npx playwright test` (Phase 9, not Phase 6) | QA / AI |
| Security / load | Auth bypass, CSRF, rate-limit, 50×31 Excel | OWASP ZAP (optional), JMeter/k6 (Phase 9) | Phase 9 | QA |

* No manual `curl` as a substitute for an automated test. Every FR must have at least one automated test at the appropriate level.
* `frontend` unit tests (Vitest/Jest) are **not used** in the MVP — Vanilla JS is tested via integration/API and E2E.

## 2. Traceability matrix (FR → test)

| Requirement | Unit | Integration | API | E2E (Phase 9) |
| --- | --- | --- | --- | --- |
| FR-01 Auth/roles | Argon2id vector, session cookie flags | `POST /auth/login` success/failure, `GET /me` roles | `401`/`403` matrix, CSRF, IDOR | Login S-01, 401 redirect, 403 page |
| FR-02，日次入力 | JST `work_date` calc, `clock_out - clock_in < 24h`, `break < elapsed` | `PUT /attendance/{workDate}` upsert, `UNIQUE(employee_id,work_date)` | `409` when `status=SUBMITTED` | S-03 save/delete, validation messages |
| FR-03 日跨ぎ | `23:00→02:00` maps to start date | Same via DB `CHECK ck_attendance_work_date_jst` | API `clockOut <= clockIn` treated as next day | S-03 next-day hint, S-02 monthly view |
| FR-04 月次表示 | `target_month = date_trunc` | `GET /attendance?month=` returns target days, `DRAFT` display | `GET /submissions/{month}` | S-02 month navigation, totals |
| FR-05 提出前チェック | Calendar-derived required days | `POST /submissions/{month}/validate` returns `422` with `fieldErrors` | `422` body | S-04 validation list, links to S-03 |
| FR-06 提出/差戻し | Status FSM `DRAFT→SUBMITTED→RETURNED→SUBMITTED` | `POST /submissions`, `POST /admin/.../return` with audit | `409` on concurrent transition | S-04 submit, S-06 return, S-02 lock |
| FR-07 通知漏れ防止 | `reminder_stage` calc from `due_at` | `notification_deliveries` `UNIQUE(employee_id,notification_date)` | `POST /internal/jobs/monthly-reminders` idempotency | S-05 Push state, S-07 list, in-app banner |
| FR-08 管理一覧 | Sort comparator | `GET /admin/submissions` filters, sorting, pagination | `X-Request-Id` present | S-06 search/sort |
| FR-12 Push状態 | Aggregation `GRANTED>DEFAULT>DENIED>UNSUPPORTED>UNKNOWN` | `push_subscriptions` encrypt/decrypt, `PUT /push-subscriptions/status` | `GET /admin/push-status?status=DENIED` | S-07 filtered view |
| FR-09 Excel | `workMinutes = elapsed - break` | `GET /admin/exports/attendance.xlsx` streams XLSX, audit logged | `Content-Disposition` + audit | S-06 Excel download |
| FR-10 管理設定 | `is_active` toggle | `POST/PATCH /admin/employees`, `.../calendar`, `.../deadlines/{month}` | `PUT dueAt +09:00` handling | S-08, S-09, S-10 |
| FR-11 監査 | `actor_type` CHECK | `audit_logs` trigger rejects UPDATE/DELETE, `GRANT` | All mutating endpoints insert audit | Audit log query (internal) |
| NFR-02 性能 | — | — | 50×31 Excel < 30s (API test with timer) | Playwright 360px + Chrome/Edge/Safari |

## 3. Database contract tests (must pass before any feature)

* Flyway `V1`–`V3` apply on a fresh `postgres:16` via Testcontainers.
* `CHECK` constraints from `04_database.md:4` are exercised: `ck_attendance_work_date_jst` with JST boundary (`23:00 JST = 14:00 UTC`), `ck_submission_status_fields`, `ck_notification_*`, `ck_push_payload_state`, and the `audit_logs` trigger.
* Roles: `clokka_app` can `SELECT, INSERT` on `audit_logs` but `UPDATE`/`DELETE` is rejected by the trigger; `PUBLIC` has no privileges.

## 4. Push encryption tests

* A subscription JSON is encrypted with `AES-GCM` (12-byte IV) and decrypted with the same `key_version`; decryption with a wrong key fails with `AEADBadTagException`.
* `DELETE /push-subscriptions/{id}` clears `ciphertext/iv/version` and sets `revoked_at`; `GET /admin/push-status` never returns plaintext endpoint/keys.

## 5. API test conventions

* Base URL `http://localhost:8080/api/v1` (Testcontainers) with `HttpOnly` session cookie + `X-CSRF-Token` header.
* Every test asserts `X-Request-Id` is present on 2xx and 4xx.
* `401` vs `403` vs `404` are tested via the matrix in `07_security.md:2` (IDOR, disabled admin, CSRF).
* `notification_deliveries` at-most-once is tested by firing `POST /internal/jobs/monthly-reminders` twice with the same `notification_date` and asserting the second run inserts zero `RESERVED` rows.

## 6. Frontend / E2E (Phase 9)

* Playwright covers S-01 → S-02/S-06, S-03 save, S-04 submit, S-06 return, and 360px responsive per `06_screen-design.md:6` and `NFR-01`.
* No `frontend` unit tests in the MVP; E2E plus API coverage is sufficient for 50 users.

## 7. CI quality gates (added in Phase 6)

* `ci.yml` runs `./gradlew check` (Spotless + test + Testcontainers) and a secret scan (e.g., `gitleaks` or `trufflehog`). A PR is not mergeable if any gate fails.
* Test reports and coverage (JaCoCo, optional) are uploaded as artifacts.

## 8. What is not tested in Phase 4

* No code is written; this plan is a contract for Phase 6–9. No `V{NN}__` files, no `*Test.java` are created in Phase 4.
* Load and security scans are Phase 9 only, not Phase 6.
