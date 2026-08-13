---
name: review-pr
description: Perform a thorough code review of a GitHub pull request — the diff read in full-codebase context — covering bugs, logic errors, code quality, lint/typing, complexity, security, exposed secrets, dead/commented-out code, and stale or misleading comments, then post each finding as an inline PR comment (line-anchored, with a fenced code snippet and a fix) plus a summary review. Use when the user asks to "review PR <N>", "review this PR and post comments", "do a full PR review", or similar.
---

# Review PR

Do a deep, adversarial code review of a GitHub pull request and publish the
findings back to GitHub as review comments. The diff is the subject, but you
read it **in the context of the whole codebase** — a line can be correct in
isolation and wrong given how the rest of the repo uses it.

The output is a single GitHub review: one inline comment per concrete finding
(anchored to the exact file and line, with a fenced snippet and a suggested
fix) and a summary body that ties it together. You are reviewing, not fixing —
do not edit code, commit, or push.

## When to use

- The user names a PR and asks for a review ("review PR 23 and post comments").
- The user asks you to review the current branch's open PR.
- After opening a PR, when the user wants an automated first-pass review before
  human reviewers look at it.

If the user wants you to *fix* reviewer comments instead of *produce* them, use
the `pr-review` skill. If they want a quick local review of uncommitted work,
use `/code-review`.

## Inputs you need

- **PR number.** If not given, resolve the current branch's PR:
  `gh pr view --json number,title,headRefName,baseRefName`. If there's no PR
  for the branch, ask.
- **Repo `owner/name`.** Get it once and reuse it:
  `gh repo view --json owner,name --jq '.owner.login + "/" + .name'`.
- **Review verdict.** Default to `event: COMMENT` (advisory, non-blocking).
  Only use `REQUEST_CHANGES` or `APPROVE` if the user explicitly asks —
  never approve or block someone's PR on your own initiative.

## Procedure

### 1. Gather context

Never review from the diff alone. Pull down the branch and the metadata first:

```bash
gh pr view <N> --json number,title,body,headRefName,baseRefName,headRefOid,files,additions,deletions
gh pr diff <N> > /tmp/pr.diff          # the unified diff (use the scratchpad dir on Windows)
gh pr checkout <N>                      # or: git fetch origin pull/<N>/head
```

Record the **head commit SHA** (`headRefOid`) — every inline comment must be
anchored to it. Then:

- **Read every changed file in full**, not just the hunks. A finding often
  lives in how a changed function interacts with lines the diff didn't touch.
- **Read the callers and callees** of changed functions across the repo
  (`Grep` for the symbol) to judge whether a change breaks a contract.
- **Read the project's conventions**: `CLAUDE.md`, `docs/CONTRIBUTING.md`,
  `docs/STYLE.md`, `docs/PLAN.md`. Findings that cite a project rule land
  harder and avoid false positives.
- Note the base branch — review against what's actually merging in, not `main`
  if the PR targets something else.

### 2. Run the automated checks

Let the tooling find the mechanical issues so your attention goes to the ones
it can't. Run the project's own commands (from `CLAUDE.md`) on the changed
files and capture the output:

- **Backend (Python)**, from `backend/`:
  - `tox run -e lint` (ruff check) — or `uv run ruff check app/`
  - `tox run -e type` (mypy) — or `uv run mypy app/`
  - `uv run pytest` — note any failures the PR introduces
  - `uv run ruff format --check .` — formatting drift
- **Frontend (TS/React)**, from `frontend/`:
  - `npm run lint` (eslint)
  - `npm run build` (`tsc -b` type-check + bundle)

Attribute each reported problem to a specific changed line before turning it
into a comment. Tooling output that predates the PR (a failure also present on
the base branch) is **not** this PR's finding — don't report pre-existing
baseline noise as if the PR caused it. When in doubt, check out the base and
re-run.

### 3. Review across every dimension

Go through the diff category by category. For each, ask "what would break, and
under what input?" — a finding needs a concrete failure scenario, not a vague
"this could be better."

- **Correctness / logic bugs** — off-by-one, wrong operator, inverted
  condition, unhandled `None`/`null`/empty, mismatched units, incorrect
  async/await, race conditions, resource leaks (unclosed sessions/files),
  wrong error handling, mutation of shared state.
- **Contract / integration** — does the change break a caller elsewhere in the
  repo? Wrong function signature, changed return shape, an ORM model added
  without importing it in `app/db/models/__init__.py` (Alembic won't see it),
  a Pydantic schema that no longer matches the response.
- **Security & vulnerabilities** — injection (SQL/command/path traversal),
  missing authz/authn on a new route, unvalidated input, SSRF, unsafe
  deserialization, `dangerouslySetInnerHTML` / XSS, CORS/misconfig, secrets in
  logs, dependency with a known CVE.
- **Exposed secrets / env** — hardcoded API keys, tokens, passwords,
  connection strings, private keys, or a real value committed where
  `.env.template` / config should be. **Redact the value in your comment**
  (show `sk-…REDACTED`), flag the location and type, and recommend rotation +
  moving to env/secret store. Never paste a live secret into a public comment.
- **Type & lint** — anything the checkers flagged in step 2, plus `Any`
  escape hatches, unchecked casts, ignored `# type: ignore` / `eslint-disable`
  without justification.
- **Complexity** — functions doing too much, deep nesting, duplicated logic
  that should be shared, an abstraction that's heavier than the problem.
  Suggest the smaller shape.
- **Dead / redundant code** — unreachable branches, unused
  imports/vars/exports introduced by the PR, and **commented-out blocks of
  code** left in the diff (distinct from explanatory comments — flag the former,
  keep the latter).
- **Documentation & comments** — missing docstrings on new public
  functions/endpoints, and **stale or misleading inline comments**: a comment
  that describes what the code used to do, contradicts the code next to it, or
  claims an invariant the code doesn't enforce.
- **Tests** — new logic with no test, a changed behavior whose test wasn't
  updated, a test that asserts nothing meaningful.
- **Style / conventions** — violations of `docs/STYLE.md`, `CLAUDE.md`, or
  local idiom (e.g. hand-adding `useMemo` when the React Compiler is on,
  editing a version file by hand instead of `scripts/sync_versions.py`,
  hitting a URL directly instead of `VITE_BACKEND_BASE_API_URL`).

Prioritize by severity: **blocker** (bug/security/secret) > **major** (likely
bug, missing test on risky code) > **minor** (quality, complexity) >
**nit** (style, naming). Don't drown the real findings in nitpicks — if you
have 20 nits, roll the trivial ones into one summary paragraph.

### 4. Write each finding as a comment

Every inline finding is anchored to a `path` and `line` **that appears in the
PR diff** (GitHub rejects inline comments on lines outside the diff — see Edge
cases for out-of-diff findings). Use the line number on the **RIGHT** side (the
new file) for added/changed lines. Give each comment this shape:

> **[severity · category]** One-sentence statement of the problem.
>
> Why it matters / the failure scenario (concrete input → wrong result).
>
> ```<lang>
> // the exact snippet the comment is about
> ```
>
> **Suggested fix:** what to change and why.

Include the code snippet in a fenced block with the right language tag so it
renders. When the fix is a small, unambiguous edit, offer a GitHub
**suggestion block** so the author can apply it in one click:

````markdown
```suggestion
if user is not None and user.is_active:
```
````

(The `suggestion` block must contain the exact replacement for the commented
line range — GitHub applies it verbatim.)

Keep each comment self-contained: someone reading only that comment should be
able to find the code, understand the problem, and see the fix.

### 5. Post the review to GitHub

Batch all inline comments into **one** review call so the author gets a single
notification, not one per finding. Build a JSON payload in the scratchpad dir
(never write it into the repo tree) and submit it:

```json
{
  "commit_id": "<headRefOid>",
  "event": "COMMENT",
  "body": "## Automated review of PR #<N>\n\n<overview: scope, N blockers / M majors / K minors, checks run and their result, and any out-of-diff findings that couldn't be inline-anchored>",
  "comments": [
    { "path": "backend/app/api/v1/example.py", "line": 42, "side": "RIGHT",
      "body": "**[blocker · correctness]** ...\n\n```python\n...\n```\n\n**Suggested fix:** ..." },
    { "path": "frontend/src/App.tsx", "start_line": 10, "line": 14,
      "start_side": "RIGHT", "side": "RIGHT",
      "body": "**[minor · complexity]** ...\n\n```tsx\n...\n```" }
  ]
}
```

```bash
gh api repos/<owner>/<repo>/pulls/<N>/reviews -X POST --input /path/to/review.json
```

- Single-line comment → `line` + `side: "RIGHT"`. Multi-line → `start_line` +
  `line` (+ `start_side`/`side`).
- If the whole call fails with `422` because one comment anchors to a line not
  in the diff, remove that comment from `comments[]`, move its content into the
  summary `body` (with a `path:line` reference), and resubmit. Don't let one
  bad anchor sink the whole review.
- Before posting, if the PR was already reviewed by you, list existing review
  comments (`gh api repos/<owner>/<repo>/pulls/<N>/comments --paginate`) and
  skip findings you've already posted — don't duplicate on a re-run.

### 6. Report back

Summarize to the user: how many comments you posted, broken down by severity;
the result of each automated check; and anything you couldn't anchor inline.
Note that the review is posted as `COMMENT` (advisory) and remind them you did
not modify, commit, or push any code.

## Conventions

- **Review only — no edits, no commits, no pushes.** Producing comments is the
  whole job. If the user wants fixes applied, that's the `pr-review` skill or a
  follow-up.
- **Default to `COMMENT`.** Never `APPROVE` or `REQUEST_CHANGES` unless the
  user explicitly asks — an automated approval is meaningless and a blocking
  review is disruptive.
- **One review, batched.** All inline comments go in a single reviews-API call.
- **Every finding is falsifiable.** State the input/condition that triggers the
  bug. If you can't describe how it fails, it's an opinion — put it in the
  summary as a suggestion, not an inline "bug" comment.
- **Cite the rule.** When a finding violates `CLAUDE.md` / `docs/STYLE.md` /
  `CONTRIBUTING.md`, name the doc — it turns a matter of taste into a matter
  of convention.
- **Redact secrets.** Flag location and type; never echo a live credential into
  a comment. Recommend rotation, because a committed secret is already leaked.
- **Snippet in every inline comment**, fenced with a language tag.

## Edge cases

- **Finding outside the diff.** The bug is real but lives in an unchanged line
  (found via full-codebase context). GitHub can't inline-anchor it. Put it in
  the summary `body` with an explicit `path:line` and snippet, labeled
  "(outside this PR's diff)".
- **Huge PR.** If the diff is very large, review in passes by area (backend
  routes, then services, then frontend) and tell the user you're paginating.
  Don't silently review only the first few files — say what you covered.
- **Generated / vendored / lockfile changes.** Skip `package-lock.json`,
  migrations' autogenerated bodies, build output, and vendored deps for
  line-level review; note at most that they changed. Do sanity-check a
  migration's *intent* against the ORM change that produced it.
- **PR from a fork.** `gh pr checkout` still works; the head SHA and
  `owner/repo` are the *base* repo's for the reviews API — post to the base
  repo, anchored to the fork's head SHA (which `headRefOid` already gives you).
- **No PR for the branch.** Don't invent one. Ask the user for the PR number or
  whether to open a PR first (they open it; you don't).
- **Checks won't run** (deps not installed, no DB for tests). Report which
  checks you *couldn't* run rather than skipping silently — an unrun type-check
  is not a passing type-check.
