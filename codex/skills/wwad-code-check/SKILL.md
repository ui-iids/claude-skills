---
name: wwad-code-check
description: WWAD ("What Would Arpan Do") coding mode for the Nexus production dashboard repo. While it is active, all work stays on a local `dev/*` branch cut from a freshly pulled `test` and kept current with it (never pushing to `test`). Nothing is pushed and no `sprint/*` branch (which triggers Azure pipeline builds) is created until the user says so. Code follows Nexus standards — reuse existing code, keep cognitive complexity at 15 or below, split long scripts into submodules, fail fast with specific typed errors (`ServiceError` subclasses) and no silent failures, respect the backend module layout (`api/v<N>`, `models`, `services`, `utils`, `config.py`, with shared code in `backend/app/common`) and the matching frontend module layout, route all logging through the shared logger, and use explicit types throughout. Use when the user invokes `$wwad-code-check`, says "WWAD mode", "code like Arpan", "Nexus coding standards", "work on Nexus", or starts feature/fix/refactor work in Nexus.
---

# WWAD code check ("What Would Arpan Do")

You are writing code for **Nexus**, a collaborative production dashboard that
the university relies on every day. Reviewers hold it to a high standard. Code
in this mode as if the most careful reviewer on the team will read every line:
small, well-placed, explicit, and boring in the best way.

This is a **mode**: the rules below apply to every change you make for the
rest of the session, not just one pass. When a change is ready for review,
use the companion skill `wwad-pr-checklist` to gate it.

## Scope and precedence

- Applies only to the Nexus repo (`~/Documents/GitHub/nexus`, origin
  `https://dev.azure.com/uidaho/AI4UI/_git/nexus`). If you're in another repo,
  tell the user and don't apply the Nexus-specific rules.
- Read Nexus's own `AGENTS.md` / `CLAUDE.md`, `docs/CodeQuality.md`, and
  `docs/StructuredLogging.md` at the start. Where they're more specific than
  this skill, they win. Nexus's `AGENTS.md`/`CLAUDE.md` says **ask before making any code
  change** and **keep responses short**. Follow both: propose the change
  (files, what, why) and wait for a go-ahead before editing.

## Branch and push rules (hard rules)

1. **Every `dev/*` branch comes from a freshly pulled `test`. No exceptions,
   no other base.** Before the first edit, check `git branch --show-current`.
   If you're not already on the `dev/*` branch for this work, create one:
   ```bash
   git fetch origin test
   # Refresh the local test branch too, fast-forward only. Never merge into it or commit to it.
   if git show-ref --verify --quiet refs/heads/test; then
     if [ "$(git branch --show-current)" = test ]; then git merge --ff-only origin/test
     else git fetch origin test:test; fi        # refuses if local test has diverged
   fi
   git switch --no-track -c dev/<git-user>-<short-kebab-description> origin/test
   ```
   `<git-user>` is `git config user.name` lowercased with spaces replaced by
   `-`. If the fast-forward fails because local `test` has commits that
   aren't on `origin/test`, **stop and ask**. Never push them and never
   reset them away. If the working tree is dirty, ask the user to commit or
   stash first. Never commit directly to `main`, `test`, or any `sprint/*`
   branch.
2. **Never upstream `test`.** Never push to `origin/test` (no
   `git push origin test`, no `git push origin HEAD:test`, no `--force`),
   and never set `test` as a branch's upstream. That's why branching uses
   `--no-track`. Verify with `git rev-parse --abbrev-ref @{u}` (it should
   fail or name `origin/dev/...`, never `origin/test`). Changes reach `test`
   only through the reviewed `sprint/*` PR flow, never directly.
3. **Never push anything** (no `dev/*`, no `sprint/*`, no tags) unless the
   user explicitly tells you to. When told to push a `dev/*` branch, push it
   to its own name: `git push -u origin dev/<suffix>`. Commit locally in
   small, focused commits.
4. **Keep the branch's base current with `test`.** `test` moves often. Before
   any of these:
   - running `$wwad-pr-checklist`
   - promoting to `sprint/*`
   - opening a PR
   - completing (merging) a PR

   make sure the branch contains the current `origin/test`:
   ```bash
   git fetch origin test
   git merge-base --is-ancestor origin/test HEAD && echo "base current" || echo "behind test"
   git rev-list --count HEAD..origin/test      # how many test commits are missing
   ```
   If the branch is behind, bring it up to date by **merging** `origin/test`
   into the `dev/*` branch (`git merge origin/test`). Merging doesn't
   rewrite history, so no force-push is needed. Rebase only if the user asks,
   and if the branch is already pushed, get explicit approval for the
   force-push (`--force-with-lease` to its own `dev/*` name only). Resolve
   conflicts carefully and ask when a resolution isn't obvious. After
   updating, re-run `uv run scripts/check.py` and recheck migration
   numbering and `VERSION` against the new `test`, since either may have
   moved. Never merge or complete a PR whose source branch is behind
   `origin/test`.
5. **Never create a `sprint/*` branch on your own initiative.** Pushing a
   `sprint/*` branch kicks off Azure pipeline builds, so only finished,
   tested work goes there, and only when the user directs it.
6. **Promotion flow, only when the user says so.** When the `dev/*` work is
   complete and tested and the user asks you to promote it:
   1. Bring the base up to date (rule 4). Confirm `scripts/check.py` passes,
      and suggest running `$wwad-pr-checklist` first if it hasn't been run against the
      current base.
   2. Create `sprint/<same-suffix>` (for `dev/jarred-osp-export`, that's
      `sprint/jarred-osp-export`) from the **freshly fetched** `origin/test`,
      with no tracking:
      `git fetch origin test && git branch --no-track sprint/<suffix> origin/test`.
      Because the `dev/*` branch already contains current `test`, the PR diff
      is exactly your work. Basing the sprint branch on the `dev/*` head
      would leave the PR with an empty diff.
   3. Push the `sprint/*` branch and the `dev/*` branch, each to its own name
      (`git push -u origin sprint/<suffix>`, `git push -u origin dev/<suffix>`),
      then open the PR `dev/<suffix>` → `sprint/<suffix>`:
      ```bash
      az repos pr create --source-branch dev/<suffix> --target-branch sprint/<suffix> \
        --title "<title>" --description "<description>" --draft false
      ```
      Each of these steps is outward-facing. Confirm them with the user right
      before running them, even if they approved the plan earlier.
7. **Before a PR is completed (merged),** whether you're asked to complete
   it or the user says it's about to be merged, re-run the rule 4 check
   against a fresh `git fetch origin test`. If `test` moved since the last
   check, update the `dev/*` branch, re-run the checks, and push the update
   (with the user's go-ahead) before anyone merges. Complete a PR only when
   the user explicitly says to.
8. **Never touch CI/CD files** (`azure-pipelines-*.yaml`, `azure-vars-*.yml`,
   `.github/workflows/*`, `Dockerfile.*`, `docker-compose*.yml`) unless the
   user explicitly asks. The one exception is the version-string lines that
   `scripts/sync_versions.py` rewrites.

## Coding principles

### 1. Reuse before you write

Before adding a function, component, hook, model, or service, search for an
existing one: `rg` across `backend/app/common/`, `backend/app/utils/`, the
module you're in, `frontend/src/sharedComponents/`, `frontend/src/utils/`,
and `frontend/src/hooks/`. Extend a shared building block rather than adding
another module-local copy. If you find near-duplicates while working, point
them out instead of creating a third.

### 2. Keep complexity at 15 or below

Every function must stay at **cognitive complexity 15 or below** (SonarQube
rule S3776). Don't pack complex logic into one function. Use early returns
and guard clauses, extract named helpers for nested branches, swap long
`if/elif` chains for lookup tables or dispatch dicts, and keep each function
to one job. If a function you're editing is already over 15, say so and
offer to split it, but don't do a drive-by refactor without asking.

### 3. No giant files

Keep files short and focused. Treat 1500 lines as a hard limit and aim far
below it. When a script grows several unrelated functions, split it into
submodules (for example `services/export/`, with `extract.py`,
`transform.py`, and `load.py`) and import what each caller needs. The
existing `osp/analytics/etl/` package is a good model. Reviewers should be
able to read one file and understand one idea.

### 4. Fail fast, fail loud, fail specifically

- Validate inputs at the boundary (Pydantic models on routes, explicit checks
  at the top of services) and raise immediately on bad state.
- **No silent failures.** Never use a bare `except:`, `except Exception: pass`,
  a swallowed error that returns `None`/`[]`/`{}`, or a `.catch(() => {})` on
  the frontend. If you catch, either handle the case meaningfully or re-raise
  with context (`raise XError(...) from exc`).
- **Use specific, typed errors.** On the backend, raise the matching subclass
  of `ServiceError` from `backend/app/utils/custom_errors.py`
  (`ValidationError` 400, `AuthenticationError` 401, `AuthorizationError`
  403, `NotFoundError` 404, `ConflictError` 409, `PayloadTooLargeError` 413,
  and so on). The global handler in `main.py` maps their `status_code`. If
  none fits, add a narrowly named subclass with the right `status_code`,
  module-local unless other modules need it too. Avoid generic
  `Exception`/`RuntimeError`/`HTTPException(500)`, and write `detail`
  messages that say what failed and why, with no PII.
- On the frontend, surface API errors through the shared toast/error
  patterns (`ToastContext`, TanStack Query `onError`). Don't quietly render
  an empty state when a request has failed.
- After writing a function, reread it and ask: "What happens on every
  failure path? Could any of them return a wrong-but-plausible value without
  anyone noticing?" Fix any path where that could happen.

### 5. Backend module layout

All domain code lives under `backend/app/modules/<module>/`:

```
modules/<module>/
  api/v1/            # FastAPI routers. New breaking resource versions go in api/v2/, …
  models/            # Pydantic request/response schemas (and SQLAlchemy ORM models)
  services/          # Business logic and intermediate tools
  utils/             # Small pure helpers used only by this module
  config.py          # Module-local configuration/settings
  db.py              # Session factory (existing pattern)
  alembic/           # Per-module migrations
```

- **Stay inside the module you're working on.** Working on
  `osp/analytics` means you don't edit `osp/preaward` or `payroll`. If a
  change outside the module is truly needed, stop and explain why before
  touching it.
- **Scan results don't widen scope.** SonarQube, lint, or type-checker
  output often lists problems in other modules. Don't fix them on your own.
  Report them and ask the user which scope to fix, using the three options
  in `$wwad-pr-checklist` step 5f: (1) this module, recommended; (2) bugs only;
  (3) everything, not recommended.
- **Shared code goes in `backend/app/common/`** (`common/services`,
  `common/models`, `common/api`, `common/config.py`). Only move something
  there when two or more modules actually use it.
- **API versioning.** Add endpoints to the module's current version folder.
  A breaking change to an existing resource's contract goes in a new
  `api/v<N+1>/` rather than editing the old one in place. Ask if you're
  unsure whether a change is breaking. Register routers in
  `backend/app/register.py` following the existing pattern.
- Module routes take input and return output through **Pydantic models** in
  `models/`. Don't use raw `dict` request bodies or untyped responses. Set
  `response_model` on routes.
- New migrations follow `<rev>_<NNNN>_<description>.py`, numbered one past
  the module's current head (see `wwad-pr-checklist` check 2).

### 6. Logging goes through the shared logger

- Get loggers with `logger = logging.getLogger(__name__)` so records
  propagate to the root logger that `app.logger.setup_logging` configures
  (JSON formatter, OTel context, PII redaction). Never attach your own
  handlers or formatters, call `logging.basicConfig`, or create a standalone
  logger setup.
- No `print()` in backend code. No stray `console.log` in committed frontend
  code.
- Follow `docs/StructuredLogging.md`: put a constant message plus
  `extra={"event_name": "<domain>.<entity>.<action>", ...}` instead of
  f-string interpolation, and never log PII or secrets.

### 7. Explicit typing everywhere

- **Python:** every function signature gets full parameter and return
  annotations, including `-> None`. Use precise types (`list[InvoiceRow]`,
  `Mapping[str, Decimal]`, `Literal[...]`, `TypedDict`, Pydantic models)
  instead of `Any`, bare `dict`/`list`, or `object`. Add `| None` only when
  `None` is a real, handled value. The code must pass `mypy app` cleanly
  without new `# type: ignore`.
- **Frontend:** follow the existing typing approach in the area you're
  editing. In `.ts`/`.tsx`, use real interfaces and types (no `any`). In
  `.js`/`.jsx`, use the module's `models/` shapes and JSDoc `@param` /
  `@returns` / `@typedef` for public functions, hooks, and component props.
  Name things for what they are, not `data`, `obj`, or `res2`.

### 8. Frontend module layout

`frontend/src/modules/<module>/` mirrors the backend domain split:

```
modules/<module>/
  api/          # API wrappers for this module
  components/   # Module-only components
  pages/        # Routed pages
  hooks/        # Module-only hooks
  models/       # Data shapes / types
  utils/        # Module-only pure helpers
  constants/    # Module-only constants
  __tests__/    # Collocated Vitest tests
```

- Anything used by more than one module goes outside `modules/`:
  `sharedComponents/`, `utils/`, `hooks/`, `contexts/`. There is no `UI/`
  folder.
- Stay inside the module you're working on, the same as on the backend.
- Server state uses TanStack Query. Global state uses the three existing
  contexts. Don't add a new global store without asking.

## Working loop

For each unit of work:

1. **Propose** the change in a few lines: files, placement, which existing
   code you're reusing. Wait for the go-ahead.
2. **Implement** the change following the principles above.
3. **Test.** Add or update tests next to the code (`backend/tests/`,
   collocated frontend `__tests__/`) and aim for at least 80% coverage of new
   lines.
4. **Verify** with `uv run scripts/check.py --fix`, then
   `uv run scripts/check.py`. Fix the root cause of anything that fails.
   Never use `--no-verify` or new suppressions.
5. **Self-review** against this checklist before you say the work is done:
   - [ ] On a `dev/*` branch cut from freshly pulled `test`, base still current with `origin/test`, nothing pushed, nothing upstreamed to `test`
   - [ ] Reused existing code where it existed
   - [ ] No function over complexity 15, no file approaching 1500 lines
   - [ ] Every error path raises a specific error; nothing fails silently
   - [ ] Files are in the right module folder; nothing outside scope touched
   - [ ] Logging uses `getLogger(__name__)` with structured `extra`
   - [ ] Full type annotations; no `Any`/`any` or ambiguous types
   - [ ] No CI/CD files touched; no planning docs staged
6. **Commit** locally with a descriptive message, then report briefly what
   changed. Don't push.
