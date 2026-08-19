# Clokka Decision Record

| Item | Value |
| --- | --- |
| Purpose | Preserve evidence-backed architecture and product decisions for future agents. |
| Audience | Product Owner, AI agents, developers, reviewers. |
| Update timing | When an approved decision is created, changed, superseded, or contradicted. |
| Audit basis | Existing primary documents inspected on 2026-08-19. |

## ADR-001 — Work-date basis for overnight attendance

**Decision:** Attendance belongs to the JST calendar date of clock-in. A clock-out time not later than clock-in is interpreted as the next day; month-end overnight work belongs to the clock-in month.

**Status:** APPROVED

**Date:** Recorded in `D-01`; exact approval timestamp is not available from repository evidence.

**Reason:** Explicit Product Owner decision captured in primary requirements.

**Alternatives:** Clock-out-date basis; splitting attendance across dates. Neither is selected.

**Consequences:** `work_date` must match JST clock-in date; validation and month submission use start month.

**Evidence:** `../01_requirements.md` D-01 and overnight-work rules; `../04_database.md`.

## ADR-002 — Initial leave scope

**Decision:** Do not implement paid leave, absence, compensatory leave, or related request/approval flow in the initial release.

**Status:** APPROVED

**Date:** Recorded in `D-03`; exact approval timestamp unavailable from repository evidence.

**Reason:** Product Owner direction; preserve simple initial scope.

**Alternatives:** Include leave display/requests/approval in MVP. Not selected.

**Consequences:** Missing attendance is not excused by leave in initial validation; future `leave` module is only a design idea.

**Evidence:** `../01_requirements.md` D-03; `../04_database.md` submission validation.

## ADR-003 — MVP architecture and hosting direction

**Decision:** Use a modular Spring Boot application with same-origin Vanilla JavaScript UI, hosted as a Docker-based Render Free Web Service and connected to Neon Free PostgreSQL. GitHub-linked `main` deployment and GitHub Actions scheduled reminder invocation are intended. AWS is a later production option; OCI is not an MVP option.

**Status:** APPROVED for MVP direction; not implemented.

**Date:** Phase 2 approval is recorded in roadmap; exact approval timestamp unavailable from repository evidence.

**Reason:** Maintainability and simplicity; Docker portability to AWS; avoid OCI account requirements; Render supports Docker/GitHub deployment; Neon avoids Render Free Postgres 30-day expiry.

**Alternatives:** OCI Always Free (superseded/prohibited for MVP), Fly.io (not selected due to card requirement), Railway (not selected due to limited continuous free capacity), Cloudflare Workers/D1 (not selected due to Spring Boot migration cost), AWS/GCP (not MVP).

**Consequences:** Render Free idle shutdown is accepted for MVP; scheduled notification requires external GitHub Actions trigger; no cloud-specific application API should be introduced. Production deployment is not designed/approved yet.

**Evidence:** `../02_architecture.md`, `../03_tech-stack.md`, `../00_プロジェクトロードマップ.md` Phase 2. Note conflicting architecture wording in `known-issues.md`.

## ADR-004 — Authentication and password storage

**Decision:** Use employee ID/password authentication with Spring Security server-side sessions and Argon2id password hashing.

**Status:** APPROVED as Phase 2 design; not implemented.

**Date:** Exact approval timestamp unavailable from repository evidence.

**Reason:** Small internal user population; same-origin `HttpOnly` session cookie avoids browser token storage. Argon2id is documented as selected security practice.

**Alternatives:** OAuth/OIDC, Keycloak, JWT, bcrypt.

**Consequences:** Need CSRF controls, session fixation protection, rate limits, secure cookie settings, administrator-led reset process, and an Argon2 implementation verification in Phase 6/7.

**Evidence:** `../03_tech-stack.md`, `../05_api.md`, `../07_security.md`.

## ADR-005 — Web Push and administrator status visibility

**Decision:** Use Web Push plus in-app warnings; show aggregated Push state in the admin UI. No external contact feature for users who deny/disable notifications in the initial release.

**Status:** APPROVED in principle; aggregation algorithm requires reconciliation before implementation.

**Date:** `D-04` recorded; exact approval timestamp unavailable from repository evidence.

**Reason:** Support the core missed-submission objective without SMS/paid email. Respect browser permission and privacy constraints.

**Alternatives:** Email, SMS, external contact workflow. Not selected for initial scope.

**Consequences:** Browser-only notifications cannot guarantee reachability; administrator status list is the fallback. A per-browser random `installation_id` is designed instead of device fingerprinting.

**Evidence:** `../01_requirements.md` D-04 and Push sections; `../04_database.md`; `../05_api.md`; conflict recorded in `known-issues.md`.

## ADR-006 — Monthly submission record lifecycle

**Decision:** Do not create all employees’ monthly submission records at month start. Create `DRAFT` with the first attendance save; a successful submission with no existing record creates `SUBMITTED`; GET/list views treat absence as display-only `DRAFT`.

**Status:** PROPOSED / Phase 3 review, not approved as a whole.

**Date:** Exact date unavailable from repository evidence.

**Reason:** Avoid empty monthly records and dependence on a monthly batch while ensuring unoperated employees appear in admin lists.

**Alternatives:** Create a row for every active employee monthly; create `DRAFT` on failed submission. Not selected in current draft.

**Consequences:** Admin queries need left joins and submission API needs careful transactional behavior.

**Evidence:** `../04_database.md`, `../05_api.md`.

## ADR-007 — Employee data on MVP external services

**Decision:** Product Owner approved handling real employee data on Render Free + Neon Free MVP, with HTTPS, authorization, audit, and secrets management required.

**Status:** APPROVED

**Date:** `D-06` recorded; exact approval timestamp unavailable from repository evidence.

**Reason:** Explicit Product Owner approval.

**Alternatives:** Dummy data only until company approval. Superseded by D-06, but stale text remains in architecture.

**Consequences:** Must reconcile stale dummy-data text; data retention/deletion remains unresolved (`R-03`).

**Evidence:** `../01_requirements.md` D-06; `../07_security.md`; contradiction in `../02_architecture.md` documented in `known-issues.md`.


