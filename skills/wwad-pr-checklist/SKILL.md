---
name: wwad-pr-checklist
description: WWAD ("What Would Arpan Do") pre-review readiness gate for pull requests on the Nexus production dashboard repo (`~/Documents/GitHub/nexus`, Azure DevOps). Before a reviewer is asked to look at a PR it first confirms the branch contains the current `test`, then verifies six things — planning docs are excluded from the diff; any Alembic migration is numbered and chained correctly against the target branch and other open PRs; `scripts/check.py` passes (fixing failures until it does); files over 1500 lines are flagged on the PR; a manual scan on the RCDS SonarQube shows at least 80% new-code coverage on backend and frontend, with any function over cognitive complexity 15 flagged; and `VERSION` is bumped ahead of the `test` branch with `scripts/sync_versions.py` output in the diff. It never touches CI/CD pipeline files. Use when the user invokes `/wwad-pr-checklist`, or asks "is this PR ready for review", "run the WWAD checklist", "run the Arpan checklist", "pre-review check my Nexus PR", or similar.
---

# WWAD PR checklist ("What Would Arpan Do")

Nexus is a shared production dashboard the university depends on, so the bar
for review is high. Your job is to make sure a pull request is **actually
ready** before a human reviewer spends time on it. Confirm the branch is
built on the current `test` (check 0), then work through the six checks below
in order, fix what you are allowed to fix, flag what you can't, and
finish with a pass/fail report.

You are a gatekeeper, not a rubber stamp. If a check fails and you can't
resolve it, the PR is **not ready**. Say so plainly.

## Scope

- This skill applies only to the Nexus repo (`~/Documents/GitHub/nexus`,
  origin `https://dev.azure.com/uidaho/AI4UI/_git/nexus`). If the working
  directory is a different repo, stop and tell the user.
- Nexus's own `CLAUDE.md` / `AGENTS.md` and `docs/CodeQuality.md` take
  precedence over this skill wherever they're more specific. Read them first.
  Nexus's `CLAUDE.md` asks you to **ask before making any code change**.
  Honor that for every fix below.
- Coding conventions for writing the code itself live in the companion skill
  `wwad-code-check`. Use them as the yardstick when you judge or fix code here.

## Non-negotiables

- **Never add, edit, or delete CI/CD pipeline files.** That covers
  `azure-pipelines-*.yaml`, `.github/workflows/*`, `Dockerfile.*`, and
  `docker-compose*.yml`, other than the version-string lines that
  `scripts/sync_versions.py` rewrites. The same goes for
  `azure-vars-*.yml`: only the major/minor/patch values that
  `sync_versions.py` writes may change. Run SonarQube **manually**. Do not
  wire it into any pipeline.
- **Never commit a `sonar-project.properties` file** or any other scanner
  config. Pass every Sonar setting as a `-D` flag on the command line.
- **Never push to, create, or retarget a `sprint/*` branch.** Pushing a
  `sprint/*` branch starts Azure pipeline builds. Fixes you make go on the
  PR's source branch (normally `dev/*`). Push only when the user says to,
  and only to that branch's own name.
- **Never upstream `test`.** Never push to `origin/test`, never commit to or
  merge into the local `test` branch (fast-forwarding it from `origin/test`
  is fine), and never set `test` as a branch's upstream. Changes reach `test`
  only through the reviewed PR flow.
- **Never fix SonarQube findings on your own.** A scan reports issues across
  the whole repo, most of them outside the PR's scope. Read the results,
  report them, and **ask the user how to proceed** (step 5f) before changing
  any code because of them. Fixing everything the scan finds would break the
  module-scope rule in `wwad-code-check`.
- **Ask before posting PR comments.** Comments on an Azure DevOps PR are
  visible to the whole team. Show the user the full set of comments you plan
  to post, get one confirmation, then post them.
- **Never ask for, echo, or write a token.** The SonarQube token and
  project key come only from Nexus's `.env`, which is generated from the
  1Password vault (see 5a). Azure DevOps access comes from the user's
  existing `az` login. Never put a token in the chat, on a command line
  (`-Dsonar.token=…`), in a repo file, in a PR comment, or in a commit.
- **One commit per fix category** (for example "exclude planning docs",
  "fix ruff lint", "sync versions"). Never use `--no-verify`.
- **Leave no tooling footprint in the repo.** Running this skill must not
  add dependencies, config, or output files to Nexus. The only repo changes
  allowed are fixes that belong to the Nexus work itself (lint fixes, version
  sync, untracking planning docs). In particular:
  - Never `npm install`, `npm i -D`, `uv add`, or `pip install` into the repo.
    Never edit `package.json`, `package-lock.json`, `pyproject.toml`,
    `uv.lock`, `vite.config.js`, `eslint.config.js`, or `.gitignore` to make a
    check run.
  - Run external tools only from locations outside the repo: the Docker
    image (it lives in Docker's image store), `npx`/`uvx` (they cache under
    `~/.npm/_npx` and `~/.cache/uv`), or a scratchpad install. The scanner's
    engine and JRE cache under `~/.sonar/cache`.
  - Write all output (coverage reports, scanner work dir, logs, comment
    payloads, diff lists) to the session scratchpad, never the repo tree.
  - **Prove it.** Record `git status --porcelain` before check 5 starts (and
    before any other check that runs a tool). Compare it again afterwards.
    Any new entry that isn't a deliberate Nexus fix is a skill bug: delete it
    and say so in the report.

## Inputs

- **PR ID** (optional). If one isn't given, look it up from the current branch:
  ```bash
  az repos pr list --source-branch "$(git branch --show-current)" --status active \
    --query "[].{id:pullRequestId, title:title, source:sourceRefName, target:targetRefName, draft:isDraft}" -o table
  ```
  If no PR exists yet, still run every check against the branch. Collect the
  would-be PR comments and post them once the user has opened the PR.
- **Target branch.** Use the PR's `targetRefName`. With no PR, assume the
  `sprint/*` branch with the same suffix as the current `dev/*` branch. The
  VERSION check (6) always compares against `origin/test`.

Before starting, check the tooling the same way `ado-pr-review` does
(`az extension show --name azure-devops`, `az account show`). If something is
missing, give the user the exact `! <command>` to run. Then run
`git fetch origin --prune` so every comparison uses current remote state.

Build the list of changed files once and reuse it for every check:

```bash
BASE=$(git merge-base HEAD origin/<target-branch>)
git diff --name-status "$BASE"...HEAD > <scratchpad>/changed.txt
git diff --name-only --diff-filter=AMR "$BASE"...HEAD > <scratchpad>/changed-paths.txt
```

## Procedure

Track checks 0–6 as tasks with TaskCreate. Mark each one `in_progress` when
you start it and `completed` when it reaches a verdict.

### 0. Base is current with `test` (gate, runs first)

`test` changes often. Every other check is only meaningful against the
**current** `test`: migration numbering, the VERSION comparison, and
`check.py` results can all change when `test` moves. Run this first:

```bash
git fetch origin test
git merge-base --is-ancestor origin/test HEAD && echo "base current" || echo "behind test"
git rev-list --count HEAD..origin/test          # test commits missing from this branch
git log --oneline HEAD..origin/test | head -20  # what they are
```

- **Current:** PASS. Record the `origin/test` SHA you checked against
  (`git rev-parse --short origin/test`) for the final report.
- **Behind:** propose bringing the branch up to date by **merging**
  `origin/test` into the `dev/*` branch (`git merge origin/test`). Merging
  doesn't rewrite history, so no force-push is needed. Rebase only if the
  user asks, and a rebase of a pushed branch needs their explicit approval
  for `--force-with-lease` to its own `dev/*` name. Ask before merging, and
  ask about any conflict whose resolution isn't obvious. After the merge,
  rebuild `changed.txt` (the merge-base has moved) and carry on with checks
  1–6 against the updated branch. Mark the row FIXED.
- Also confirm the branch isn't tracking `test`:
  `git rev-parse --abbrev-ref @{u}` must not print `origin/test`. If it does,
  unset it with `git branch --unset-upstream` and say so.
- If the branch can't be brought current (unresolved conflicts, or the user
  declines), the PR is **Not ready**. Still run checks 1–6 so the user sees
  everything, but note that they ran against a stale base.

**Re-check before merge.** A PASS here goes stale as soon as `test` moves.
Re-run this check right before a PR is completed (merged), whether you're
asked to complete it or the user says it's about to merge. If `origin/test`
has moved past the SHA recorded in the report, the branch needs updating and
checks 2, 3, and 6 need re-running before anyone merges. Never complete a PR
whose source branch is behind `origin/test`, and complete one only when the
user explicitly says to.

### 1. Planning documentation excluded from the PR

Planning material must never ship in a Nexus PR. Scan `changed.txt` for
anything that looks like planning, scratch, or agent-session output:

- `docs/*planning*/`, `docs/research/`, `*PLAN*.md`, `*plan*.md`, `TODO*.md`,
  `NOTES*.md`, `scratch*`, `brainstorm*`, design-draft markdown that isn't
  wired into the docs index
- agent config and state: `.claude/`, `.codex/`, `CLAUDE.local.md`,
  `*.lock` session files, `settings.local.json`
- local artifacts: `coverage.xml`, `a11y-report/`, `.coverage`, `htmlcov/`,
  `test-output*.txt`, `.env*`

Also check that nothing planning-related is tracked on the branch even if it
didn't change in this PR (`git ls-files | grep -iE 'plan|scratch|brainstorm'`).

When something matches, list it and ask the user to confirm it's planning
material. Some docs, such as a `docs/*DataFlow.md` update, are legitimate.
Once confirmed:

- Untrack it from the branch with `git rm --cached <path>`, keeping the local
  copy.
- To keep it out for good, add the pattern to `.git/info/exclude`
  (local-only). Only edit `.gitignore` if the user asks for it, because that
  itself shows up in the diff.
- Commit: `Exclude planning documents from PR`.

**Pass** when no planning material remains in `git diff BASE...HEAD`.

### 2. Migration numbering and ordering

Skip this check if `changed.txt` has no files under
`backend/app/modules/**/alembic/versions/` (excluding `versions/archive/`).
Otherwise, for each module that has a new or changed migration:

1. **Naming convention.** Nexus migrations are named
   `<revision>_<NNNN>_<description>.py`, with a zero-padded sequence number
   per module (for example `c5e81d3b9a46_0006_widen_analytics_grants_sponsor_id.py`).
   The new file must follow this pattern.
2. **Sequence against the target.** List that module's migrations on
   `origin/<target-branch>`, `origin/test`, and `origin/main`
   (`git ls-tree -r --name-only origin/<branch> -- backend/app/modules/<module>/alembic/versions/`).
   The new `NNNN` must be exactly one more than the highest number on the
   most advanced of those branches.
3. **Chain integrity.** Open the new migration and confirm `down_revision`
   equals the `revision` of the current head on the target branch. Then
   confirm the module has a single head:
   ```bash
   PYTHONPATH=backend alembic -c backend/app/modules/<module>/alembic/alembic.ini heads
   ```
   (This command doesn't need a database.) More than one head means a branch
   in the chain, which fails the check.
4. **Collisions with other open PRs.** Look at every active PR that touches
   the same module's `alembic/versions/`:
   ```bash
   az repos pr list --status active --query "[].{id:pullRequestId, source:sourceRefName, target:targetRefName}" -o json
   ```
   For each one, `git fetch origin <sourceRefName>` and list its migrations
   for the same module. If another PR claims the same `NNNN` or the same
   `down_revision`, flag it. Whichever merges second has to renumber and
   rebase its chain. Also check the recently completed PRs into the target
   (`--status completed --top 20`) to confirm nothing landed after this branch
   was cut.
5. **Applied DB state** (if the user has a reachable dev database). Run
   `alembic ... current` and confirm the DB is at the target's head, so this
   migration would apply next. If no database is reachable, say so. Don't
   guess.

If the number or `down_revision` is wrong, propose the corrected filename and
`down_revision` and ask before renaming. Never edit a migration that already
exists on `origin/test` or `origin/main`.

**Pass** when the numbering is sequential, there's a single head, the chain
attaches to the target's head, and no open PR collides.

### 3. `scripts/check.py` passes

Run the project's full gate from the repo root:

```bash
uv run scripts/check.py
```

If it fails:

1. Run `uv run scripts/check.py --fix` to auto-fix formatting and lint
   (ruff format, ruff check, eslint/prettier), then re-run the full check.
2. Fix the remaining failures by hand: mypy errors, eslint errors without an
   autofix, failing pytest/Vitest tests. Fix the root cause. Never use
   `# type: ignore`, `# noqa`, `eslint-disable`, `@pytest.mark.skip`, or
   threshold edits to silence a check unless the user explicitly approves
   that specific suppression.
3. If a failing test looks like a real behavior regression rather than a
   stale test, stop and report it. Don't rewrite the test to match broken
   code.
4. Keep going until the full check (including tests) exits 0. Commit fixes
   by category (`Apply ruff format`, `Fix mypy errors in <module>`, and so
   on).

**Pass** when `uv run scripts/check.py` exits 0 with no flags skipped.

### 4. Files over 1500 lines

For every added or modified code or script file in `changed.txt`
(`.py`, `.js`, `.jsx`, `.ts`, `.tsx`, `.sql`, `.sh`; skip lockfiles,
generated files, fixtures, and migrations' data blobs), count lines on HEAD:

```bash
grep -E '\.(py|jsx?|tsx?|sql|sh)$' <scratchpad>/changed-paths.txt | xargs wc -l | awk '$1 > 1500'
```

For each file over 1500 lines, prepare a PR comment anchored to that file:

> **WWAD: file length.** `<path>` is <N> lines (limit 1500). Please split it
> into focused submodules (for example move helpers to `utils/`, service
> logic to `services/`) that import what they need. See `wwad-code-check`
> for the module layout.

Note whether this PR made the file grow (compare `wc -l` on BASE): "grew
from X to Y" is more actionable than the raw count. Don't refactor the file
yourself as part of this check. Flag it only.

**Pass** when no changed file exceeds 1500 lines. Otherwise **flagged**,
with a comment queued for each file.

### 5. SonarQube pass: coverage and complexity

Run a **manual** scan against the shared RCDS server
`https://sonarqube.k8s-dev.hpc.uidaho.edu`. Nothing about this check touches
pipeline files.

**5a. Prerequisites**

- **A token is required.** The server enforces authentication:
  anonymous calls to `api/authentication/validate` return
  `{"valid":false}`, and the scanner's engine bootstrap endpoint
  (`api/v2/analysis/engine`) returns 401 (verified 2026-11-19). There is no
  anonymous scan path, so don't try one.
- **Where the token and project key come from.** Both live in the team's
  1Password vault and are referenced from Nexus's tracked `.env.template`:
  ```
  SONAR_TOKEN="op://ReactFast-Dev/<item>/<field>"
  SONAR_PROJECT_KEY="op://ReactFast-Dev/<item>/<field>"
  ```
  Each developer's gitignored `.env` is generated with
  `op inject -i .env.template -o .env`. Read the two values from the
  environment if they're already exported. Otherwise copy just those two
  lines from Nexus's `.env` into a private scratch file. Never `source` the
  whole `.env`, because it holds every other app secret too:
  ```bash
  umask 077 && grep -E '^(SONAR_TOKEN|SONAR_PROJECT_KEY)=' .env \
    | sed -E 's/="/=/; s/"[[:space:]]*(#.*)?$//; s/^/export /' > <scratchpad>/.sonar_env
  grep -c '^export' <scratchpad>/.sonar_env   # expect 2; never cat this file
  ```
  If either value is missing (count below 2, or the `.env` line still reads
  `op://…`), stop check 5's server scan. Tell the user to add the entries to
  the vault and `.env.template`, then re-run
  `op inject -i .env.template -o .env`. Run the local fallback below in the
  meantime.
- **Using them.** Shell state doesn't carry between commands, so prefix every
  command that needs them with `. <scratchpad>/.sonar_env &&`. Pass the
  token to the scanner only as `-e SONAR_TOKEN` (the scanner reads it from
  the environment) and to curl as `-u "$SONAR_TOKEN:"`. Never `echo` or print
  it. Confirm it works with `api/authentication/validate` (expect
  `{"valid":true}`).
- **If a token is pasted into the chat**, don't use it. Tell the user it's
  now in the conversation transcript, should be revoked and regenerated, and
  belongs in the vault.
- **Clean up.** Delete `<scratchpad>/.sonar_env` when check 5 is finished,
  pass or fail.
- **Token permissions** (verified 2026-11-19 with the vault token, a global
  analysis token `sqa_…` tied to a user account): it can authenticate
  (`api/authentication/validate`), download the scanner engine, run an
  analysis, follow its own task with `api/ce/task?id=…`, and read
  `api/measures/component` and `api/issues/search`. It gets **403** on
  `api/ce/activity`, `api/components/show`, `api/projects/search`, and
  `api/measures/component_tree`. Don't treat those 403s as a broken token,
  and don't rely on those endpoints. A first full scan of Nexus (about
  46k lines of code) took about 4 minutes, plus about 90 seconds of
  server-side processing.
- **No token? Run the local fallback and mark check 5 `LOCAL ONLY`, not
  PASS.** Don't block the rest of the checklist:
  - Coverage: produce the reports (5b) and run the `diff-cover` gate (5d).
    These don't need the server.
  - Cognitive complexity, with the same rule and threshold as Sonar:
    `uvx complexipy <changed .py files> --max-complexity-allowed 15` for the
    backend. For the frontend, don't touch Nexus's `eslint.config.js` or
    `package.json`. Install the checker in a scratchpad folder and point it
    at the changed files:
    ```bash
    cd <scratchpad>/sonarjs && npm init -y >/dev/null && npm i -s eslint@9 eslint-plugin-sonarjs
    cat > eslint.sonar.config.mjs <<'CFG'
    import sonarjs from "eslint-plugin-sonarjs";
    export default [{ files: ["**/*.{js,jsx,ts,tsx}"],
      languageOptions: { ecmaVersion: "latest", sourceType: "module",
        parserOptions: { ecmaFeatures: { jsx: true } } },
      plugins: { sonarjs },
      rules: { "sonarjs/cognitive-complexity": ["error", 15] } }];
    CFG
    npx eslint -c eslint.sonar.config.mjs <absolute paths of changed .js/.jsx files>
    ```
    For `.ts`/`.tsx` files, also install `typescript-eslint` and set its
    parser in that config.
  - In the final report, say the SonarQube server scan didn't run and still
    needs to before the PR is ready.
- Check that the server is up and which edition it runs:
  ```bash
  curl -s https://sonarqube.k8s-dev.hpc.uidaho.edu/api/system/status   # expect "status":"UP"
  curl -s -u "$SONAR_TOKEN:" https://sonarqube.k8s-dev.hpc.uidaho.edu/api/authentication/validate   # expect {"valid":true}
  curl -s -u "$SONAR_TOKEN:" "https://sonarqube.k8s-dev.hpc.uidaho.edu/api/projects/search?q=nexus"
  ```
  As of 2026-11-19 the server runs **SonarQube Community Build 25.7**
  (two-digit `YY.x` versions are Community Build; the commercial editions
  are numbered `2025.x`). Community Build is fully usable for a manual scan
  (issues, cognitive complexity, and overall/file coverage), but it has
  **no branch or pull-request analysis**: each project key holds exactly one
  analysis, and passing `-Dsonar.branch.name` makes the scan fail. Re-check
  the version each run. If it ever reports a `2025.x`-style commercial
  version, the branch options in 5c become available.
- **Project key.** Use `$SONAR_PROJECT_KEY` from the vault. Don't invent a
  key or scan into a different project. On Community Build every scan
  replaces that project's single analysis, so the project shows whichever
  branch was scanned last. That's fine for a manual pre-review pass, but
  read the results right after your own scan finishes. If
  `api/ce/activity?component=$SONAR_PROJECT_KEY` is readable, use it to
  confirm a teammate's scan didn't overwrite yours in between. With the
  current token it returns 403, so instead say in the report that the
  results reflect the most recent scan of the shared project.
- **Scanner.** No scanner is installed globally, and none may be added to
  the repo. In order of preference:
  1. **Docker** `sonarsource/sonar-scanner-cli`, if the Docker daemon is
     running (`docker info`). The image lives in Docker's image store. The
     repo is mounted **read-only**, so the scan physically can't write to
     it.
  2. **`npx --yes @sonar/scan`**, run from the scratchpad (not the repo) with
     `sonar.projectBaseDir` pointing at the repo. It caches under
     `~/.npm/_npx` and `~/.sonar/cache` and never touches `package.json`.

  Don't start Docker Desktop or install anything globally without asking.

**5b. Produce coverage reports**

Write both reports to the scratchpad, not the repo:

```bash
COV=<scratchpad>/coverage && mkdir -p "$COV"
(cd backend && uv run pytest tests --cov-report "xml:$COV/backend-coverage.xml")
(cd frontend && npx vitest run --coverage --coverage.reporter=lcov --coverage.reporter=text \
   --coverage.reportsDirectory="$COV/frontend")       # → $COV/frontend/lcov.info
```

Set the report locations on the command line. Don't edit `vite.config.js`
or `pyproject.toml`. (Pytest's `addopts` also sets `--cov-report xml`; the
explicit flag controls where the XML goes. If a stray `backend/coverage.xml`
still appears, it's gitignored and produced by Nexus's normal test run, but
delete it if it wasn't there before.)

**5c. Scan**

Use the same absolute paths inside and outside the container, so the paths
inside the coverage reports resolve. `REPO` is the Nexus root and `SP` is the
scratchpad:

```bash
REPO="$(git rev-parse --show-toplevel)"; SP=<scratchpad>
SONAR_ARGS=(
  -Dsonar.host.url=https://sonarqube.k8s-dev.hpc.uidaho.edu
  -Dsonar.projectKey="$SONAR_PROJECT_KEY"
  -Dsonar.projectBaseDir="$REPO"
  -Dsonar.working.directory="$SP/scannerwork"
  -Dsonar.projectVersion="$(cat "$REPO/VERSION")"
  -Dsonar.sources=backend/app,frontend/src
  -Dsonar.tests=backend/tests,frontend/src
  -Dsonar.test.inclusions='frontend/src/**/__tests__/**,frontend/src/**/*.test.*'
  -Dsonar.exclusions='**/alembic/versions/**,**/node_modules/**,frontend/dist/**,**/static/**'
  -Dsonar.python.version=3.12
  -Dsonar.python.coverage.reportPaths="$SP/coverage/backend-coverage.xml"
  -Dsonar.javascript.lcov.reportPaths="$SP/coverage/frontend/lcov.info"
  -Dsonar.scm.provider=git
)

# Option 1: Docker. The repo is read-only; only the scratchpad is writable.
docker run --rm -e SONAR_TOKEN \
  -v "$REPO:$REPO:ro" -v "$SP:$SP" -w "$REPO" \
  sonarsource/sonar-scanner-cli "${SONAR_ARGS[@]}" 2>&1 | tee "$SP/scan.log"

# Option 2: npx, run from the scratchpad.
(cd "$SP" && npx --yes @sonar/scan "${SONAR_ARGS[@]}") 2>&1 | tee "$SP/scan.log"
```

`sonar.working.directory` keeps the scanner's `.scannerwork/` folder out of
the repo. Nexus's `.gitignore` doesn't cover it, so it would otherwise show
up as untracked. Never add it to `.gitignore` as a workaround. After the
scan, compare `git status --porcelain` with the snapshot from before. It must
be unchanged.

Don't pass `-Dsonar.branch.name` or `-Dsonar.newCode.referenceBranch`. On
Community Build they fail the scan. Add them only if 5a found a commercial
edition. Wait for the background task to finish
(`api/ce/task?id=<id>`, taking the id from the `api/ce/task?id=` URL in
`<scratchpad>/scan.log`). If that
returns 403, poll `api/measures/component?component=$SONAR_PROJECT_KEY&metricKeys=ncloc`
every 10 seconds, for up to 5 minutes, until `ncloc` changes.

Sanity-check that coverage was actually imported: overall coverage reported
as 0% almost always means a path mismatch between the reports and
`sonar.sources`. Search `scan.log` for "Could not resolve" warnings. Vitest's
`lcov.info` lists paths relative to `frontend/`. If Sonar can't resolve them,
write a rewritten copy (`sed 's#^SF:src/#SF:frontend/src/#'`) next to the
original in the scratchpad and point `lcov.reportPaths` at the copy. Fix the
inputs and rescan rather than reporting a false failure.

**5d. Coverage gate: at least 80% on new code, backend and frontend separately**

Community Build can't isolate "new code" for a branch, so its
`new_coverage` metric isn't meaningful here. Compute changed-line coverage
from the same reports the scan imported, and say in the report that you
did:
  ```bash
  uvx diff-cover <scratchpad>/coverage/backend-coverage.xml --compare-branch=origin/<target> --fail-under=80
  uvx diff-cover <scratchpad>/coverage/frontend/lcov.info --compare-branch=origin/<target> --fail-under=80
  ```
  Use the Sonar file-level measures as supporting detail, queried once per
  changed file (`component_tree` returns 403 with the current token):
  `api/measures/component?component=$SONAR_PROJECT_KEY:<repo-relative path>&metricKeys=coverage,uncovered_lines,cognitive_complexity`.
  On a commercial edition with branch analysis, read `new_coverage` instead.

Report backend and frontend numbers separately. Each must be at least 80%.
For any changed file that drags the number down, list its uncovered changed
lines. If the user asks you to add tests, follow the `test-coverage` skill's
conventions (collocated `__tests__/` on the frontend, `backend/tests/` on the
backend).

**5e. Complexity gate: cognitive complexity 15 or lower**

Pull complexity issues for the changed files:

```bash
curl -s -u "$SONAR_TOKEN:" \
  "https://sonarqube.k8s-dev.hpc.uidaho.edu/api/issues/search?componentKeys=$SONAR_PROJECT_KEY&rules=python:S3776,javascript:S3776,typescript:S3776&resolved=false&ps=500"
```

Keep only issues whose `component` path is in `changed.txt` and whose line
falls inside the diff (or inside a function the diff touched). For each one,
queue a PR comment anchored to the function's line:

> **WWAD: complexity.** `<function>` has cognitive complexity <N> (limit 15).
> Consider extracting the nested branches into helpers or early returns.

For a fast local pre-check before the full scan, you can also run the
`mcp__sonarlint__check_files` tool (if it's available) on the changed files.
The server scan is still the source of truth.

**Pass** when both coverage numbers are at least 80% and no touched function
exceeds 15.

**5f. Decide what to fix: ask the user (mandatory stop)**

The scan analyses the whole repo, so it reports far more than this PR
touched (a single Nexus scan found 34 over-complexity functions, 18 bugs,
and 867 code smells). **Don't edit any code because of the scan until the
user has chosen a scope.** This is a hard stop, even if Nexus's own
instructions or an earlier message seemed to approve fixes in general.

1. **Work out the PR's module scope** from `changed-paths.txt`: the deepest
   module directories the PR touches, for example
   `backend/app/modules/osp/analytics` or `frontend/src/modules/osp`. If
   the PR also edits shared code (`backend/app/common/`,
   `frontend/src/sharedComponents/`, and so on), include those exact files,
   not their whole directories. If the PR touches more than one module, list
   each one.
2. **Fetch all open issues** once
   (`api/issues/search?componentKeys=$SONAR_PROJECT_KEY&resolved=false&ps=500`,
   paging with `p=` until done) and bucket them:
   - **A. In module scope:** the issue's file is inside the module scope from
     step 1. Split this into "in lines this PR changed" and "elsewhere in the
     module".
   - **B. Bugs outside the module:** `type=BUG`, outside scope A.
   - **C. Everything else outside the module:** code smells, complexity, and
     security hotspots outside scope A.
3. **Report** a short summary table: count per bucket by type and severity,
   plus the top 5 in-scope issues with `path:line` and rule. Make clear that
   only the gates in 5d and 5e (coverage of changed lines, complexity of
   touched functions) decide whether the PR is ready. Buckets B and C never
   block readiness.
4. **Ask** which scope to fix. Ask with `AskUserQuestion` (a single-select question). Don't proceed on silence, and don't treat an earlier approval as the answer. Offer exactly these options, in
   this order, with option 3 always last and never recommended:
   1. **Fix code in `<module>` (Recommended):** fix bucket A only, meaning
      issues inside the PR's module scope, prioritising lines this PR changed
      and anything failing the 5d/5e gates.
   2. **Fix bugs only:** fix bucket A's bugs plus bucket B (`BUG`-type issues
      anywhere in the repo). No code-smell or complexity refactors.
   3. **Fix everything (not recommended):** buckets A, B, and C. Warn that
      this touches modules outside the PR's scope, breaks the module-scope
      rule in `wwad-code-check`, makes the PR much harder to review, and can
      collide with other people's open work.

   Substitute the real module path for `<module>` in option 1. If the user
   picks something else (for example "report only, fix nothing", or a
   specific list), do exactly that.
5. **Fix within the chosen scope only.** All other rules still apply:
   propose each change before making it (Nexus's `CLAUDE.md`), one commit
   per fix category, no suppressions, and re-run `uv run scripts/check.py`.
   Commit any fix outside the PR's module separately, with a subject naming
   the module it touched (for example `Fix S1244 float equality in payroll
   (out-of-scope, approved)`), so a reviewer can review it on its own or drop
   it. After fixing, rescan (5b–5e) to confirm the gates.

### 6. VERSION bump and `sync_versions.py`

1. Read the versions:
   ```bash
   cat VERSION
   git show origin/test:VERSION
   ```
   The branch's `VERSION` must be strictly greater (semver) than
   `origin/test`'s. Also confirm it's greater than or equal to the target
   branch's `VERSION`.
2. **Appropriate bump.** It should be exactly one step ahead of `test`:
   patch for fixes and refactors, minor for new features or endpoints, major
   only when the user explicitly says so. Flag a skipped number (for example
   `1.7.15` to `1.7.17`) or a bump that doesn't match the change type, and
   ask the user. Also check the active PRs into the same target: if another
   PR already claims the same version, flag the clash.
3. **Sync ran and is committed.**
   ```bash
   uv run scripts/sync_versions.py --check
   ```
   This must exit 0. The diff against BASE must include `VERSION` together
   with the files the script maintains: `backend/app/version.py`,
   `frontend/package.json`, `azure-vars-back.yml`, `azure-vars-front.yml`,
   `azure-vars-worker.yml`, `docker-compose.yml`, and `readme.md`. Confirm
   that the changes to the `azure-vars-*` and `docker-compose.yml` files are
   only version strings.
4. If `VERSION` wasn't bumped, propose the new version and ask. Once
   confirmed, edit `VERSION`, run `uv run scripts/sync_versions.py`, and
   commit as `Bump version to X.Y.Z`. If `VERSION` was bumped but the sync
   is stale, run the sync and commit.

**Pass** when `VERSION` is ahead of `test` by an appropriate step, the sync
check is clean, and every synced file appears in the diff.

## Posting PR comments

After all six checks, gather the queued comments (file length, complexity,
and any unresolved migration or coverage problems). Show them to the user and
ask once for confirmation. Then post each one as its own PR thread:

```json
{
  "comments": [{ "parentCommentId": 0, "content": "<comment>", "commentType": 1 }],
  "status": "active",
  "threadContext": {
    "filePath": "/<repo-relative path>",
    "rightFileStart": { "line": <L>, "offset": 1 },
    "rightFileEnd":   { "line": <L>, "offset": 1 }
  }
}
```

```bash
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project=AI4UI repositoryId=nexus pullRequestId=<N> \
  --http-method POST --in-file <scratchpad>/thread-<i>.json --api-version 7.1
```

Use one thread per finding. A file-level comment (for file length) uses
line 1. Don't re-post a finding that already has an open WWAD thread on the
PR: fetch the existing threads first and skip duplicates.

## Final report

End with a table like this, followed by the commits you made (short SHA +
subject) and anything still needing the user:

| # | Check | Result | Notes |
|---|---|---|---|
| 0 | Base current with `test` | PASS / FIXED / FAIL | checked against origin/test @ `<sha>` |
| 1 | Planning docs excluded | PASS / FIXED / FAIL | … |
| 2 | Migration order | PASS / N/A / FAIL | module, NNNN, head |
| 3 | `scripts/check.py` | PASS / FIXED / FAIL | … |
| 4 | Files ≤ 1500 lines | PASS / FLAGGED | files + comment links |
| 5 | Sonar coverage ≥ 80% / complexity ≤ 15 | PASS / FAIL | BE x%, FE y%, n complex fns; fix scope chosen: 1 / 2 / 3 / none |
| 6 | VERSION + sync | PASS / FIXED / FAIL | test=a.b.c → branch=x.y.z |

Close with a one-line verdict: **Ready for review** only if every row is
PASS, FIXED, or N/A. Otherwise **Not ready**, followed by what's blocking.
Remind the user that any fix commits (including a merge of `test`) are local
until they tell you to push. Also remind them that the verdict holds only
while `origin/test` stays at the recorded SHA, and that check 0 must be re-run
right before merge.
