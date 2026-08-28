# Clokka Formal Handoff

| Item | Value |
| --- | --- |
| Purpose | Hand over the project without relying on chat history. |
| Audience | Next AI agent, Product Owner, developers, reviewers. |
| Update timing | Before replacing an agent, after a phase change, or after a material audit finding. |
| Audit date | 2026-08-28. |

## Handoff Summary

Clokka is a design-first attendance-management MVP. Phase 1 (requirements) and Phase 2 (architecture/technology selection) are recorded as approved. **Phase 3 (detailed DB/API/UI/security design) is approved as a whole (Q-01 approved 2026-08-28) including the draw.io 12 PNG and the normalized repository structure (`frontend/`/`backend/`).** Phase 4 governance/test design is now in progress. No software implementation, test, CI/CD, Docker, or deployment configuration exists in the repository.

## What Has Been Completed

- Requirements, user roles, acceptance conditions, overnight-work rule, and initial scope are documented.
- Architecture and stack were selected: Spring Boot/Java 21, Vanilla JavaScript, PostgreSQL/Neon, Render Free MVP, GitHub Actions concept, Argon2id/session auth, Flyway, Apache POI, Web Push.
- Phase 3 review drafts exist for DB/API/UI/security.
- Product Owner decisions recorded in primary requirements include: overnight work uses clock-in date (`D-01`), leave is excluded (`D-03`), Push rejection status is visible to admins and external contact is out of scope (`D-04`), Render idle behavior is accepted for MVP (`D-05`), and real employee data was approved for MVP external services (`D-06`).
- Handoff documentation and permanent agent rules were created.

## What Is In Progress

- Phase 4 design and review (`08_directory.md` through `11_test-plan.md`).
- Repository structure alignment completed 2026-08-28 before Phase 4.

## What Has Not Started

- Phase 5 deployment/operation design.
- All source implementation and infrastructure: Gradle project, Spring Boot, UI, SQL migrations, Dockerfile, Render configuration, GitHub Actions workflow, test code, CI, deployment, manuals.

## Feature Traceability Snapshot

| Requirement area | Design exists | Implemented | Tested | CI verified | Runtime verified |
| --- | --- | --- | --- | --- | --- |
| Auth/roles | Yes | No | No | No | No |
| Attendance/overnight rule | Yes | No | No | No | No |
| Monthly submission/return | Yes | No | No | No | No |
| Push and status list | Yes, conflict noted | No | No | No | No |
| Admin list/filters | Yes | No | No | No | No |
| Excel export | Yes | No | No | No | No |
| Audit logging | Yes | No | No | No | No |
| Backup/operations | Requirements/concepts only | No | No | No | No |

## Known Problems

Read `known-issues.md` before editing. Highest-priority items are:

1. The repository has no implementation, tests, CI, Docker, or deployment configuration.
2. The notification idempotency model is approved as an individual Phase 3 decision; it intentionally favors at-most-once Push attempts over retry after a crash. Whole-phase approval is still required.
3. Data retention/deletion policy remains unresolved before formal operation.
4. Push subscription encryption-key backup/recovery must be designed in Phase 5 before operations can be claimed.

## Important Decisions

See `decisions.md` for evidence and alternatives. Do not revise these silently:

- `D-01`: Overnight attendance belongs to clock-in date in JST.
- `D-03`: Leave is out of initial scope.
- `D-04`: Admin sees Push status; external contact for Push-refusing employees is out of scope.
- `D-05`: Render Free idle shutdown is accepted only for MVP.
- `D-06`: Product Owner approved real employee data on the Render Free + Neon Free MVP design, subject to security controls.
- MVP hosting direction: Render Free web service + Neon Free Postgres; AWS is a later company-approved production option. OCI is prohibited for this project’s MVP.

## Do Not Change

- Do not start product implementation or Phase 4 before Phase 3 approval.
- Do not change work-date semantics, initial out-of-scope leave policy, hosting policy, or data-handling decision without Product Owner approval.
- Do not treat old Japanese documents as current architecture without resolving documented migration/conflict issues.
- Do not commit secrets, `.env` files, or employee data.
- Do not mark planned items as implemented/tested/deployed.

## Human Approval Required

| ID | Required decision |
| --- | --- |
| Q-01 | **RESOLVED 2026-08-28** — Phase 3 approved as a whole. |
| Q-04 | Approve/reject Phase 4 governance/test documents (`08`–`11` and PR template). |
| Q-05 | Before Phase 5/10: set employee-data retention/deletion policy (`R-03`) and production hosting/availability design (`R-02`, `R-05`). |

## Immediate Next Actions

1. Review Phase 4 documents (`08_directory.md` through `11_test-plan.md` plus PR template).
2. Obtain Product Owner approval for `Q-04` (Phase 4).
3. After Q-04, create Phase 5 deployment/operation design.
4. Only after Phase 4/5 approval, begin Phase 6 foundation implementation under the approved rules.

## Recommended Reading Order

1. `../../AGENTS.md`
2. `current-state.md`
3. `known-issues.md`
4. `decisions.md`
5. `roadmap.md`
6. `../01_requirements.md`
7. `../02_architecture.md` and `../03_tech-stack.md`
8. `../04_database.md`, `../05_api.md`, `../06_screen-design.md`, `../07_security.md`
9. Legacy Japanese docs only for historical comparison, not as current truth.
