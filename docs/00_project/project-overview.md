# Clokka Project Overview

| Item | Value |
| --- | --- |
| Purpose | Explain the project’s confirmed product scope and intended technical direction for successors. |
| Audience | Product Owner, AI agents, developers, reviewers, operators. |
| Update timing | When approved requirements, scope, architecture, or final goal changes. |
| Audit basis | Repository contents inspected on 2026-08-19. |

## Project

Clokka is a browser-based attendance and monthly-submission management application for approximately 50 employees. Employees enter daily attendance on a smartphone or PC; administrators manage submission status and export monthly data as Excel.

## Most important objective

Prevent missed monthly attendance submissions. The target is zero missed submissions at the deadline.

## Users

| User | Confirmed responsibilities |
| --- | --- |
| Employee | Enter and edit own attendance before submission, review monthly status, submit, and configure browser Push notifications. |
| Administrator | View and filter submission status, return submitted records with a reason, export Excel, manage employees/departments/calendar/deadlines, and view aggregated Push status. |

## Confirmed initial scope

- Browser use on smartphone and PC.
- Employee ID/password authentication and employee/admin authorization.
- Daily clock-in, clock-out, break minutes, and note.
- Work date is based on JST clock-in date. Overnight work belongs to the start date; multiple clock-ins and 24-hour-or-longer work are out of scope.
- Monthly validation, submission, edit lock after submission, and administrator return.
- Submission-status management, sorting/filtering, and XLSX export.
- Web Push plus in-app warnings; an administrator can filter Push status (`GRANTED`, `DENIED`, `DEFAULT`, `UNSUPPORTED`, `UNKNOWN`). External contact for Push-refusing users is out of initial scope.
- Company holidays and workday overrides.
- Audit logging, HTTPS, authorization, and planned backup/restore operations.

## Explicitly out of scope for the initial release

- Leave requests/display (future consideration).
- Payroll, overtime-pay calculation, statutory reports, shifts, expense management, SMS, paid email, native apps, biometrics, location/QR punching, and multi-tenant operation.

## Intended MVP stack (approved in Phase 2)

| Area | Intended choice | Status in repository |
| --- | --- | --- |
| Backend | Java 21, Spring Boot 3, Gradle Wrapper | Designed only; no project files/code. |
| Frontend | HTML/CSS + ES Modules Vanilla JavaScript | Designed only; no files except `.gitkeep`. |
| Database | Neon Free PostgreSQL | Designed only; no schema/migrations/configuration. |
| DB migrations | Flyway OSS | Designed only. |
| Auth | Spring Security server session + Argon2id | Designed only. |
| Notifications | Web Push (VAPID) + in-app warning | Designed only. |
| Excel | Apache POI XLSX | Designed only. |
| MVP hosting | Render Free Web Service built from a Dockerfile, GitHub-linked deployment | Designed only; no Dockerfile, Render manifest, or deployment evidence. |
| CI | GitHub Actions | Designed only; no workflow exists. |

## Development policy

Priorities are maintainability, simplicity, free cost, security, extensibility, then performance. The approved MVP tolerates Render Free’s 15-minute idle shutdown; a production hosting migration will be reconsidered before formal operation. AWS/GCP/OCI are not MVP hosting choices; OCI is prohibited by Product Owner direction.

## Final goal

Complete and operable system: approved design documents, repository/development environment, DB/API/UI/backend/frontend, tests/CI, MVP deployment, operational and maintenance procedures, setup instructions, user manual, and administrator manual. This goal is not yet reached.

## Primary evidence

- `../01_requirements.md`
- `../02_architecture.md`
- `../03_tech-stack.md`
- `../04_database.md`
- `../05_api.md`
- `../06_screen-design.md`
- `../07_security.md`


