# Clokka Roadmap Status

| Item | Value |
| --- | --- |
| Purpose | Record phase completion against the original roadmap, not inferred implementation progress. |
| Audience | Product Owner, AI agents, developers, reviewers. |
| Update timing | On an explicit phase approval, phase completion, block, or scope change. |
| Audit basis | `../00_プロジェクトロードマップ.md` and repository inspection on 2026-08-19. |

Status legend: `NOT_STARTED`, `IN_PROGRESS`, `REVIEW`, `APPROVED`, `BLOCKED`, `COMPLETED`.

| Phase | Name | Purpose | Deliverables | Original completion condition | Current state | Approval state |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | Requirements and project governance | Make requirements implementable and establish scope, terminology, non-scope, and document migration. | `01_requirements.md`, glossary, acceptance criteria, traceability, backlog. | Priorities and acceptance criteria approved. | `COMPLETED` per original roadmap. | Product Owner approval recorded in conversation; primary document header is stale (`レビュー・承認待ち`). |
| 2 | Architecture and technology selection | Select implementation, deployment, notification, auth, and DB approach. | `02_architecture.md`, `03_tech-stack.md`, comparisons/ADRs. | Every technology has comparison, reason, drawbacks, and migration cost; free-tier alternative approved. | `COMPLETED` per original roadmap and explicit Product Owner approval. | Approved; individual document headers are stale (`レビュー・承認待ち`). |
| 3 | Detailed design | Fix data, API, UI, authorization, and security contracts. | `04_database.md`, `05_api.md`, `06_screen-design.md`, `07_security.md`, ERD, flows, threat analysis. | Traceability to design/tests; role boundaries, submission, Excel, and notification reviewed. | `IN_PROGRESS` / design review. | Not approved as a whole. Individual residual risks were accepted, but this is not Phase 3 approval. |
| 4 | Development, quality, and Git rules design | Define directory, coding, Git, PR, and test rules. | `08_directory.md`, `09_development-rules.md`, `10_branch-strategy.md`, `11_test-plan.md`, templates. | Rules and test levels approved. | `NOT_STARTED`. Existing Japanese placeholder docs are not Phase 4 deliverables. | Not requested. |
| 5 | Deployment and operations design | Define sustainable deployment, backups, monitoring, recovery, and secrets. | `12_deployment.md`, `13_operation.md`, backup/recovery/monitoring procedures. | Environment, HTTPS, backup, recovery, responsibility, free-tier monitoring approved. | `NOT_STARTED`. | Not requested. |
| 6 | Development environment and repository foundation implementation | Build reproducible local environment and CI skeleton. | Spring Boot/frontend skeleton, Docker, migration base, CI, `.env.example`, README. | New developer can start it and CI passes. | `NOT_STARTED`; no code/config/CI exists. | Not requested. |
| 7 | Backend implementation | Implement auth, attendance, submission, admin, export, notifications. | Schema/migrations, API, scheduler, audit, API tests. | API/auth contracts and tests pass. | `NOT_STARTED`. | Not requested. |
| 8 | Frontend implementation | Implement employee/admin UI. | Login, attendance, submission, notification, admin, export UI. | Designed screens operate on target browsers. | `NOT_STARTED`. | Not requested. |
| 9 | Integration, acceptance, and security verification | Verify business flows and failure behavior. | E2E/acceptance/security/load results, Excel sample. | 50-employee dummy month succeeds; no critical/high defects. | `NOT_STARTED`. | Not requested. |
| 10 | Production deployment and setup | Establish approved production environment and initial data. | Production, deployment/setup/recovery records, release notes. | HTTPS/monitoring/backup/restore verified and release approved. | `NOT_STARTED`. | Not requested. |
| 11 | Operations, training, and improvement | Make system supportable and improve after launch. | User/admin manuals, checklists, backlog. | Admin demonstration and first-month retrospective completed. | `NOT_STARTED`. | Not requested. |

## Phase gate warning

Do not start Phase 4 or implementation. Complete Phase 3 design review, resolve or explicitly accept the design/documentation issues in `known-issues.md`, and obtain explicit Product Owner approval first.

