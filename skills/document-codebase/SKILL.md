---
name: document-codebase
description: Document an existing project by (1) adding detailed inline comments to source files, (2) creating or updating architecture docs under `docs/` (frontend, backend, services), and (3) rewriting the root `README.md` with install/run instructions, a service summary, and a complete env-var inventory backed by a `.env.template` (or `.env.example`). Reads container compose files (Podman/Docker) for run instructions, and greps the codebase for every env-var reference so nothing is missed. Use when the user invokes `/document-codebase`, asks to "document this project", "write docs for the codebase", "generate a README", or "create a .env template".
---

# Document codebase

You are producing **durable documentation** for an existing project. The
deliverable has three parts — inline code comments, architecture docs
under `docs/`, and an authoritative root `README.md` + env template —
and they must agree with each other and with the code as it exists
today. Accuracy beats completeness: if you cannot verify something from
the codebase, say so or omit it rather than guess.

## Non-negotiables

- **Work on a dedicated branch.** Never document directly on `main` /
  `master` / `develop`. Create and switch to:
  `docs/<git-user>-document-<MM>-<DD>-<YYYY>`
  where `<git-user>` is `git config user.name` lowercased with spaces
  replaced by `-`, and the date is today.
- **One commit per phase** so any phase can be reverted independently.
  Do not squash.
- **No behavior changes.** Comments and docs only. Do not refactor,
  rename, reformat, or "improve" code while documenting it. The single
  exception is creating `.env.template` / `.env.example` if missing.
- **Verify before you write.** Every env var, port, service name, run
  command, and dependency in the README must come from a file you
  actually read — compose file, package manifest, source code, or
  config. If something can't be verified, mark it `TODO(verify)` rather
  than inventing it.
- **Don't overwrite intent.** If a file already has meaningful
  docstrings or a README has hand-written sections, preserve them.
  Augment, don't replace.

## Procedure

### 1. Pre-flight

1. Confirm a clean working tree: `git status --porcelain`. If dirty,
   ask the user to commit or stash before continuing.
2. Detect project shape:
   - Language(s): `pyproject.toml` / `requirements*.txt` / `*.py` →
     Python; `package.json` / `*.ts` / `*.tsx` / `*.js` / `*.jsx` →
     JS/TS; `go.mod` → Go; `Cargo.toml` → Rust; etc.
   - Frontend vs backend split: look for `frontend/`, `client/`, `web/`,
     `app/` (frontend) and `backend/`, `server/`, `api/`, `services/`
     (backend). Note monorepo layouts.
   - Container/orchestration files: `Dockerfile*`, `docker-compose*.y*ml`,
     `podman-compose*.y*ml`, `compose.y*ml`, `Containerfile`,
     `.devcontainer/`, `k8s/`, `helm/`.
   - Existing docs: `README*`, `docs/`, `CONTRIBUTING*`, `ARCHITECTURE*`.
   - Existing env templates: `.env.template`, `.env.example`,
     `.env.sample`, `.env.dist`.
3. **Create and switch to the docs branch now**, before any edits:
   `git checkout -b docs/<git-user>-document-<MM>-<DD>-<YYYY>`.
4. Ensure `docs/` exists at the project root; create it if not.

### 2. Phase 1 — Codebase discovery

Build a mental model you can cite from. Do this **before writing
anything**.

1. **Entry points.** Find `main` / `if __name__ == "__main__"` /
   `package.json` `scripts` / `Makefile` / `Procfile` / compose
   `command:` — anything that starts a process.
2. **Services.** From compose files, enumerate every service: name,
   image, ports, depends_on, volumes, env vars consumed. Cross-check
   against any standalone `Dockerfile`s.
3. **Dependencies.** Read `package.json`, `pyproject.toml` /
   `requirements*.txt`, `go.mod`, `Cargo.toml`. Note runtime vs dev
   deps. Note explicit version pins for things mentioned in the README
   (DB version, Node version, Python version).
4. **Env vars — exhaustive grep.** This is the most error-prone part,
   so be thorough. Search for every reference pattern:
   - Python: `os.environ`, `os.getenv`, `environ.get`, `getenv(`,
     `Settings(` / `BaseSettings` field names (pydantic).
   - JS/TS: `process.env.`, `import.meta.env.`, `Deno.env.get`,
     `Bun.env`.
   - Shell / compose: `${VAR}`, `$VAR`, `env_file:`, `environment:`.
   - Config files: `.envrc`, `direnv`, `dotenv`, framework-specific
     `settings.py` / `config.ts`.
   - Frameworks: Django `settings.py`, Flask `app.config[...]`,
     FastAPI `Settings`, Next.js `NEXT_PUBLIC_*` / `next.config.js`,
     Vite `VITE_*`.
   Build a single deduplicated list with: name, where it's read
   (`file:line`), whether it has a default, whether it appears
   required, and inferred purpose. Keep this list — you'll use it in
   phase 4 and phase 5.
5. **Architecture map.** Sketch (in scratch notes, not committed) how
   frontend → backend → services → datastores flow. Note auth,
   background workers, scheduled jobs, external APIs.

No commit yet — this phase is research.

### 3. Phase 2 — Inline comments

Add comments to source files **where they add information that isn't
obvious from the code**. Do not narrate.

**Add comments for:**

- Public functions / classes / modules that lack docstrings — write a
  concise docstring (one-line summary + args/returns if non-trivial).
  Use the project's existing docstring style (Google, NumPy, JSDoc,
  TSDoc) — match what's already there.
- Non-obvious *why*: workarounds, ordering constraints, performance
  tricks, security-sensitive checks, framework-specific gotchas,
  surprising defaults.
- Complex algorithms or state machines: a short header comment
  describing the invariant or the high-level steps.
- Public API boundaries: route handlers, exported functions, CLI
  entrypoints, message handlers.

**Do not add:**

- Comments that restate the code (`# increment i by 1`).
- Section banners (`# ===== HELPERS =====`).
- Author/date tags — git blame already has this.
- TODO/FIXME you can't substantiate.
- Comments inside trivial getters/setters/one-liners.

Walk the codebase systematically: per top-level package/module, read
the file, decide what comments are missing, edit. Skip vendored code,
generated files, migrations, and lockfiles.

Commit: `docs: add inline comments (document-codebase phase 2)`.

### 4. Phase 3 — `docs/` architecture documentation

Create or update markdown files under `docs/`. Use these filenames
unless the project already has its own convention — in which case
match it.

- `docs/architecture.md` — high-level overview: a diagram (mermaid is
  fine) or ASCII sketch of services and their relationships, request
  lifecycle, data flow, where state lives.
- `docs/frontend.md` — only if a frontend exists. Framework, entry
  point, routing approach, state management, build tooling, where API
  calls originate, env var conventions (e.g. `NEXT_PUBLIC_*`,
  `VITE_*`).
- `docs/backend.md` — only if a backend exists. Framework, entry
  point, routing/handler layout, ORM/data layer, auth model,
  background jobs, task queue.
- `docs/services.md` — every external/in-compose service: purpose,
  image/version, exposed ports, how the app talks to it, what data it
  holds, env vars needed to reach it. One section per service.
- `docs/development.md` — local dev loop: how to run tests, how to
  run the app in watch mode, how to seed data, how to reset state.
  Optional if the README already covers this fully.

Rules for these docs:

- Cite real file paths (`backend/api/users.py:42`) so the reader can
  jump into the code.
- Don't duplicate the README. The README is the front door (install +
  run + summary); `docs/` is the depth. Link from README to relevant
  `docs/` pages.
- If a doc would be empty or just a stub, **don't create it**. Better
  to have four real docs than seven thin ones.
- Preserve any existing `docs/` content — read it first, update only
  the stale parts, and note in your hand-off which sections you
  rewrote vs. left alone.

Commit: `docs: add architecture docs under docs/ (document-codebase phase 3)`.

### 5. Phase 4 — Env template

Using the exhaustive env-var list from phase 1:

1. If `.env.template` / `.env.example` / `.env.sample` exists, read
   it and reconcile with the discovered list:
   - Add any var the codebase references that's missing from the
     template.
   - Flag (don't auto-remove) vars in the template that the codebase
     no longer references — list them in your hand-off for the user
     to decide.
   - Keep the existing file's name and ordering style.
2. If none exists, create `.env.template` (preferred name unless the
   project's framework dictates otherwise, e.g. Next.js conventions).
3. For each var, write:
   ```
   # <one-line purpose>. Read at <file:line>.
   # Required: <yes|no>. Default: <value or "none">.
   VAR_NAME=
   ```
   Leave the value empty unless it's a safe non-secret default (e.g.
   `LOG_LEVEL=info`). **Never** put a real secret, API key, password,
   or token in the template — even if you find one in a developer's
   local `.env`. If you encounter a real secret while grepping, flag
   it to the user and do not copy it anywhere.
4. Group vars by service/concern with short section headers (`# ---
   Database ---`).
5. If `.gitignore` doesn't ignore `.env`, add it (but not the
   template). Confirm before changing `.gitignore` if the project's
   ignore strategy is unusual.

Commit: `docs: add/update env template (document-codebase phase 4)`.

### 6. Phase 5 — Root `README.md`

The README is the front door. Rewrite (or augment, preserving
hand-written sections) so it has, in this order:

1. **Project name + one-paragraph summary** — what it is, who it's
   for, the problem it solves. Derive from `pyproject.toml` /
   `package.json` description, existing docs, and the code itself.
2. **Architecture at a glance** — 2–4 sentences naming the major
   pieces (frontend framework, backend framework, datastores, key
   external services). Link to `docs/architecture.md`.
3. **Dependent services** — bulleted list of every service the app
   needs to run (DB, cache, queue, object store, third-party APIs),
   each with its role and where it's configured. Link to
   `docs/services.md`.
4. **Prerequisites** — language runtimes with versions, container
   runtime (Docker / Podman) if used, package manager, anything else
   the user must have installed first. Pull versions from `engines`,
   `python_requires`, `.tool-versions`, `.nvmrc`, `Dockerfile` base
   images.
5. **Installation** — exact commands, copy-pasteable, in the order
   they must run. Cover dependency install and any one-time setup
   (DB migrations, seed data, building static assets).
6. **Configuration** — point to `.env.template`, explain "copy this
   to `.env` and fill in the blanks", list which vars are *required*
   to start the app vs optional. Link to the template file, do not
   re-list every var inline.
7. **Running the app** — at least two paths if applicable:
   - Containerized: `docker compose up` or `podman-compose up`, with
     the exact compose file path. Note which services start, which
     ports they bind, and how to reach the app (e.g.
     `http://localhost:3000`).
   - Local / bare-metal: how to start each process (frontend,
     backend, worker) in its own terminal.
   Pull commands from the actual compose / `Makefile` /
   `package.json` `scripts` — don't fabricate.
8. **Testing** — how to run the test suite.
9. **Project layout** — short tree (top 2 levels) with one-line
   descriptions per top-level dir. Link to `docs/` for deeper
   reading.
10. **Further reading** — links into `docs/`.

Rules:

- If a section already exists and is correct, leave it alone — only
  rewrite stale or missing parts.
- Use fenced code blocks with the language tag for all commands.
- Don't include badges, ASCII-art logos, contributor lists, or
  license boilerplate unless the user asks — focus on the install/run
  story.
- If the project is a library (not an app), adapt: emphasize install
  + usage example over compose / services.

Commit: `docs: rewrite README with install/run/env (document-codebase phase 5)`.

### 7. Final verification

Run, in order, and report:

1. **Spot-check the env vars.** Re-grep one or two patterns from the
   phase-1 list and confirm every hit is in the template.
2. **Compose-up dry run** (if a compose file exists and the user is
   comfortable): `docker compose config` or `podman-compose config`
   — should parse cleanly. Don't actually `up` services without
   asking.
3. **Markdown link check.** Open each new/edited `.md` and confirm
   internal links (to other docs, to source files) resolve.
4. **Tests still pass.** Inline comments shouldn't break anything,
   but run the suite to be sure. If the project has no tests, say so
   plainly.

If anything regresses, identify the phase commit, `git revert` it on
the docs branch, and report.

### 8. Hand-off

Summarize for the user:

- Branch name and the fact that you're still on it.
- One-line summary per phase commit (files touched, counts).
- Env vars added to the template; env vars in the template that the
  codebase no longer references (flagged, not removed).
- `docs/` files created vs updated vs left alone.
- README sections rewritten vs preserved.
- Anything you marked `TODO(verify)` — these are the spots where you
  couldn't confirm from the code alone and need the user's input.
- Suggested next step (open a PR, fill in TODOs, merge locally). Do
  not push, merge, or switch branches yourself unless asked.

## What to avoid

- Running on a dirty working tree.
- Committing to `main` / `master` / `develop` — always the docs
  branch.
- Squashing the phase commits — granularity is the safety net.
- Inventing env vars, ports, service names, or commands. If you can't
  cite a file, don't write it.
- Copying real secrets out of a local `.env` into the template.
- Replacing hand-written README or docs sections wholesale when you
  could augment them.
- Writing comments that just restate the code.
- Creating placeholder doc files (`docs/foo.md` with a heading and
  nothing else) to look thorough.
- Declaring success without verifying the env-var inventory or that
  the test suite still passes.
