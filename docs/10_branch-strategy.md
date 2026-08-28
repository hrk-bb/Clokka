# Clokka Branch Strategy and Git Rules (Phase 4)

| Item | Value |
| --- | --- |
| Purpose | Define how AI and humans create branches, commits, and PRs without history pollution or secret leaks. |
| Audience | Developers, AI agents, reviewers |
| Status | REVIEW — awaiting Q-04 |
| Last updated | 2026-08-28 |
| Depends on | `08_directory.md`, `09_development-rules.md`, `AGENTS.md` |

## 1. Branch model

* **Main branch:** `main` (protected). Direct push is forbidden; all changes go via PR. `main` is always deployable to Render and always passes CI.
* **Feature branches:** `feat/<short-topic>` or `ai/<topic>` for AI work. Examples: `feat/attendance-validation`, `ai/fix-push-encryption`.
* **Fix branches:** `fix/<short-topic>` for bug fixes on `main`.
* **Docs-only branches:** `docs/<short-topic>` for documentation changes.
* **No long-lived branches.** A feature branch lives < 3 days or < 300 LOC changed; split if larger.
* **No `develop` branch** in the MVP (single `main` + short branches keeps Render's GitHub auto-deploy simple).

## 2. From branch to PR

```
main (protected)
  └─ feat/xxx  ──PR──► main
```

1. `git fetch origin && git checkout -b feat/xxx origin/main`
2. Small commits on the branch (see §4)
3. `git push -u origin feat/xxx` and open a PR via `gh pr create` (or GitHub UI)
4. PR must be **reviewed by one human or one designated AI reviewer** before merge
5. Merge via **Squash and merge** (keeps `main` linear). The squash commit message is the PR title.

## 3. Commit rules

* **One purpose per commit.** `feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:` prefixes. Examples:
  * `feat: add attendance PUT validation`
  * `fix: reject attendance write when submission is SUBMITTED`
  * `docs: update 04_database DDL for break check`
* **No secrets, no employee data, no `.env`.** The pre-commit hook (added in Phase 6) must reject `*.env`, `*.pem`, `*.key`, and files matching `*_test.csv` containing real names.
* **No `Co-authored-by` noise** unless pairing.
* **No force-push to `main`.** Force-push on a feature branch is allowed only before review starts.

## 4. PR rules

* **Template:** `.github/pull_request_template.md` must be filled. Every PR describes: purpose, related FR/ADR/Q, testing done, and whether secrets/migrations are involved.
* **Size:** < 400 lines of production code (excluding generated migrations). Larger PRs must be split.
* **Checks:** `ci.yml` must be green (build + test + secret scan). A PR with a failing CI is not reviewed.
* **Review:** At least one approval. Reviewers check `AGENTS.md:5` (no guessing, no silent contract changes) and `09_development-rules.md:3-6`.
* **Do not mark as draft** to bypass CI. Draft is only for early feedback.

## 5. Tags and releases

* No tags in Phase 6–9. Tag `v0.1.0-mvp` is created only at Phase 10 production deployment, after PO release approval.

## 6. Local Git hygiene

* `core.autocrlf=input` on macOS/Linux, `core.autocrlf=true` on Windows (or rely on `.editorconfig` `lf`). Do not commit CRLF-only changes.
* `git config pull.rebase false` (merge pull) is fine for this repo; no rebase of `main` onto feature branches after review has started.
* Keep `git fetch --prune` hygiene; delete merged branches with `git branch -d`.

## 7. What is forbidden

* Direct push to `main`, force-push to `main`, or merging one's own PR without review.
* Committing `V{NN}__` migrations that have already been merged to `main` (they are immutable).
* Committing `frontend/node_modules/`, `build/`, `out/`, or IDE files (they are `.gitignore`d).

## 8. Verification before Phase 6

Create a throwaway branch `docs/test-branch-strategy` from `main`, push, open a PR, see `ci.yml` (added in Phase 6) run, then close without merging. This verifies branch protections without polluting `main`.
