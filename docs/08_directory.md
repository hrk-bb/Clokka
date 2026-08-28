# Clokka Directory Design (Phase 4)

| Item | Value |
| --- | --- |
| Purpose | Define the repository file layout that an AI or human developer can follow without guessing where to place or find code, tests, migrations, docs, and CI. |
| Audience | Developers, AI agents, reviewers, Product Owner |
| Status | REVIEW — awaiting Q-04 |
| Last updated | 2026-08-28 |
| Depends on | `01_requirements.md`, `02_architecture.md`, `03_tech-stack.md`, `04_database.md`, `05_api.md`, `06_screen-design.md`, `07_security.md` |

## 1. Top-level layout

```
Clokka/
├── .github/
│   ├── workflows/          # GitHub Actions (CI, scheduled reminders)
│   └── pull_request_template.md
├── backend/                # Spring Boot application (Java 21)
├── frontend/               # Vanilla JS UI (HTML/CSS/ES Modules)
├── docs/                   # Design and project docs (this directory)
│   ├── 00_project/         # Current-state, handoff, decisions, roadmap, known-issues
│   ├── 00_プロジェクトロードマップ.md # Original Japanese roadmap (history)
│   ├── 01_requirements.md .. 14_backlog.md
│   └── 06_screen-images/   # PNG exports from 06_screen-design.drawio
├── .gitignore
├── .env.example            # No secrets, only keys and placeholders
├── docker-compose.yml      # Local development only (optional)
├── Dockerfile              # Backend + frontend static bundle for Render
└── README.md
```

* `src/` and `tests/` at the repository root are **not used**. The normalized structure before Phase 4 removed them in favor of `frontend/` and `backend/` to match `README.md:11-12` and the Phase 2 decision for a separate frontend bundle.
* `frontend/` and `backend/` each contain a `.gitkeep` until Phase 6 creates their skeletons; they must not be removed to keep the directories tracked.

## 2. Backend (`backend/`)

```
backend/
├── build.gradle            # Gradle Wrapper, Java 21, Spring Boot 3
├── settings.gradle
├── src/
│   ├── main/
│   │   ├── java/com/clokka/
│   │   │   ├── ClokkaApplication.java
│   │   │   ├── identity/       # Auth, session, UserDetails, Argon2id
│   │   │   ├── attendance/     # Attendance, work_date JST logic
│   │   │   ├── submission/     # Monthly submission + validation
│   │   │   ├── admin/          # Admin queries, return, Excel, calendar, deadlines
│   │   │   ├── notification/   # Push, deliveries, attempts, job API
│   │   │   ├── audit/          # Audit log service (append-only)
│   │   │   └── config/         # Security, web, Flyway, Jackson
│   │   └── resources/
│   │       ├── application.yml          # No secrets; env overrides
│   │       ├── application-local.yml    # Local profile (gitignored if needed)
│   │       └── db/migration/            # Flyway V1__..., V2__... (SQL only)
│   └── test/
│       ├── java/com/clokka/   # Mirrors main packages
│       └── resources/
│           └── application-test.yml
├── .gitkeep                # Until Phase 6
└── README.md               # Backend-only run instructions (added in Phase 6)
```

* Package boundaries follow `02_architecture.md:3` modular monolith: `identity`, `attendance`, `submission`, `admin`, `export` (inside `admin`), `notification`, `audit`.
* `db/migration` holds Flyway versioned SQL only; no Java migrations in the MVP.
* `src/main/resources/static/` is **generated** at build time by copying `frontend/` — never hand-edited.

## 3. Frontend (`frontend/`)

```
frontend/
├── index.html              # Single entry; same-origin served by Spring Boot
├── css/
│   └── style.css           # Single stylesheet (no framework)
├── js/
│   ├── app.js              # Router / bootstrap
│   ├── api.js              # Fetch wrapper + CSRF + X-Request-Id
│   ├── auth.js             # Login / me / logout
│   ├── attendance.js       # S-02, S-03, S-04
│   ├── admin.js            # S-06, S-07, S-08, S-09, S-10
│   └── notification.js     # S-05, installation_id, Push
├── .gitkeep
└── README.md               # Added in Phase 6
```

* No build tool, no `node_modules`, no bundler in the MVP (per `03_tech-stack.md`). ES Modules are used directly.
* `frontend/` is copied into `backend/src/main/resources/static/` during `./gradlew build` (Gradle `copy` task). The source of truth remains `frontend/`.

## 4. Docs and images

```
docs/
├── 01_requirements.md      # Phase 1 approved
├── 02_architecture.md      # Phase 2 approved
├── 03_tech-stack.md        # Phase 2 approved
├── 04_database.md          # Phase 3 approved (includes DDL contract)
├── 05_api.md               # Phase 3 approved
├── 06_screen-design.md     # Phase 3 approved
├── 06_screen-design.drawio # Visual source (12 diagrams)
├── 06_screen-images/       # PNG exports, one per diagram (descriptive names)
├── 07_security.md          # Phase 3 approved
├── 08_directory.md         # This file (Phase 4)
├── 09_development-rules.md
├── 10_branch-strategy.md
├── 11_test-plan.md
├── 12_deployment.md        # Phase 5 (still NOT_STARTED)
├── 13_operation.md         # Phase 5
└── 14_backlog.md           # Phase 11
```

* `00_プロジェクトロードマップ.md` (Japanese) is the original plan; `00_project/roadmap.md` (English) is the phase-status tracker. Both are kept; the English file is the `Audit basis` per `roadmap.md:8`.
* `06_screen-images/` must contain one PNG per draw.io diagram with descriptive names (`S-01 共通ログイン (PC).png` etc.), not generic `S-01.png`.

## 5. CI and GitHub

```
.github/
├── workflows/
│   ├── ci.yml              # PR and main: build, test, lint, secret scan (Phase 6)
│   └── reminders.yml       # Scheduled POST /internal/jobs/monthly-reminders (Phase 10, optional for MVP)
└── pull_request_template.md
```

* `.github/workflows/` currently holds only `.gitkeep`; workflows are added in Phase 6 after the repository skeleton exists.

## 6. Root files

| File | Purpose | Secret handling |
| --- | --- | --- |
| `.gitignore` | Ignore `.env`, `build/`, `.gradle/`, `node_modules/` (if ever), IDE files | Must list `.env` before Phase 6 |
| `.env.example` | Documents required env keys with placeholder values (e.g., `DATABASE_URL=`) | No real secrets |
| `docker-compose.yml` | Local PostgreSQL for development (Phase 6) | Uses `.env` via `env_file`, not hardcoded |
| `Dockerfile` | `eclipse-temurin:21-jre` based, copies built backend jar | No secrets baked in |
| `README.md` | Points to `01_requirements.md` and `00_project/current-state.md` | No secrets |

## 7. What must not be created

* `src/` or `tests/` at the repository root (replaced by `frontend/`/`backend/`).
* `backend/src/main/resources/static/` hand-edited files (generated).
* `node_modules/`, `dist/`, `build/` committed to Git (ignored).
* `.env` with real values, `*.pem`, `*.key`, or employee CSVs.

## 8. Verification before Phase 6

A new developer/AI must be able to run:

```bash
git clone <url> && cd Clokka
cp .env.example .env   # fill local values, never commit .env
docker compose up -d   # starts local PostgreSQL
./gradlew :backend:build
```

and see `frontend/` copied into `backend/build/resources/main/static/` without manual steps. This verification is part of Phase 6 acceptance, not Phase 4.
