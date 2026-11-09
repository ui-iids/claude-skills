---
name: ado-review-pr
description: Perform a thorough code review of an Azure DevOps (ADO) pull request, reading the diff in full-codebase context and covering bugs, logic errors, code quality, lint/typing, complexity, security, exposed secrets, dead/commented-out code, and stale or misleading comments, then post each finding as a line-anchored ADO comment thread (with a fenced code snippet and a fix) plus a summary thread. Uses the `az` CLI with the azure-devops extension and never votes unless asked. Use when the user invokes `/ado-review-pr`, asks to "review ADO PR <N>", "review this Azure DevOps PR and post comments", "do a full PR review in Azure Repos", or similar for a PR hosted in Azure DevOps.
---

# ADO review PR

Do a deep, adversarial code review of an Azure DevOps pull request and post
the findings back to ADO as comment threads. The diff is the subject, but you
read it **in the context of the whole codebase**. A line can be correct in
isolation and wrong given how the rest of the repo uses it.

The output is one summary thread plus one inline thread per concrete finding
(anchored to the exact file and line, with a fenced snippet and a suggested
fix). You are reviewing, not fixing. Do not edit code, commit, or push.

This is the Azure DevOps counterpart of the `review-pr` skill. Use
`review-pr` when the PR lives on GitHub.

## When to use

- The user names an ADO PR and asks for a review ("review PR 1423 and post
  comments").
- The user asks you to review the current branch's open ADO PR.
- After opening a PR, when the user wants an automated first-pass review before
  human reviewers look at it.

If the user wants you to *fix* reviewer comments instead of *produce* them, use
the `ado-pr-review` skill. If they want a quick local review of uncommitted work,
use `/code-review`.

## Inputs you need

- **PR ID.** If not given, resolve the current branch's PR:
  ```bash
  az repos pr list --source-branch "$(git branch --show-current)" --status active \
    --query "[].{id:pullRequestId, title:title, target:targetRefName}" -o table
  ```
  If there's no PR for the branch, ask.
- **Organization, project, and repository.** Parse them once from the git remote
  (`https://dev.azure.com/<org>/<project>/_git/<repo>`, the older
  `https://<org>.visualstudio.com/<project>/_git/<repo>`, or
  `git@ssh.dev.azure.com:v3/<org>/<project>/<repo>`) and reuse them.
- **Vote.** Default to **no vote**, so the review is advisory comments only.
  Only call `az repos pr set-vote` if the user explicitly asks. Never approve or
  block someone's PR on your own initiative.

## Procedure

### 0. Check the ADO tooling

```bash
az extension show --name azure-devops --query version -o tsv   # extension installed?
az account show --query user.name -o tsv                        # signed in?
az devops configure --list                                      # org/project defaults?
```

If the extension is missing, tell the user to run
`az extension add --name azure-devops`. If they aren't signed in, tell them to
run `az login`. If no defaults are set, pass `--org` / `--project` explicitly.
Never ask for a Personal Access Token in chat or echo one into a command.

### 1. Gather context

Never review from the diff alone. Pull down the metadata and branches first:

```bash
az repos pr show --id <N> \
  --query "{title:title, description:description, source:sourceRefName, target:targetRefName, head:lastMergeSourceCommit.commitId, base:lastMergeTargetCommit.commitId, isDraft:isDraft}"
```

ADO has no `pr diff` / `pr checkout` command, so use git directly (strip
`refs/heads/` from the ref names):

```bash
git fetch origin <target-branch> <source-branch>
git diff origin/<target-branch>...origin/<source-branch> > <tmpdir>/pr.diff
git diff --stat origin/<target-branch>...origin/<source-branch>
git checkout <source-branch>     # or a detached checkout of the head commit
```

Record the **head commit** (`lastMergeSourceCommit.commitId`). Your local
checkout must match it, so that the line numbers you anchor to are the ones
ADO shows. Then:

- **Read every changed file in full**, not just the hunks. A finding often
  lives in how a changed function interacts with lines the diff didn't touch.
- **Read the callers and callees** of changed functions across the repo
  (`Grep` for the symbol) to judge whether a change breaks a contract.
- **Read the project's conventions**: `AGENTS.md`, `docs/CONTRIBUTING.md`,
  `docs/STYLE.md`, `docs/PLAN.md`. Findings that cite a project rule land
  harder and avoid false positives.
- **Read the linked work items** (`az repos pr work-item list --id <N>`), and
  check the change against each User Story's Acceptance Criteria. A scenario
  the PR doesn't satisfy is a finding.
- Note the target branch. Review against what's actually merging in, not
  `main` if the PR targets something else.

### 2. Run the automated checks

Let the tooling find the mechanical issues so your attention goes to the ones
it can't. Run the project's own commands (from `AGENTS.md`) on the changed
files and capture the output:

- **Backend (Python)**, from `backend/`:
  - `tox run -e lint` (ruff check), or `uv run ruff check app/`
  - `tox run -e type` (mypy), or `uv run mypy app/`
  - `uv run pytest`, noting any failures the PR introduces
  - `uv run ruff format --check .` for formatting drift
- **Frontend (TS/React)**, from `frontend/`:
  - `npm run lint` (eslint)
  - `npm run build` (`tsc -b` type-check + bundle)

Also note the PR's own build-validation results, if the branch policy runs a
pipeline:
`az repos pr policy list --id <N> --query "[].{policy:configuration.type.displayName, status:status}" -o table`.

Attribute each reported problem to a specific changed line before turning it
into a comment. Tooling output that predates the PR (a failure also present on
the target branch) is **not** this PR's finding. Don't report pre-existing
baseline noise as if the PR caused it. When in doubt, check out the target and
re-run.

### 3. Review across every dimension

Go through the diff category by category. For each, ask "what would break, and
under what input?" A finding needs a concrete failure scenario, not a vague
"this could be better."

- **Correctness / logic bugs**: off-by-one, wrong operator, inverted
  condition, unhandled `None`/`null`/empty, mismatched units, incorrect
  async/await, race conditions, resource leaks (unclosed sessions/files),
  wrong error handling, mutation of shared state.
- **Contract / integration**: does the change break a caller elsewhere in the
  repo? Look for a wrong function signature, a changed return shape, an ORM model
  added without importing it in `app/db/models/__init__.py` (so Alembic won't see
  it), or a Pydantic schema that no longer matches the response.
- **Acceptance criteria**: for each linked User Story, does every Gherkin
  `SCENARIO` have code (and ideally a test) that makes its `THEN` true?
- **Security & vulnerabilities**: injection (SQL/command/path traversal),
  missing authz/authn on a new route, unvalidated input, SSRF, unsafe
  deserialization, `dangerouslySetInnerHTML` / XSS, CORS/misconfig, secrets in
  logs, a dependency with a known CVE.
- **Exposed secrets / env**: hardcoded API keys, tokens, passwords,
  connection strings, private keys, or a real value committed where
  `.env.template` / config should be. **Redact the value in your comment**
  (show `sk-…REDACTED`), flag the location and type, and recommend rotation +
  moving it to env, a secret store, or Azure Key Vault / pipeline secret
  variables. Never paste a live secret into a comment.
- **Type & lint**: anything the checkers flagged in step 2, plus `Any`
  escape hatches, unchecked casts, and `# type: ignore` / `eslint-disable`
  without justification.
- **Complexity**: functions doing too much, deep nesting, duplicated logic
  that should be shared, an abstraction that's heavier than the problem.
  Suggest the smaller shape.
- **Dead / redundant code**: unreachable branches, unused
  imports/vars/exports introduced by the PR, and **commented-out blocks of
  code** left in the diff. Flag those, but keep explanatory comments.
- **Documentation & comments**: missing docstrings on new public
  functions/endpoints, and **stale or misleading inline comments**, meaning a
  comment that describes what the code used to do, contradicts the code next to
  it, or claims an invariant the code doesn't enforce.
- **Tests**: new logic with no test, a changed behavior whose test wasn't
  updated, a test that asserts nothing meaningful.
- **Style / conventions**: violations of `docs/STYLE.md`, `AGENTS.md`, or
  local idiom (e.g. hand-adding `useMemo` when the React Compiler is on,
  editing a version file by hand instead of `scripts/sync_versions.py`,
  hitting a URL directly instead of `VITE_BACKEND_BASE_API_URL`).

Prioritize by severity: **blocker** (bug/security/secret) > **major** (likely
bug, missing test on risky code, unmet acceptance criterion) > **minor**
(quality, complexity) > **nit** (style, naming). Don't drown the real findings
in nitpicks. If you have 20 nits, roll the trivial ones into one paragraph of
the summary thread.

### 4. Write each finding as a comment

Every inline finding is anchored to a file and line **in a file the PR
changes**, using the line number in the **new** (source-branch) version of the
file. Give each comment this shape:

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
renders (ADO comments are markdown). When the fix is a small, unambiguous
edit, you may add a **suggestion block** so the author can apply it from the
web UI:

````markdown
```suggestion
if user is not None and user.is_active:
```
````

The `suggestion` block must contain the exact replacement for the commented
line range. Azure DevOps Services renders these as an "Apply change" button.
Some older on-prem Azure DevOps Server versions show them as a plain code
block, which is still readable, so the comment works either way.

Keep each comment self-contained. Someone reading only that comment should be
able to find the code, understand the problem, and see the fix.

### 5. Post the review to ADO

ADO has **no batched review**. Each comment is its own thread, created with a
separate POST. Build every payload as a JSON file in a temporary directory outside the repo
(never in the repo tree).

**Check for an earlier run first.** List the existing threads and skip any
finding you've already posted, so a re-run doesn't create duplicates:

```bash
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> \
  --http-method GET --api-version 7.1 -o json > <tmpdir>/existing-threads.json
```

**Get the latest iteration** so inline threads attach to the current diff
rather than floating on an old one:

```bash
az devops invoke --area git --resource pullRequestIterations \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> \
  --http-method GET --api-version 7.1 --query "value[-1].id" -o tsv
```

**1) Post the summary thread first** (no `threadContext`, so it shows as a
general PR comment at the top of the Overview):

```json
{
  "comments": [{ "parentCommentId": 0, "commentType": 1,
    "content": "## Automated review of PR !<N>\n\n<overview: scope, N blockers / M majors / K minors, checks run and their result, acceptance-criteria coverage, and any findings that couldn't be anchored inline>" }],
  "status": "active"
}
```

**2) Then post one inline thread per finding:**

```json
{
  "comments": [{ "parentCommentId": 0, "commentType": 1,
    "content": "**[blocker · correctness]** ...\n\n```python\n...\n```\n\n**Suggested fix:** ..." }],
  "status": "active",
  "threadContext": {
    "filePath": "/backend/app/api/v1/example.py",
    "rightFileStart": { "line": 42, "offset": 1 },
    "rightFileEnd":   { "line": 42, "offset": 200 }
  },
  "pullRequestThreadContext": {
    "iterationContext": { "firstComparingIteration": 1, "secondComparingIteration": <latest-iteration-id> }
  }
}
```

```bash
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> \
  --http-method POST --in-file <tmpdir>/thread-<k>.json --api-version 7.1
```

- `filePath` must start with `/` and be relative to the repo root.
- For a single line, use the same line for `rightFileStart` and `rightFileEnd`.
  For a range, use the first and last line. `offset` is a 1-based column, so set
  the end offset to the end of the line (any value past the line's length works).
- Anchor only to the **right** side (new code). Use `leftFileStart/End` only
  when the finding is about a *deleted* line.
- If the API version is rejected, retry with `--api-version 7.1-preview.1`.
- If one POST fails (bad path, line out of range), don't abandon the review.
  Move that finding into a reply on the summary thread with an explicit
  `path:line` and snippet, and keep going.

**3) Vote only if asked.** If the user explicitly wants a vote:
`az repos pr set-vote --id <N> --vote approve | approve-with-suggestions | wait-for-author | reject`.

### 6. Report back

Summarize to the user: how many threads you posted, broken down by severity;
the result of each automated check and of the PR's policy builds; which
acceptance-criteria scenarios are covered or uncovered; and anything you
couldn't anchor inline. Say that no vote was cast (unless one was requested),
and remind them you did not modify, commit, or push any code.

## Conventions

- **Review only: no edits, no commits, no pushes.** Producing comments is the
  whole job. If the user wants fixes applied, that's the `ado-pr-review` skill
  or a follow-up.
- **No vote by default.** Never approve, wait-for-author, or reject unless the
  user explicitly asks. An automated approval is meaningless and a blocking vote
  is disruptive. Never complete or abandon the PR.
- **Summary first, then inline threads.** That keeps the overview readable and
  makes clear what follows.
- **Every finding is falsifiable.** State the input/condition that triggers the
  bug. If you can't describe how it fails, it's an opinion, so put it in the
  summary as a suggestion rather than an inline "bug" thread.
- **Cite the rule.** When a finding violates `AGENTS.md` / `docs/STYLE.md` /
  `CONTRIBUTING.md` or a linked story's acceptance criteria, name the source. It
  turns a matter of taste into a matter of convention.
- **Redact secrets.** Flag location and type, and never echo a live credential
  into a comment. Recommend rotation, because a committed secret is already
  leaked.
- **Snippet in every inline comment**, fenced with a language tag.

## Edge cases

- **Finding in a file the PR doesn't touch.** The bug is real but lives in an
  unchanged file (found via full-codebase context). Put it in the summary thread
  with an explicit `path:line` and snippet, labeled "(outside this PR's diff)".
  A finding on an *unchanged line of a changed file* can still be anchored
  inline. ADO allows it, but say in the comment that the line isn't part of the
  diff.
- **Huge PR.** If the diff is very large, review in passes by area (backend
  routes, then services, then frontend) and tell the user you're paginating.
  Don't silently review only the first few files. Say what you covered.
- **Generated / vendored / lockfile changes.** Skip `package-lock.json`,
  migrations' autogenerated bodies, build output, and vendored deps for
  line-level review, and note at most that they changed. Do sanity-check a
  migration's *intent* against the ORM change that produced it.
- **PR from a fork.** Fetch the fork's source branch with
  `git fetch <fork-remote-url> <branch>` (the fork URL is in
  `az repos pr show --id <N> --query forkSource`), and post threads to the
  *target* repository.
- **Draft PR.** Review it if asked, but mention in the summary that it's a draft.
- **No PR for the branch.** Don't invent one. Ask the user for the PR ID or
  whether to open a PR first (they open it, not you).
- **Checks won't run** (deps not installed, no DB for tests). Report which
  checks you *couldn't* run rather than skipping silently. An unrun type-check
  is not a passing type-check.
- **`az devops invoke` returns 401/403.** The signed-in account lacks access
  to the repo. Stop and tell the user. Don't fall back to asking for a PAT.
