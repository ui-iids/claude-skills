---
name: ado-pr-review
description: Address reviewer comments on an Azure DevOps (ADO) pull request by fixing each comment with its own focused commit, or justifying why no change is needed, then replying on every comment thread with the resolving commit SHA and setting the thread status (Fixed / Won't Fix / By Design); finishes by linking the ADO work items the PR completes (asking the user when unsure). Uses the `az` CLI with the azure-devops extension. Use when the user invokes `/ado-pr-review`, asks to "address ADO PR comments", "fix the review comments on Azure DevOps PR <N>", "resolve Azure DevOps review feedback", or similar for a PR hosted in Azure Repos.
---

# ADO PR review response

Work through every review comment on an Azure DevOps pull request in a
disciplined, traceable way: one commit per addressed comment, a SHA-linked
reply on each thread, the thread status set to match the outcome, and a
written justification when no code change is appropriate.

This is the Azure DevOps counterpart of the `pr-review` skill. Use
`pr-review` when the PR lives on GitHub.

## When to use

- The user names an ADO PR and asks you to handle reviewer comments
  (e.g. "address the comments on ADO PR 1423").
- The user pastes a list of review comments from Azure DevOps and asks you
  to resolve them.
- After pushing changes that you expect to attract review comments, when
  the user circles back to clear them.

## Inputs you need

- **PR ID** (ADO calls it `pullRequestId`). If the user says "the open PR",
  look it up from the current branch:
  ```bash
  az repos pr list --source-branch "$(git branch --show-current)" --status active \
    --query "[].{id:pullRequestId, title:title, target:targetRefName}" -o table
  ```
  If there is no PR, or more than one, ask.
- **Organization, project, and repository.** Parse them from the git remote
  (`https://dev.azure.com/<org>/<project>/_git/<repo>`, or the older
  `https://<org>.visualstudio.com/<project>/_git/<repo>`, or
  `git@ssh.dev.azure.com:v3/<org>/<project>/<repo>`) and reuse them.
- **Author filter.** By default, act on threads from every reviewer except
  the PR author and ADO's own system messages. If the user names a reviewer
  or a bot, filter to that author's `displayName` / `uniqueName`.

## Procedure

### 0. Check the ADO tooling

Before anything else:

```bash
az extension show --name azure-devops --query version -o tsv   # extension installed?
az account show --query user.name -o tsv                        # signed in?
az devops configure --list                                      # org/project defaults?
```

- If the extension is missing, tell the user to run
  `az extension add --name azure-devops`.
- If they aren't signed in, ask them to run `az login`.
- If no defaults are set, pass `--org https://dev.azure.com/<org>` and
  `--project <project>` explicitly on each command, or suggest
  `az devops configure --defaults organization=https://dev.azure.com/<org> project=<project>`.

Never ask the user to paste a Personal Access Token into the chat, and never
echo one into a command or a file.

### 1. Enumerate the comment threads

The `az repos pr` command group has no thread commands, so call the REST
API through `az devops invoke`:

```bash
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> \
  --http-method GET --api-version 7.1 -o json > <tmpdir>/threads.json
```

(If the API version is rejected, retry with `--api-version 7.1-preview.1`.)

Each thread has an `id`, a `status` (`active`, `pending`, `fixed`,
`wontFix`, `byDesign`, `closed`), an optional `threadContext` (the file and
line anchor), and a `comments[]` array. Keep threads that:

- have `status` of `active` or `pending`;
- are not system threads (skip any thread whose first comment has
  `commentType` of `system`, and skip vote, policy, and "updated the pull
  request" status threads, which ADO marks with `properties.CodeReviewThreadType`);
- were not opened by the PR author (unless the user asks for those too);
- don't already contain a reply with "resolved in commit" (handled in an
  earlier pass).

For each thread, record:

- `id`: the thread ID, needed to reply and to set the status.
- `threadContext.filePath` + `threadContext.rightFileStart.line`: where the
  comment is anchored. Threads without a `threadContext` are general PR
  comments and are handled the same way, just without a file anchor.
- `comments[0].content`: the feedback itself. Read the later comments in the
  thread too, because the discussion may have moved on.

### 2. Plan

If there are 3+ threads, list them as an explicit plan (one item per
thread) so progress is visible. Mark an item in progress when you start a
thread and done (noting the resolving SHA) when its commit is pushed.

### 3. For each thread, decide: fix or justify

Read the file at the anchored path and line before deciding. ADO file paths
start with `/` and are relative to the repo root. There are two outcomes:

**(a) Fix it.**

- Make the smallest change that addresses the comment. Don't fold in unrelated
  cleanup. Reviewers expect one commit per concern so the trail is auditable.
- Run the relevant local checks before committing (`tsc -b` for TS changes,
  `pytest` for Python changes, etc.). If a check fails, fix the root cause.
  Never bypass hooks with `--no-verify`.
- Commit with a **detailed message**, not a one-liner. The message should have:
  - First line: imperative, under ~70 chars, describing the change.
  - Body paragraph(s): explain _what was wrong_, _why the fix is correct_,
    and any subtlety a future reader would need. Reference the PR
    (`Addresses review comment on ADO PR !<N>.`).
- Push after each commit (or batch and push once at the end, since the SHAs
  are stable either way). Capture the SHA from `git rev-parse HEAD`.

**(b) Justify not changing it.**

- Use this when the reviewer is wrong (e.g. misreads the code), the
  suggestion conflicts with project conventions in `AGENTS.md` or memory,
  or the cost of the change outweighs the benefit.
- Write the justification as the reply itself, not "resolved in commit …".
  Make it concrete: cite the line of code, the constraint, or the doc that
  supports your position. A bare "won't fix" is not acceptable.
- If you're uncertain, ask the user before posting a "won't fix" reply.
  Pushback on a shared PR is harder to walk back than a private question.

### 4. Reply on every thread

Post a reply comment into the thread. Every reply must explain the
outcome, not just point at a SHA. The reviewer (and anyone auditing the PR
later) should understand the resolution without opening the commit.

ADO does not reliably auto-link a bare SHA in PR comments, so write the SHA
as a markdown link to the commit page in the repo:

```
https://dev.azure.com/<org>/<project>/_git/<repo>/commit/<40-char-SHA>
```

**For fixes**, the body is `resolved in commit [<short-SHA>](<commit-url>)`
followed by a one- to two-sentence description of _what the fix actually
does_. Keep it concrete (what changed, where, and why it addresses the
comment) rather than restating the comment.

Write the payload to a temporary directory outside the repo (never into the repo tree):

```json
{ "content": "resolved in commit [3eae0a2f](https://dev.azure.com/<org>/<project>/_git/<repo>/commit/3eae0a2f…) — narrowed the cast so DOMPurify is called directly on the `string | null` input.", "parentCommentId": 1, "commentType": 1 }
```

```bash
az devops invoke --area git --resource pullRequestThreadComments \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> threadId=<thread_id> \
  --http-method POST --in-file <tmpdir>/reply.json --api-version 7.1
```

(`parentCommentId: 1` makes the reply a child of the thread's first comment.
`commentType: 1` is a normal text comment.)

**For justifications** (no code change), the body _is_ the justification.

Every reviewer thread must get exactly one reply. Don't batch multiple
resolutions into one reply, and don't skip a thread because it "seems
minor". Silently ignored review comments are the most common source of
friction with reviewers.

### 5. Set each thread's status

After replying, set the thread status so the PR overview shows it as
handled. ADO uses a status instead of GitHub's "resolved" flag. Pick the
status that matches the outcome:

| Outcome | Status |
|---|---|
| Code fix committed | `fixed` |
| Justified: the reviewer's concern doesn't apply / the code is correct as designed | `byDesign` |
| Justified: valid point, but deliberately not changing it (cost, scope, deferred) | `wontFix` |

```json
{ "status": "fixed" }
```

```bash
az devops invoke --area git --resource pullRequestThreads \
  --route-parameters project=<project> repositoryId=<repo> pullRequestId=<N> threadId=<thread_id> \
  --http-method PATCH --in-file <tmpdir>/status.json --api-version 7.1
```

Set the status on every thread you addressed. If you couldn't address a
thread (blocked, needs user input), leave it `active` and call it out in the
summary. Branch policies often require all threads to be non-active before
the PR can complete, so an unresolved thread is a real blocker.

### 6. Link the work items the PR completes

Only after every thread has been answered and its status set, check which
Azure Boards work items the PR delivers. ADO links PRs to **work items**
(User Stories, Bugs, Tasks). There are no GitHub-style issues or milestones.

See what is already linked:

```bash
az repos pr work-item list --id <N> --query "[].{id:id, title:fields.\"System.Title\", state:fields.\"System.State\"}" -o table
```

Find candidates: work items mentioned in the PR title, description, or branch
name, or in commit messages (`#1234`, `AB#1234`), plus open items assigned to
the user:

```bash
az repos pr show --id <N> --query "{title:title, description:description, source:sourceRefName}"
az boards query --wiql "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.TeamProject] = @project AND [System.AssignedTo] = @me AND [System.State] NOT IN ('Closed', 'Removed')" -o table
```

For each work item the PR **specifically completes** (its changes fully
satisfy that item's acceptance criteria, rather than just touching the same
area), link it:

```bash
az repos pr work-item add --id <N> --work-items <id> [<id> ...]
```

Linking is additive and doesn't change the PR description. Don't edit the
author's description to add work-item references.

**Completing work items on merge.** ADO can move linked work items to their
done state when the PR completes (`az repos pr update --id <N>
--transition-work-items true`). At U of I, User Stories move through
Testing and Pending Release before Closed, so auto-transitioning can skip
the PO's verification. **Don't turn this on unless the user explicitly asks.**

**Iteration paths.** Iteration Path is GitHub's nearest equivalent of a
milestone, and it belongs to the work item, not the PR. Report the linked items'
iteration paths in the summary, but never change a work item's iteration,
state, or fields from this skill.

**When unsure, ask.** If you can't tell whether the PR fully completes a work
item (partial fix, overlapping scope, vague acceptance criteria), present the
candidates to the user with a one-line reason each and link only what they
confirm.

### 7. Report back

When done, summarize which threads were fixed (with SHAs), which were
justified (and the status each was given), whether anything needed user input
or is still `active`, and which work items were linked (or that none applied).
Don't restate the full commit bodies, since the user can read them in `git log`.

## Conventions

- One concern → one commit. Don't bundle two unrelated review fixes into a
  single commit, even if they touch the same file.
- Detailed commit messages. The body explains the _why_ of the fix in
  depth and references the PR. The thread reply summarizes the fix in one or
  two sentences alongside the linked SHA. The commit message is where the full
  reasoning lives.
- Push before replying. The commit link in the reply must already resolve on
  the remote.
- Never force-push or amend after replying. Doing so makes the reply point at
  a rewritten SHA and breaks the audit trail (and ADO adds a confusing new PR
  iteration). If a follow-up is needed, add a new commit and post a second reply.
- Don't use `--no-verify` to silence pre-commit hooks. Fix the underlying
  issue.
- Don't open a new PR or branch. Work on the PR's existing source branch.
- Every addressed thread gets its status set (`fixed` / `byDesign` /
  `wontFix`). A reply alone leaves the thread `active` and can block PR
  completion under branch policy.
- Work-item linking comes last, and you link only what the PR *specifically*
  completes. "Related to" is not "completed by". When in doubt, ask.
- Never cast a vote (`az repos pr set-vote`) or complete the PR from this
  skill. That belongs to the reviewers and the author.

## Edge cases

- **Suggested-change comments.** Some ADO comments contain a suggested change.
  Treat them like any other comment: read the surrounding code first, then
  decide whether to accept verbatim, adapt, or justify rejecting. Don't click
  "Apply change" in the web UI, because that bypasses local type-checks and tests
  and creates a commit you didn't author locally.
- **Comment on code you already rewrote.** ADO anchors threads to a PR
  *iteration*. If the anchored line has moved since the comment was filed,
  locate the equivalent code and address it there. Say so in the reply
  ("resolved in commit … — relocated to `<new path>:<new line>` and applied
  the fix there").
- **Stale or duplicated comments.** If the same issue is raised twice, fix it
  once and reply on _both_ threads with the same SHA.
- **Comment on a file outside this repo.** If the path doesn't exist, ask the
  user. Don't guess.
- **Thread already `fixed` by the reviewer but no reply.** Leave it alone
  unless the user asks. The reviewer has closed it themselves.
- **`az devops invoke` returns 401/403.** The signed-in account lacks access
  to the repo. Stop and tell the user. Don't fall back to asking for a PAT.
