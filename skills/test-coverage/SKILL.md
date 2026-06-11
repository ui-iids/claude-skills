---
name: test-coverage
description: Add or update a comprehensive test suite for a project and drive it to at least 80% coverage on every half of the stack that exists. Frontend uses Playwright with tests under `test/frontend/`; backend uses pytest (with pytest-cov) under `test/backend/`. Backend tests must cover every API route and every health-check endpoint. Works on a dedicated branch, commits per phase, and iterates until both halves hit the 80% threshold or it explicitly reports it cannot. Use when the user invokes `/test-coverage`, asks to "add tests", "write a test suite", "get coverage to 80%", "test all the API routes", or similar.
---

# Test coverage

You are building (or expanding) a project's automated test suite to a
**hard minimum of 80% line coverage** on every half of the stack that
exists in the repo. The deliverable is real, runnable tests that catch
regressions — not stubs, not snapshots of current output, not tests
that mock away the thing they claim to verify.

## Non-negotiables

- **Work on a dedicated branch.** Never write tests directly on `main`
  / `master` / `develop`. Create and switch to:
  `test/<git-user>-coverage-<MM>-<DD>-<YYYY>`
  where `<git-user>` is `git config user.name` lowercased with spaces
  replaced by `-`, and the date is today. Example:
  `test/jarred-kvistad-coverage-05-22-2026`.
- **80% is a hard gate, not a target.** Keep iterating — identify
  uncovered lines, write tests for them, re-run coverage — until each
  half of the stack that exists is at ≥80% line coverage, OR you
  explicitly report which files/branches you cannot cover and why
  (e.g. unreachable error paths, hardware-dependent code). Do not
  declare success below 80% silently.
- **Tests live in `<project-root>/test/`.** Frontend tests under
  `test/frontend/`, backend tests under `test/backend/`. Create these
  directories if missing. If the project already uses a different
  conventional location (e.g. `tests/`, `__tests__/`, `spec/`), ask
  the user before moving — but new tests written by this skill go
  under `test/frontend/` or `test/backend/`.
- **One commit per phase**, so any phase can be reverted independently.
  Do not squash.
- **No behavior changes in production code.** Adding tests should not
  require editing source files. Exceptions: (a) exporting an internal
  symbol so a test can import it — ask first, (b) adding a test-only
  hook (e.g. `data-testid` attributes) — ask first.
- **Skip the half that doesn't exist.** Frontend-only repo → only do
  Playwright. Backend-only repo → only do pytest. Don't fabricate the
  missing half.
- **Tests must actually run and pass.** A failing or skipped test
  doesn't count toward coverage. If a test you wrote fails, fix the
  test (or the underlying bug, with the user's approval) — don't
  delete it or mark it `xfail` to make the suite green.

## Procedure

### 1. Pre-flight

1. Confirm a clean working tree: `git status --porcelain`. If dirty,
   ask the user to commit or stash before continuing.
2. Detect project shape:
   - **Frontend** present if any of: `package.json` with a UI
     framework dep (`react`, `vue`, `svelte`, `next`, `nuxt`, `astro`,
     `vite`), an `index.html` entry, or directories like `frontend/`,
     `client/`, `web/`, `app/`, `src/components/`.
   - **Backend** present if any of: `pyproject.toml` / `requirements*.txt`
     with a web framework (`fastapi`, `flask`, `django`, `starlette`,
     `aiohttp`, `tornado`), or directories like `backend/`, `server/`,
     `api/`, `services/`.
   - A repo may be one, the other, or both. Note which.
3. Detect existing test infrastructure:
   - Frontend: `playwright.config.*`, `vitest.config.*`, `jest.config.*`,
     `cypress.config.*`, existing `test*/` / `__tests__/` / `e2e/`
     directories.
   - Backend: `pytest.ini`, `pyproject.toml [tool.pytest.ini_options]`,
     `conftest.py`, `tox.ini`, `tests/` / `test/` directories.
   - Existing coverage config: `.coveragerc`, `[tool.coverage.*]` in
     `pyproject.toml`, `nyc`/`c8` config.
4. Detect the runtime/package manager:
   - JS/TS: `pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, otherwise npm.
   - Python: `poetry.lock` → poetry, `uv.lock` → uv, `Pipfile.lock` →
     pipenv, otherwise plain `pip` + venv.
5. **Create and switch to the test branch now**, before any edits:
   `git checkout -b test/<git-user>-coverage-<MM>-<DD>-<YYYY>`.
6. Create `test/frontend/` and/or `test/backend/` (only the halves
   that apply). If the project has existing tests in another location,
   leave them in place — do not move them.

### 2. Phase 2 — Install / verify tooling

Only install what's missing. Report each command to the user before
running it.

**Frontend (if present):**
- `@playwright/test` as a dev dependency.
- `playwright install` (browsers) — chromium at minimum.
- Coverage: Playwright's built-in V8 coverage via the `coverage` flag
  in `page.coverage.startJSCoverage()`, or `monocart-reporter` /
  `@playwright/test` + `nyc`/`c8` for instrumentation-based coverage
  of the app code under test. Prefer `monocart-reporter` for a single
  HTML+JSON coverage artifact.
- `playwright.config.ts` (or `.js`) at the repo root, with
  `testDir: './test/frontend'`.

**Backend (if present):**
- `pytest`, `pytest-cov`, and the framework's test client
  (`httpx` + `fastapi.testclient.TestClient` for FastAPI, Flask's
  built-in `app.test_client()` for Flask, Django's test client for
  Django).
- `pytest.ini` or `[tool.pytest.ini_options]` in `pyproject.toml`
  configured with `testpaths = ["test/backend"]` and
  `addopts = "--cov=<source_pkg> --cov-report=term-missing --cov-report=html --cov-fail-under=80"`.
- A `conftest.py` under `test/backend/` with shared fixtures
  (app instance, test client, isolated DB/session if applicable).

Commit this phase: `chore(test): scaffold test tooling and config`.

### 3. Phase 3 — Backend tests (if backend exists)

Goal: every API route is exercised, every health-check returns the
expected payload, error paths are tested, and line coverage on the
backend source package is ≥80%.

1. **Enumerate routes.** Read the app's route registration code
   (`@app.route`, `@router.get`, `app.include_router`, Django
   `urls.py`, etc.) and produce a complete list of `(method, path,
   handler)` tuples. Use this list as the test checklist — every
   entry must have at least one test.
2. **Health checks.** Identify any `/health`, `/healthz`, `/ready`,
   `/live`, `/status`, `/ping`, or similar endpoints. Each gets a
   dedicated test asserting status code, content-type, and payload
   shape.
3. **For each route, write tests that cover:**
   - The happy path (valid input → expected status + body).
   - Auth/authorization branches if the route is guarded
     (unauthenticated → 401, wrong scope → 403).
   - Input validation failures (missing required fields, wrong types
     → 422/400) for routes that accept a body or query params.
   - Notable error paths the handler explicitly raises (404 for
     missing resources, 409 for conflicts, etc.).
4. **Non-route code** (services, repositories, utilities, business
   logic): write focused unit tests for any module pulled into the
   coverage report below 80%.
5. **Fixtures and isolation:**
   - Database: use an isolated test DB (sqlite in-memory or a
     transactional fixture that rolls back per test). Never run tests
     against a real dev/prod DB.
   - External services: stub at the HTTP boundary using `respx`
     (httpx), `responses` (requests), or framework equivalents — do
     not mock the function being tested.
   - Time/UUIDs/randomness: freeze with `freezegun` /
     `pytest-freezer` or dependency injection.
6. **Run the suite with coverage:**
   `pytest --cov=<pkg> --cov-report=term-missing --cov-fail-under=80`.
   Read the `Missing` column. For every file below 80%, write tests
   for the listed line ranges and re-run. Repeat until the gate
   passes.
7. If a file genuinely cannot be covered (e.g. `if __name__ ==
   "__main__":` blocks, optional-import fallbacks, hardware-only
   code), exclude it via `# pragma: no cover` on those specific lines
   — not whole files — and note each exclusion in the final report.

Commit: `test(backend): add comprehensive route + coverage tests`.

### 4. Phase 4 — Frontend tests (if frontend exists)

Goal: every user-facing route/page renders without error, primary user
flows are exercised end-to-end, and line coverage on the frontend
source is ≥80%.

1. **Enumerate pages/routes.** Read the router config (React Router,
   Next.js `app/`/`pages/`, Vue Router, etc.) and list every route.
   Each gets at least one navigation + render test.
2. **Enumerate primary flows.** Ask the user (or infer from the app)
   the top user journeys — login, search, submit form, etc. — and
   write one Playwright test per flow that asserts the visible
   outcome (text on screen, URL change, network call made), not
   implementation details.
3. **Component tests** for shared/reusable components: render in
   isolation via Playwright component testing
   (`@playwright/experimental-ct-*`) if the project already uses it,
   otherwise via the existing unit runner (Vitest/Jest). Don't
   introduce a second runner if one is already in use — extend it,
   and source new Playwright E2E files into `test/frontend/`.
4. **Accessibility & error states:** include at least one
   `@axe-core/playwright` accessibility check on a representative
   page, and tests for 404 / error-boundary behavior.
5. **Wire up coverage.** Use `monocart-reporter` (preferred) or the
   `page.coverage` API to collect V8 coverage during E2E runs and
   produce a single coverage report. Configure the reporter in
   `playwright.config.ts` with `coverage: { entryFilter: ...,
   sourceFilter: ... }` limiting to the app's source directory.
6. **Run and iterate:**
   `npx playwright test --reporter=monocart-reporter` (or the
   project's package-manager equivalent). Inspect the coverage HTML.
   For every source file below 80%, write additional tests targeting
   the uncovered lines and re-run. Repeat until ≥80%.
7. The Playwright test server: prefer `webServer` config in
   `playwright.config.ts` so tests start the app automatically. Use a
   dedicated port and a `reuseExistingServer: !process.env.CI` flag.

Commit: `test(frontend): add Playwright E2E + coverage tests`.

### 5. Phase 5 — Glue, scripts, and a short README

1. Add (or update) top-level test scripts so the suite is one
   command per half:
   - `package.json`: `"test:frontend": "playwright test"`,
     `"test:frontend:coverage": "playwright test --reporter=monocart-reporter"`.
   - Backend: a `Makefile` target or `pyproject.toml` script,
     `test-backend = "pytest"`.
   - A combined `test` script that runs both halves if both exist.
2. Add a short `test/README.md` (one screen max) explaining:
   - How to run each half.
   - Where coverage reports land.
   - The 80% threshold and how it's enforced
     (`--cov-fail-under=80` / Playwright config).
3. Do **not** modify CI config (`.github/workflows/*`, etc.) unless
   the user explicitly asks — that's a separate change with its own
   blast radius.

Commit: `chore(test): add test runner scripts and test/README`.

### 6. Phase 6 — Final verification & report

1. Run the full suite, both halves, from a clean checkout of the
   branch. Capture:
   - Pass/fail/skip counts per half.
   - Final coverage % per half, plus per-file table sorted ascending.
   - Total wall-clock time.
2. Confirm the 80% gate passes for every half that exists. If it
   doesn't, **do not declare success** — return to Phase 3/4 and add
   more tests, or escalate to the user with a specific list of files
   that can't be covered and why.
3. Report to the user:
   - Branch name and commit list.
   - Coverage numbers per half.
   - Any `# pragma: no cover` / coverage-config exclusions added,
     with one-line justifications.
   - Any production-code changes requested (e.g. exposed internals,
     added test hooks) — these should already have been approved in
     earlier phases; the final report just re-lists them.
   - Commands to run the suite locally.

## Things to avoid

- **Tautological tests.** `expect(x).toEqual(x)`, snapshotting the
  current output without asserting anything about it, asserting that
  a function was called when calling it is the only behavior under
  test.
- **Over-mocking.** If the test mocks the function it's supposed to
  verify, delete it.
- **Coverage-padding.** Don't write tests that import a module just
  to bump its coverage % without exercising real behavior.
- **Flaky tests.** Avoid `setTimeout`/`sleep` for synchronization;
  use Playwright's auto-waiting and pytest's explicit fixtures. A
  flaky test is worse than no test.
- **Coupling to implementation details.** Test observable behavior
  (HTTP responses, rendered DOM, URL changes) — not internal call
  graphs.
- **Leaving the branch dirty.** Every phase ends with a clean
  working tree and one commit. If a phase produced no changes, say
  so and skip the commit rather than committing empty.
