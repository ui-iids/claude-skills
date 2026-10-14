---
name: track-my-work
description: Turn a coding session into actionable ClickUp tasks — either planning new work (a parent task plus subtasks for each major component) or logging completed work (same decomposition, then marking tasks complete and logging time in 15-minute increments per subtask). Verifies the ClickUp connector first, asks whether this is planning or logging, reuses an existing task ID or creates a new task in a confirmed space, always includes PR-review and addressing-PR-comments subtasks, and previews every write before touching ClickUp. Use when the user invokes `$track-my-work`, asks to "log my time in ClickUp", "track this work", "turn this session into ClickUp tasks", "plan this in ClickUp", "log this PR to ClickUp", or similar.
---

# Track my work

You are converting the work in this session into a clean, honest record in
ClickUp. There are two modes:

- **Plan** — the work hasn't happened yet. Create a task and subtasks that
  describe what will be done.
- **Log** — the work is done (typically a coding session carried through to a
  merged PR). Create or reuse a task and subtasks that describe what *was*
  done, mark them complete, and log the time spent on each.

The deliverable is a ClickUp task a teammate or supervisor could read and
understand at a glance — not a transcript of every command you ran.

## Non-negotiables

- **Never write to ClickUp without an approved preview.** Before creating
  tasks, changing statuses, or logging time, show the user exactly what you
  will write (task names, subtasks, statuses, time entries) and wait for an
  explicit yes. Edits the user asks for get re-previewed.
- **Never delete or overwrite existing ClickUp data.** Don't delete tasks,
  subtasks, comments, or time entries. Don't rename or re-describe an existing
  task unless the user asks. If something existing looks wrong, flag it.
- **Never invent time.** Every logged duration must trace back to evidence
  (commit/PR timestamps, the session itself, or the user's own numbers) and
  be shown to the user with its basis before logging.
- **Time is logged in 15-minute increments.** Round each subtask to the
  nearest 15 minutes, with a 15-minute minimum for any subtask that had real
  work. No 7- or 22-minute entries.
- **Only mark work complete if it is complete.** In log mode, if the PR is
  not merged (or the work is otherwise unfinished), say so and ask before
  closing anything.
- **Don't create a new task when one already exists.** Always ask for an
  existing task ID first, and search before creating.

## Procedure

### 1. Verify the ClickUp connection

1. Look for ClickUp tools in this session (names containing `clickup_`, e.g.
   `clickup_get_workspace_hierarchy`, `clickup_create_task`,
   `clickup_add_time_entry`), exposed through a configured ClickUp MCP
   server.
2. Make one cheap read call to prove the connection actually works — e.g.
   `clickup_get_workspace_hierarchy` — and note the workspace name.
3. **If no ClickUp tools exist, or the call fails with an auth/connection
   error, stop.** Tell the user the connector isn't configured and how to add
   it:
   - **Codex CLI / IDE:** run `codex mcp list` to check configured servers.
     If ClickUp isn't listed, add ClickUp's MCP server (see ClickUp's MCP
     documentation for the current URL) with `codex mcp add`, or under
     `[mcp_servers]` in `~/.codex/config.toml`, complete the ClickUp
     authorization, then restart the Codex session.

   Don't continue until the connection is verified. Don't fall back to
   writing a markdown "task list" as a substitute.

### 2. Ask: planning or logging?

Ask the user whether this is:

- **Plan new work** — create tasks for work that is about to happen, or
- **Log completed work** — record and time work that's already done.

If the session makes the answer obvious (e.g. you just merged a PR together),
recommend that option, but still ask.

### 3. Identify the target task

Ask whether the user already has a ClickUp task for this work and, if so, for
its **task ID** (or URL — extract the ID from it).

**If they give an ID:**

1. Fetch it with `clickup_get_task` (include subtasks). Show the user its
   name, list/space, status, and existing subtasks, and confirm it's the
   right task.
2. Treat existing subtasks as the starting point. Map planned/completed work
   onto them, and only propose *new* subtasks for components that aren't
   already covered. Never duplicate a subtask that exists under a slightly
   different name.

**If there is no task:**

1. Ask for the **Space** name (and the List/Folder within it, if the space has
   more than one list). Resolve it against the workspace hierarchy; if the
   name is ambiguous or not found, show the closest matches and ask.
2. Search that list (`clickup_search` / `clickup_filter_tasks`) for an
   existing task that looks like the same work. If you find a likely match,
   offer it before creating anything.
3. **Explicitly confirm** it's okay to create a new task in that list before
   proceeding.

### 4. Gather evidence about the work

Build the picture from real sources, not memory:

- **This session's conversation** — what was asked, designed, built, debugged.
- **Git** — `git log` on the working branch (subjects, authors, timestamps),
  `git diff --stat` against the base branch for the shape of the change.
- **The PR**, when one exists — `gh pr view <n> --json
  title,url,state,mergedAt,createdAt,reviews,comments,commits` and
  `gh api repos/{owner}/{repo}/pulls/<n>/comments` for review threads. Note
  who reviewed, how many review rounds, and which commits addressed review
  feedback.

In plan mode, the evidence is the plan itself: the user's description, any
design discussion in the session, and the parts of the codebase it touches.

If there's no PR or git history to lean on (e.g. a non-coding session), ask
the user for a short summary of what was done and roughly when.

### 5. Decompose into a task and subtasks

Produce **one parent task** and **subtasks for each major component** of the
work.

- **Aim for roughly 3–8 subtasks.** Each subtask should be a meaningful chunk
  someone would recognize as a unit of work — "Add Postgres-backed session
  store", "Write backend tests for auth routes", "Update deployment docs".
- **Avoid granular subtasking.** Don't make subtasks for individual commits,
  files, functions, lint fixes, or one-line bug fixes. Fold small items into
  the component they belong to. If a subtask would take under ~15 minutes, it
  probably belongs inside another one.
- **Always include the review cycle** when the work involves a PR:
  - **PR review** — opening the PR, writing the description, requesting and
    receiving review.
  - **Address PR review comments** — changes made in response to review.

  In plan mode, include both by default. In log mode, include "Address PR
  review comments" only if review feedback actually led to changes; if the
  PR merged with no requested changes, say so and leave it out.
- **Parent task** (when creating one): a concise, outcome-oriented name; a
  description with a 2–4 sentence summary, plus links to the PR, branch, and
  repo when they exist. Assign it to the user (resolve "me" via
  `clickup_resolve_assignees` or `clickup_find_member_by_name`).
- **Subtask descriptions**: one or two lines on scope. In log mode, mention
  the relevant commits or PR sections briefly.
- **Plan mode only:** offer (don't force) time estimates per subtask, also in
  15-minute increments.

### 6. (Log mode only) Estimate time per subtask

Derive a duration for each subtask from evidence, in this order of
preference:

1. **Numbers the user gives you** — always win.
2. **Git/PR timestamps** — cluster commits by subtask; use the span between
   the first and last commit of a cluster, plus reasonable lead-in time
   before the first commit. Use PR `createdAt`, review timestamps, and
   `mergedAt` to bound the review subtasks.
3. **The session itself** — how long the conversation spent on each part.

Then:

- Round each subtask to the nearest 15 minutes (minimum 15).
- Don't double-count: time goes on **subtasks**, not on the parent task —
  ClickUp rolls subtask time up to the parent. Only log on the parent if it
  has no subtasks.
- Check existing time entries on the task/subtasks
  (`clickup_get_time_entries`) and **don't re-log time already recorded**.
  Show any existing entries alongside your proposal.
- Set each entry's start time to when that work actually happened (from the
  git/PR timestamps) when known; otherwise ask, defaulting to today.

### 7. Preview and confirm

Show the user a single preview before any write. For example:

```
Parent: Add OAuth login to the RCDS portal   [new · Space: RCDS / List: Portal]
  PR: #142 (merged 2026-10-13)

Subtask                                  Status     Time    Basis
─────────────────────────────────────────────────────────────────────────
Implement OAuth flow & callback route    complete   2h 00m  commits a1b2..c3d4, 10:05–11:50
Add session persistence                  complete   1h 15m  commits e5f6..g7h8
Backend tests for auth routes            complete   1h 00m  commits i9j0..k1l2
PR review                                complete   0h 30m  PR opened 13:40, approved 15:10
Address PR review comments               complete   0h 45m  2 review threads, commits m3n4..o5p6
─────────────────────────────────────────────────────────────────────────
Total                                               5h 30m
```

Ask the user to approve, or to adjust names, subtasks, or times. Re-preview
after changes. Plan mode previews the same structure without statuses or
time entries (and with estimates only if the user opted in).

### 8. Write to ClickUp

Only after approval:

1. **Create or reuse the parent task.** If new, create it in the confirmed
   list.
2. **Create the new subtasks** under the parent. When creating, updating, or
   timing several items at once, check `clickup_get_operators` for a bulk
   operation and run it with `clickup_execute_operator` rather than calling a
   single-object tool once per item.
3. **Log mode only:**
   - Log each subtask's time with `clickup_add_time_entry` (duration and start
     time from the approved preview, assigned to the user).
   - Mark each completed subtask closed. Look up the list's statuses first
     (`clickup_get_list`) and use its actual closed/done status name — don't
     guess "complete".
   - Mark the parent task closed **only** if all of its subtasks are closed
     (including pre-existing ones). If some remain open, leave the parent open
     and tell the user which ones.
4. If any write fails, stop, report exactly what succeeded and what didn't,
   and ask before retrying. Don't leave a half-written state unreported.

### 9. Report

Finish with a short summary:

- The parent task as an inline markdown link, e.g.
  [Add OAuth login to the RCDS portal](https://app.clickup.com/t/abc123) —
  never a bare URL.
- Subtasks created vs. reused.
- Log mode: total time logged, and which tasks were marked complete.
- Anything you skipped or flagged (unmerged PR, open subtasks left as-is,
  pre-existing time entries you didn't duplicate).
