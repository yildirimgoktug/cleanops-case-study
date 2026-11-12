# Sources for the numbers

Every figure in this repository was measured from a full clone of the
private source repository, or quoted from a document inside it. Paths
below are relative to that private repository. The clone was taken on
12 November 2026 at commit `6f7da47` (`main`). Counts are static: the
application, its tests and its containers were not run for this case
study.

## Measured

| Figure | Value | How it was measured |
|---|---|---|
| Django apps | 16 | Directories under `backend/` containing `apps.py` (`ls backend/*/apps.py`); matches the project apps in `INSTALLED_APPS` in `backend/config/settings.py` |
| Database models | 87 concrete (+1 abstract base) | AST scan of every non-test, non-migration module in the 16 apps for classes deriving from `models.Model`, excluding `abstract = True`. Cross-checked against migrations: 88 `CreateModel` − 1 `DeleteModel` = 87 |
| API routes | 329 | AST scan of every `urls*.py`: each `path()`/`re_path()` pointing at a view, plus the routes DRF routers generate per registered ViewSet (list route, detail route, one per `@action`). Excludes the router's API-root views, format-suffix variants, Django admin, static/media serving and the 2 health probes |
| API operations (route × HTTP method) | 494 (GET 223, POST 135, PATCH 55, DELETE 53, PUT 28) | Same scan; methods taken from each view's handlers, DRF generic-view bases, ViewSet mixins and `@action(methods=...)` |
| Frontend page components | 74 | Files named `*Page.tsx` (all under `frontend/src/pages/`) |
| Frontend client routes | 77 | Distinct `path="..."` values on `<Route>` elements in `frontend/src/App.tsx`, including the catch-all and about ten legacy redirect paths |
| Backend test methods | 6,169 in 428 modules | `grep -rhE "^\s+(async )?def test" --include="test*.py" backend` ; modules = `test*.py` files containing at least one |
| Backend tests executed by CI | 6,177 (all OK, 2 skipped) | Sum of the 13 `Ran N tests` lines in the log of the `test` workflow run for commit `6f7da47` on `main` (12 Nov 2026), read with `gh run view --log` |
| Tests per shard group | see docs/testing-and-ci.md | Same CI log |
| Frontend unit tests | 142 `it()` blocks in 21 files | `*.test.ts` / `*.test.tsx` under `frontend/src/` |
| Playwright tests | 324 `test()` blocks in 54 spec files (50 end-to-end, 4 visual) | `*.spec.ts` under `frontend/tests/` |
| CI backend shards | 13 | Entries in `backend/ci_shards.json`; the CI log's plan step prints "13 shards cover 17 of 18 units with tests" (the 18th is the exempt fixtures module) |
| CI wall time, sharded | latest run 14m41s; last 8 green runs 14m21s–15m46s; longest shard 14m21s | `gh run list --workflow test.yml` (startedAt → updatedAt) and `gh run view --json jobs` |
| Containers in the production compose file | 6 (db, redis, backend, worker, beat, frontend) | `services:` keys in `docker-compose.prod.yml` |
| Scheduled background jobs | 8 | Entries in `CELERY_BEAT_SCHEDULE`, `backend/config/settings.py` |
| Hard RBAC invariants | 12 (H-1 … H-12) | Rows in section 3 of `docs/reference/rbac-matrix.md` |
| Permission keys | 13 provider-side, 17 customer-side | Elements of the two key sets in `backend/accounts/permissions_v2.py` and `backend/customers/permissions.py` |
| Translation bundles | 14 per language (nl, en) | Files in `frontend/src/i18n/nl` and `frontend/src/i18n/en` |
| Languages | Python and TypeScript (main); GitHub also lists CSS, Shell, JavaScript, Dockerfile, HTML — 7 in total | `gh api repos/<owner>/<repo>/languages` |
| Lines of code | Python app code 111,771; Python tests 141,753; frontend TypeScript/TSX (excluding tests) 142,886 | `git ls-files` + `wc -l`; Python app code excludes tests and migrations |
| Commits | 639 on `main`, 3 May – 12 Nov 2026 | `git rev-list --count HEAD`, `git log --reverse --format=%ad` |
| Merged pull requests | 129 | `gh pr list --state merged --limit 300` |
| Model, state and transition names in docs/domain-model.md | — | `backend/tickets/state_machine.py`, `backend/extra_work/state_machine.py`, `backend/extra_work/proposal_state_machine.py`, `backend/invoicing/state_machine.py` |

## Quoted from documents in the private repository

These were not re-measured, because doing so would mean running the
application or its full test suite.

| Figure | Value | Source |
|---|---|---|
| Full backend suite, serial run | 6,077 tests in 21,268 s (5.9 h), OK | `docs/testing/p16-suite-repair.md` and `docs/planning/sprint-checklist.md` (P-16, Sep 2026): "Ran 6077 tests in 21267.918s — OK (skipped=4)"; also the comment in `.github/workflows/test.yml` |
| Unsharded CI job killed at | 50m22s (50-minute timeout) | `.github/workflows/test.yml` comment; `backend/ci_shard_check.py` docstring |
| First sharded plan | 7 shards | `docs/planning/sprint-checklist.md`, P-19 Part C |
| Runner shrink 4 → 2 vCPU on going private, and the shard re-tune | — | Commit message of `82d550b` in the private repository |
| Project-package tests missed by the first checker | 12 tests in 2 modules | `docs/planning/sprint-checklist.md`, P-20 Part A; `backend/ci_shard_check.py` docstring |
| Read-endpoint probe | 194 GET endpoints × 9 roles: 0 server errors, 0 tenant leaks, content-verified | `docs/testing/p14-findings.md` ("The numbers") and `docs/planning/sprint-checklist.md` (P-14), Sep 2026. Reported in that audit; the nine personas are not individually listed there |
| State-machine probe | 354 attempts (ticket 191 illegal + 5 legal, extra work 85 + 4, proposal 31 + 3, invoice 30 + 5): 0 illegal transitions allowed, 0 non-sentence refusals, 0 server errors | `docs/testing/p14-findings.md`, section "B6 — the state-machine sweep's verdict" |
| Quote-to-credit-note money chain reconciled | to the cent | `docs/testing/p14-findings.md` and `docs/planning/sprint-checklist.md` (P-14) |
| Full Playwright run | 401 passed (1.1 h), 0 failed, 0 skipped | `docs/planning/sprint-checklist.md`, P-16 Part B |
| 99 worktrees removed at a clean-up | 99 | `docs/planning/sprint-checklist.md`, P-20 Part C |
| Deployment status | production not deployed; runs on an acceptance/test environment | `README.md` of the private repository ("production is not deployed yet") |

Rounded figures used in earlier drafts were replaced by the exact static counts described above.
