# Pull Request

## Purpose
<!-- What FR/ADR/Q does this change address? Link to the design doc section. -->

## Related
- Q- / ADR- / FR-
- `known-issues.md` entry if applicable:

## Changes
<!-- Small, single-purpose. List files and the rule from 08_directory.md that justifies the location. -->

## How to test
<!-- `./gradlew check` and the specific test from 11_test-plan.md. Include X-Request-Id and role checks if touching API. -->

## Secrets / migrations
- [ ] No secrets, no `.env`, no employee data
- [ ] No Flyway migration after it was merged to `main` (or new `V{NN}__` with rationale)
- [ ] Push subscription fields remain encrypted (if touched)

## Checklist (per AGENTS.md and 09_development-rules.md)
- [ ] `design → review → PO approval` was followed (or this is a docs-only Phase 4 change)
- [ ] No silent contract change (or ADR filed)
- [ ] `docs/00_project/current-state.md` / `known-issues.md` / `decisions.md` updated if the change is material
- [ ] `X-Request-Id` present on changed endpoints (if API)
- [ ] CI is green (`ci.yml`)
