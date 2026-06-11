---
name: clean-code
description: Safely clean up a project codebase — remove dead/deprecated code, strip commented-out blocks, auto-format, and fix lint errors — without breaking behavior. Detects project language (Python → ruff, JS/TS → eslint/prettier), works on a dedicated refactor branch with one commit per phase, asks before deleting anything non-trivial, and verifies with the existing test suite at the end. Use when the user invokes `/clean-code`, asks to "clean up the codebase", "remove dead code", "run ruff and fix lint", "tidy this project", or similar housekeeping requests.
---

# Clean code

You are doing a careful, low-risk housekeeping pass on the project. The
goal is a smaller, tidier codebase that **behaves identically** to the
one you started with. Behavior preservation beats thoroughness — if a
cleanup is ambiguous, leave it and flag it.

## Non-negotiables

- **Work on a dedicated branch.** Never clean directly on `main` /
  `master` / `develop`. Create the branch, switch to it, and stage
  commits there. Branch name format:
  `refactor/<git-user>-cleanup-<MM>-<DD>-<YYYY>`
  where `<git-user>` is `git config user.name` lowercased with spaces
  replaced by `-`, and the date is today. Example:
  `refactor/jarred-kvistad-cleanup-05-22-2026`.
- **Never delete non-trivial code without confirming.** Auto-deletable:
  unused imports, unused local variables, trivially unreachable code
  (e.g. code after `return`/`raise`), `console.log`/`print` debug
  statements that are obviously stray. Anything else (unused functions,
  classes, modules, exported symbols, dead branches longer than a few
  lines) → list it and ask.
- **The existing test suite must pass at the end.** If there is no
  test suite, say so explicitly and stop before declaring success.
- **No behavior changes.** Don't "improve" logic, rename public APIs,
  reorder arguments, or refactor structure. Cleanup only.
- **One commit per phase** on the refactor branch, so any phase can be
  reverted independently. Do not squash.

## Procedure

### 1. Pre-flight

1. Confirm a clean working tree: `git status --porcelain`. If dirty,
   ask the user to commit or stash before continuing.
2. Detect project language(s) by inspecting the repo (e.g.
   `pyproject.toml` / `*.py` → Python; `package.json` / `*.ts` /
   `*.tsx` / `*.js` / `*.jsx` → JS/TS). A repo can be both.
3. Detect tooling and test runner:
   - Python: `ruff` (check `pyproject.toml`/`ruff.toml`), test runner
     (`pytest`, `unittest`, `tox`, or whatever `pyproject.toml` /
     `Makefile` / CI config indicates).
   - JS/TS: `eslint` + `prettier` (check `package.json` scripts and
     config files), test runner (`npm test`, `pnpm test`, `yarn test`,
     `vitest`, `jest`).
4. If a required tool isn't installed, tell the user the exact install
   command and stop. Don't install global tooling on the user's
   machine without asking.
5. **Baseline.** Run the test suite *before* any changes and record
   the result. If it's already failing, stop and report — don't paper
   over pre-existing breakage.
6. **Create and switch to the refactor branch now**, before any edits:
   `git checkout -b refactor/<git-user>-cleanup-<MM>-<DD>-<YYYY>`.
   All subsequent phase commits go on this branch.

### 2. Phase 1 — Dead & deprecated code (conservative)

For each language detected, gather candidates:

- **Python:** run `ruff check --select F401,F841,F811,F823` (unused
  imports, unused vars, redefinitions) for the auto-safe set.
  Separately, scan for functions/classes/constants with zero
  references, `@deprecated`-marked symbols, modules not imported
  anywhere, and `TODO: remove`/`# DEPRECATED` markers.
- **JS/TS:** run `eslint` with `no-unused-vars` / `@typescript-eslint/
  no-unused-vars`. Use `ts-prune` or `knip` if available for unused
  exports. Scan for `@deprecated` JSDoc tags.

Then:

1. **Auto-apply** removals only for: unused imports, unused locals,
   trivially unreachable statements, stray debug prints/logs.
2. **List everything else** with file:line and a one-line "why this
   looks dead" rationale, and ask the user which to remove. Group by
   confidence (high / medium / low) so they can bulk-approve the
   obvious ones.
3. Watch out for false positives: dynamic imports, string-based
   lookups (`getattr`, `globals()`, `require(name)`), framework
   entrypoints (Flask routes, Django URLs, pytest fixtures, React
   component exports consumed by name), public-API exports.
4. Commit on the refactor branch: `chore: remove dead code (clean-code phase 1)`.

### 3. Phase 2 — Commented-out code blocks

Target **blocks of commented-out code**, not legitimate comments.
Heuristics for "this is commented-out code, not a note":

- Multiple consecutive comment lines that parse as code in the host
  language (contain `=`, `(`, `)`, `def`, `function`, `import`, etc.).
- Comments containing a former code line followed by `# TODO: revisit`
  / `// old impl` / similar.

**Keep** comments that:

- Explain *why* something is done a certain way.
- Reference an issue, ticket, RFC, or external doc.
- Are docstrings or JSDoc.
- Mark a workaround, gotcha, or invariant.

When in doubt, keep it and flag for the user. Commit on the refactor
branch: `chore: remove commented-out code (clean-code phase 2)`.

### 4. Phase 3 — Auto-format

- **Python:** `ruff format .` (and `ruff format --check .` to verify).
- **JS/TS:** `npx prettier --write .` (respect `.prettierignore`).
  If the project has a `format` script, prefer that.

Commit on the refactor branch: `chore: auto-format (clean-code phase 3)`.

### 5. Phase 4 — Lint fixes

- **Python:** `ruff check --fix .` for safe autofixes, then
  `ruff check --fix --unsafe-fixes .` **only after** showing the user
  what unsafe fixes would do and getting confirmation. Remaining
  manual lint errors: fix only those that are clearly mechanical
  (missing type hints in obvious cases, simple naming). Anything
  requiring judgment → list and ask.
- **JS/TS:** `npx eslint . --fix`. Same rule for non-auto fixes —
  list, don't guess.

Don't disable rules to make lint pass. If a rule is wrong for this
codebase, surface it as a question, not a silent `# noqa` /
`// eslint-disable-next-line`.

Commit on the refactor branch: `chore: fix lint errors (clean-code phase 4)`.

### 6. Final verification

Run, in order, and report the result of each:

1. Formatter check (`ruff format --check .` / `prettier --check .`) —
   should be clean.
2. Linter (`ruff check .` / `eslint .`) — should be clean or down to a
   pre-existing baseline you noted.
3. **Full test suite.** Same command you ran in pre-flight.
4. Type checker if the project uses one (`mypy`, `pyright`, `tsc
   --noEmit`) — should not be worse than baseline.

If anything regresses vs. baseline, **do not declare success**.
Identify the offending phase commit, `git revert` it on the refactor
branch (don't rewrite history), re-run verification, and report what
was rolled back and why.

### 7. Hand-off

Summarize for the user:

- Branch name and the fact that you're still on it.
- One-line summary per phase commit (files touched, counts).
- List of items you flagged but did **not** remove, with file:line.
- Test / lint / type-check status vs. baseline.
- Suggested next step (open a PR, merge locally, or review the
  flagged items). Do not push, merge, or switch branches yourself
  unless asked.

## What to avoid

- Running on a dirty working tree.
- Committing to `main` / `master` / `develop` — always the refactor
  branch.
- Squashing the phase commits — granularity is the safety net.
- Touching files outside the project's source tree (vendored deps,
  generated files, lockfiles, migrations).
- "Improving" code while you're in there (renames, extractions,
  reorderings). That's a separate task.
- Hiding lint errors with disable comments instead of fixing or
  asking.
- Declaring success when tests didn't run, or when the project has no
  tests — say so plainly instead.
