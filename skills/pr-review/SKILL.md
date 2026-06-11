---
name: pr-review
description: Address Copilot (or other reviewer) comments on a GitHub pull request by fixing each comment with its own focused commit, or justifying why no change is needed, then replying to every comment with the resolving commit SHA. Use when the user asks to "address PR comments", "fix Copilot comments on PR <N>", "resolve PR review feedback", or similar.
---

# PR review response

Process every review comment on a GitHub pull request — typically from the
Copilot reviewer bot — in a disciplined, traceable way: one commit per
addressed comment, a SHA-linked reply on each thread, and a written
justification when no code change is appropriate.

## When to use

- The user names a PR and asks you to handle reviewer comments
  (e.g. "address Copilot comments on PR 15").
- The user pastes a list of review comments and asks you to resolve them.
- After pushing changes that you expect to attract review comments, when
  the user circles back to clear them.

## Inputs you need

- PR number (and repo, if the working dir isn't the right repo).
- Author filter, if the user wants comments from only one reviewer (default:
  comments from `Copilot`, the GitHub bot login).

If the PR number isn't given, ask. If the user says "the open PR", check
`gh pr view --json number,title` for the current branch.

## Procedure

### 1. Enumerate the comments

Use the GitHub CLI / API to list review comments on the PR. Filter to the
relevant reviewer (default Copilot) and pull out the fields you need:

```bash
gh api repos/<owner>/<repo>/pulls/<N>/comments --paginate \
  --jq '.[] | select(.user.login=="Copilot") | {id, path, line: (.line // .original_line), body, in_reply_to_id}'
```

Skip threads that already have a reply containing "resolved in commit"
(or the older "resolve in commit") — they have been handled in a previous
pass. Skip replies (`in_reply_to_id` not null); only act on the top-level
review comments.

For each comment, record:

- `id` — needed to post the reply.
- `path` + `line` — where the comment is anchored.
- `body` — the actual feedback.

### 2. Plan with TaskCreate

If there are 3+ comments, create one `TaskCreate` task per comment so progress
is visible. Mark `in_progress` when you start a comment, `completed` (with
the resolving SHA in metadata) when its commit is pushed.

### 3. For each comment, decide: fix or justify

Read the file at the indicated path/line before deciding. Two outcomes:

**(a) Fix it.**

- Make the smallest change that addresses the comment. Don't fold in unrelated
  cleanup; reviewers expect one commit per concern so the trail is auditable.
- Run the relevant local checks before committing (`tsc -b` for TS changes,
  `pytest` for Python changes, etc.). If a check fails, fix the root cause —
  never bypass hooks with `--no-verify`.
- Commit with a **detailed message**, not a one-liner. The message should:
  - First line: imperative, under ~70 chars, describing the change.
  - Body paragraph(s): explain _what was wrong_, _why the fix is correct_,
    and any subtlety a future reader would need. Reference the PR
    (`Addresses Copilot review comment on PR #<N>.`).
- Push after each commit (or batch and push once at the end — the SHAs are
  stable either way). Capture the SHA from `git rev-parse HEAD`.

**(b) Justify not changing it.**

- Use this when the reviewer is wrong (e.g. misreads the code), the
  suggestion conflicts with project conventions in `CLAUDE.md` or memory,
  or the cost of the change outweighs the benefit.
- Write the justification as the reply itself, not "resolve in commit …".
  Make it concrete: cite the line of code, the constraint, or the doc that
  supports your position. A bare "won't fix" is not acceptable.
- If you're uncertain, ask the user before posting a "won't fix" reply —
  reviewer pushback on a public PR is harder to walk back than a private
  question.

### 4. Reply to every comment

Post a reply to the comment's thread. Every reply must explain the
outcome, not just point at a SHA — the reviewer (and future readers
auditing the PR) should be able to understand the resolution without
opening the commit.

**For fixes**, the body is `resolved in commit <40-char-SHA>` followed by
a one- to two-sentence description of _what the fix actually does_. Keep
it concrete (what changed, where, and why it addresses the comment) — not
a restatement of the comment itself.

```bash
gh api repos/<owner>/<repo>/pulls/<N>/comments/<comment_id>/replies \
  -X POST -f body="resolved in commit <40-char-SHA> — <short fix description>"
```

Example body:

> resolved in commit 3eae0a2f… — narrowed the `SafeHtml` cast so DOMPurify
> is called directly on the `string | null` input, removing the redundant
> `as string` and letting the type checker enforce the contract.

**For justifications** (no code change), the body _is_ the justification.
Make it concrete: cite the line of code, the constraint, or the doc that
supports your position. Explain why the suggested change is wrong,
unnecessary, or conflicts with a project convention. A bare "won't fix"
is not acceptable.

In both cases, use the **full 40-character SHA**, not the short form —
that's what the user requested and it's also what GitHub auto-links to
the commit on hover.

Every top-level Copilot comment must get exactly one reply. Don't batch
multiple resolutions into one reply, and don't skip a comment because it
"seems minor" — silently ignored review comments are the most common
friction point with bot reviewers.

### 5. Resolve each review thread

After posting the reply, mark the thread as resolved. The REST API can't
do this — use the GraphQL `resolveReviewThread` mutation. First fetch the
thread IDs (one per top-level comment), then resolve them one by one:

```bash
# Fetch thread IDs alongside the first comment's databaseId so you can
# match each thread back to the REST comment you replied to.
gh api graphql -f query='
{
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <N>) {
      reviewThreads(first: 50) {
        nodes {
          id
          isResolved
          comments(first: 1) { nodes { databaseId } }
        }
      }
    }
  }
}'

# Resolve a single thread.
gh api graphql \
  -f query='mutation($id: ID!) { resolveReviewThread(input: {threadId: $id}) { thread { id isResolved } } }' \
  -f id=<thread_id>
```

Resolve every thread you addressed, whether the resolution was a code
commit or a written justification — in either case the conversation has
been responded to, and leaving threads "unresolved" creates noise on the
PR overview. If you couldn't address a comment (blocked, needs user
input), leave that thread unresolved and call it out in the summary.

### 6. Report back

When done, summarize: which comments were fixed (with SHAs), which were
justified, and whether anything needed user input. Don't restate the full
commit bodies — the user can read them in `git log`.

## Conventions

- One concern → one commit. Don't bundle two unrelated Copilot fixes into a
  single commit even if they touch the same file.
- Detailed commit messages. The body explains the _why_ of the fix in
  depth and references the PR. The Copilot reply summarizes the fix in
  one or two sentences alongside the SHA; the commit message is where
  the full reasoning lives.
- Push before replying. The SHA in the reply must already be visible on the
  remote, otherwise GitHub won't render it as a commit link.
- Never force-push or amend after replying — the reply now points at a
  rewritten SHA and the audit trail breaks. If a follow-up is needed, add a
  new commit and post a second reply.
- Don't use `--no-verify` to silence pre-commit hooks; fix the underlying
  issue.
- Don't open a new PR or branch — work on the branch the PR is already
  tracking.
- Every addressed thread must be resolved at the end via the GraphQL
  `resolveReviewThread` mutation. Posting the reply is not enough; the PR
  overview still shows unresolved threads until the mutation runs.

## Edge cases

- **Reviewer-suggested code blocks.** Some Copilot comments include a
  GitHub "suggestion" diff. Treat them like any other comment: read the
  surrounding code first, then decide whether to accept verbatim, adapt, or
  justify rejecting. Don't apply suggestions blind via the GitHub UI; you'll
  bypass local type-checks and tests.
- **Comment on code you already rewrote.** If the reviewer's anchor line has
  changed since the comment was filed, locate the equivalent code and address
  it there. Reply explaining the move ("resolved in commit … — relocated
  to <new path>:<new line> and applied the fix there").
- **Stale or duplicated comments.** If the same issue is raised twice, fix it
  once and reply to _both_ threads with the same SHA.
- **Comment on a file outside this repo.** If the path doesn't exist, ask the
  user — don't guess.
