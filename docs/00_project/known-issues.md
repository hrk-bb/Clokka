# Clokka Known Issues and Audit Findings

| Item | Value |
| --- | --- |
| Purpose | Make risks, gaps, and contradictions visible without silently changing existing specifications. |
| Audience | Product Owner, AI agents, developers, reviewers, operators. |
| Update timing | When a problem is found, resolved, accepted, or superseded. |
| Audit basis | Repository inspection and documentation synchronization on 2026-08-24. |

| ID | Category | Content | Impact | Current state | Workaround | Recommended response | Priority | Related files |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| KI-001 | IMPLEMENTATION | No backend/frontend code exists. `backend/` and `frontend/` contain only `.gitkeep`. | All required features are unimplemented. | Open; expected before Phase 6–8. | None. Do not claim functionality. | Complete Phase 3–5 approvals before implementing. | Critical | `backend/.gitkeep`, `frontend/.gitkeep`, roadmap. |
| KI-002 | TEST / CI/CD | No test source, build configuration, Dockerfile, Compose file, environment template, or GitHub Actions workflow exists. `.github/workflows/` has only `.gitkeep`. | No automated quality, reproducibility, deployment, or security verification. | Open; expected before Phase 6. | None. | Design Phase 4/5, then implement CI and test foundation. | Critical | `.github/workflows/.gitkeep`, `README.md`. |
| KI-003 | GIT / UNKNOWN | Git executable is unavailable. `.git/HEAD` names `master`, but no refs, index, logs, remote, or reachable history were found. Local recent commits, uncommitted changes, remote/PR/Actions state cannot be verified. | Cannot truthfully report branch ancestry, commit history, PRs, or CI results. | Open; environment/repository metadata issue. | Treat all Git facts except raw `.git` files as unverified. | Restore Git tooling; verify intended GitHub remote and compare working tree before any commit/push. | High | `.git/HEAD`, `.git/config`. |
| KI-009 | DESIGN_ISSUE | Architecture/API require notification history/idempotency (“same employee not twice per day”), but DB design has no notification delivery/log table or equivalent field. | Cannot implement reliable deduplication/audit from current DB design alone. | Open; Phase 3 gap. | Do not invent schema during implementation. | Add/approve notification-delivery persistence design and API/job behavior. | High | `docs/02_architecture.md`, `docs/04_database.md`, `docs/05_api.md`. |
| KI-010 | SECURITY | No `.gitignore` or secret-scanning/CI configuration exists. No secrets were found in workspace files, but GitHub/Render/Neon secret stores are unverified. | Future accidental secret commit risk. | Open; no exposed secret evidenced. | Do not create `.env` under version control. | Phase 4/6: add approved `.gitignore`, `.env.example` without values, secret scanning, and CI checks. | High | Repository root, `.github/workflows/`. |
| KI-011 | DEPLOYMENT | Render/Neon/GitHub deployment is design only. No Dockerfile, Render config, GitHub connection, Action schedule, secrets, health check, or deployed URL exists. | MVP cannot run; deployment status is unknown. | Open; expected before Phase 6/10. | None. | Do not state deployment exists; design Phase 5 then create foundation after approval. | High | `docs/02_architecture.md`, `docs/03_tech-stack.md`. |
| KI-012 | OPERATIONS / SECURITY | Data retention and deletion after retirement is unresolved (`R-03`). Current intent is disable account and retain history. | Legal/policy compliance unknown before real operation. | Accepted residual risk; revisit before Phase 10. | Do not delete history by default. | Product Owner/company policy must specify retention/deletion and backup retention before production. | High | `docs/04_database.md`. |
| KI-013 | DEPLOYMENT / PERFORMANCE | Render Free 15-minute idle shutdown and GitHub Actions best-effort reminder triggering are accepted only for MVP (`D-05`, `R-02`, `R-05`). | Slow cold starts and unreliable scheduled reminder availability; not suitable for formal operation. | Accepted for MVP, must be redesigned before production. | Admin status list is designed as fallback. | Reassess hosting, availability, scheduler, monitoring, backup and alerting before Phase 10. | High | `docs/02_architecture.md`, `docs/03_tech-stack.md`, `docs/07_security.md`. |

## Resolved audit findings (2026-08-24)

| ID | Resolution |
| --- | --- |
| KI-004 | README now points to `01_requirements.md`; Japanese legacy documents carry an explicit history/primary-document notice. |
| KI-005 | The functional-requirements table was repaired; FR-01 through FR-12 are in one table. |
| KI-006 | `04_database.md` now uses the same Push aggregation algorithm as requirements/API: `GRANTED`, then `DEFAULT` if any unconfigured installation, then `DENIED` only if all supported installations deny, then `UNSUPPORTED`, then `UNKNOWN`. |
| KI-007 | Duplicate primary Push-state definition and duplicate S-07 help text were removed. |
| KI-008 | Architecture risk wording now follows D-06 and the Render-based MVP; stale dummy-only and VM-specific wording was replaced. |
| KI-014 | Phase 1/2 primary-document headers and current roadmap wording now reflect approved/completed status. |

## Security audit result

- No `.env`, certificate/key/credential-named workspace file, or obvious literal secret was found by filename and text-pattern scan.
- No code/configuration exists, so there was no application-level secret/configuration audit to perform.
- GitHub Secrets, Render environment variables, Neon credentials, remote history, and private repository security settings are **not accessible/verified**.
- No real employee data was found in the workspace files inspected.
