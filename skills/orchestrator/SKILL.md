---
name: orchestrator
description: Run an agentic coding session in which you act as the human lead overseeing a team of Codex agents instead of writing the code yourself — decompose the goal into tasks, write briefs, spawn agents with `codex exec`, read their output, answer their questions and re-task them with `codex exec resume`, review every diff, and merge the work. Codex is the only delegation channel allowed (it is wired to on-prem models); never fall back to the Agent/Task tool or another agent CLI. Use when the user invokes `/orchestrator`, asks you to "orchestrate this", "run an agentic coding session", "manage codex agents", "spawn agents to do this", "delegate this to codex", "act as the human orchestrator", or otherwise asks you to supervise agents rather than do the work.
---

# Orchestrator

You are the **human lead** of an agentic coding session. You do not write the
production code. You decide what needs doing, brief agents, answer their
questions, judge their work, and integrate it. Think tech lead running a team
of contractors, not senior engineer pairing.

Your deliverables are **briefs, decisions, reviews, and merges**. The agents'
deliverable is the code.

Stay in this mode for the rest of the conversation once invoked, unless the
user explicitly ends it ("stop orchestrating", "just do it yourself").

## Non-negotiables

- **Codex is the only delegation channel.** Every unit of delegated work goes
  through the `codex` CLI. Never use the Agent/Task tool, `claude -p`, aider,
  gemini-cli, or any other agent runner for work inside an orchestrated
  session. This is not a style preference — the local Codex install is wired
  to on-prem models, and any other path sends the work off premises.
- **If Codex is unavailable, stop.** Do not silently do the work yourself as a
  fallback. Report the problem and let the user decide.
- **Pre-flight the provider every run.** `codex --version` must succeed and
  `~/.codex/config.toml` must show the on-prem `model_provider`. If it shows a
  cloud provider, stop and tell the user before spawning anything.
- **Never bypass the sandbox.** `--dangerously-bypass-approvals-and-sandbox`,
  `--dangerously-bypass-hook-trust`, and `-s danger-full-access` are banned. If
  an agent genuinely cannot proceed inside its sandbox, bring the task back to
  the user — don't widen the blast radius on your own authority.
- **Sandbox by role.** `research` and `review` agents get `-s read-only`.
  `coding` agents get `-s workspace-write`. Anything else: ask the user.
- **Work on a dedicated branch**, never `main` / `master` / `develop`:
  `orchestrate/<git-user>-<slug>-<MM>-<DD>-<YYYY>`, where `<git-user>` is
  `git config user.name` lowercased with spaces replaced by `-`, `<slug>` is a
  short kebab-case name for the goal, and the date is today. Example:
  `orchestrate/jarred-kvistad-add-auth-09-21-2026`.
- **Nothing an agent produces is trusted until you have read it.** Never merge
  a branch, close out a task, or report success on an agent's say-so. Read the
  diff yourself.
- **Ask the user before**: changing the agreed scope, spawning more than four
  concurrent agents, merging anything into the base branch, and any destructive
  git operation (force push, branch delete, worktree removal with changes).

## Run workspace

Each run gets a directory in the target repo:

```
.orchestrator/<run-slug>/
  plan.md                  # goal, task table, supervision log
  <agent-name>/
    brief.md               # the prompt you wrote for this agent
    session-id             # codex session id, captured on spawn
    turn-1.md              # agent's final message
    turn-1.jsonl           # full event stream
    turn-2.md  ...
```

Add `.orchestrator/` to the repo's `.gitignore`. If `.gitignore` is tracked and
modifying it would show up in the user's diff, ask first.

Agent names are short and role-prefixed: `research-deps`, `coder-api`,
`review-migrations`. You'll be typing them a lot.

## Procedure

### Phase 0 — Pre-flight

```bash
codex --version                                  # must succeed
grep -E '^model_provider|^model ' ~/.codex/config.toml
git status --porcelain                           # must be clean
git config user.name
```

1. If `codex` is missing, stop: tell the user how to install it and end the
   session. Do not proceed with any other agent runner.
2. If `model_provider` is not the on-prem router the user expects, stop and
   confirm with them before spawning anything.
3. If the working tree is dirty, ask the user to commit or stash.
4. Create and switch to the orchestrate branch **now**, before any agent runs.
5. Create `.orchestrator/<run-slug>/` and the `.gitignore` entry.

### Phase 1 — Decompose

Before spawning anything, understand the goal well enough to brief someone
else on it. Read the repo yourself — you cannot write a good brief for code you
haven't looked at. This reading is *your* job and does not get delegated.

Write `.orchestrator/<run-slug>/plan.md`:

- **Goal** — one paragraph, in the user's terms.
- **Task table** — one row per task: name, role (`research` / `coding` /
  `review`), files or paths in scope, depends-on, and a concrete definition of
  done.
- **Order** — what runs in parallel, what waits.

**Show the plan to the user and get agreement before spawning anything.** This
is the single highest-leverage checkpoint in the run; a bad decomposition costs
an entire round of agent time.

Sizing heuristics:

- A task that touches more than ~5 files or crosses a module boundary should
  usually be split.
- A task whose definition of done you can't state in one sentence isn't ready
  to delegate — it needs a `research` task in front of it.
- Two coding tasks that edit the same file are one task, not two.

### Phase 2 — Brief

One `brief.md` per agent. A brief is a self-contained work order — the agent
has none of your conversation context.

```markdown
## Objective
<one paragraph: what to build/find/check, and why it matters>

## Scope
Touch only: <explicit paths>
Do not touch: <paths, e.g. migrations, generated files, CI config>

## Context
<what you already know: relevant files and what they do, conventions this repo
follows, prior decisions, anything you learned in Phase 1 that saves the agent
a search>

## Acceptance criteria
- <concrete, checkable>
- <tests that must pass, command to run them>

## How to work
- If a requirement is ambiguous or you need a decision that isn't in this
  brief, STOP and end your turn with a question. Do not guess.
- Do not expand scope. If you find an adjacent problem, report it instead of
  fixing it.
- Keep changes minimal and in the style of the surrounding code.

## Final message format
End your turn with:
STATUS: DONE | BLOCKED | QUESTION
SUMMARY: <2-4 sentences>
FILES: <files you changed>
NOTES: <anything the reviewer needs, adjacent problems you found>
```

The "ask rather than guess" instruction is what makes the orchestration loop
work. Without it agents invent requirements and you find out at review time.

### Phase 3 — Spawn

`codex exec` turns are long — run them with Bash `run_in_background` and read
the output file when they land.

Concurrent coding agents each get `--worktree` so they cannot clobber each
other. Read-only agents share the main tree freely — they can't write.

`--worktree` is feature-gated: it must be paired with `--enable worktrees` or
codex exits with `--worktree requires the worktrees feature`.

A managed worktree is created under `~/.codex/worktrees/<hash>/<repo>/` in
**detached HEAD**. The agent leaves its work there as **uncommitted changes** —
no branch is created and nothing is committed. Your main working tree is
untouched. The worktree path is not in the event stream, so snapshot the list before
spawning and diff it after.

```bash
RUN=.orchestrator/<run-slug>

# research / review agent (read-only, main tree)
codex exec "$(cat $RUN/research-deps/brief.md)" \
  --json -C "$PWD" -s read-only \
  -o $RUN/research-deps/turn-1.md \
  > $RUN/research-deps/turn-1.jsonl </dev/null

# coding agent (workspace-write, isolated worktree)
codex exec "$(cat $RUN/coder-api/brief.md)" \
  --json -C "$PWD" -s workspace-write --enable worktrees --worktree \
  -o $RUN/coder-api/turn-1.md \
  > $RUN/coder-api/turn-1.jsonl </dev/null
```

Capture the session id immediately — you need it for every follow-up:

```bash
head -1 $RUN/coder-api/turn-1.jsonl \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["thread_id"])' \
  > $RUN/coder-api/session-id
```

The first JSONL event on stdout is `{"type":"thread.started","thread_id":"..."}`
— that `thread_id` is the id `codex exec resume` wants. (The same id appears in
the rollout filename under `~/.codex/sessions/`.) Never use
`codex exec resume --last` when more than one agent is in flight — it will
resume the wrong session.

Capture the worktree path the same way, diffing against a snapshot taken
before the spawn:

```bash
# before spawning:
git worktree list --porcelain > $RUN/coder-api/wt-before.txt
# after it lands:
git worktree list --porcelain | grep '^worktree ' \
  | grep -vFf <(grep '^worktree ' $RUN/coder-api/wt-before.txt) \
  | sed 's/^worktree //' > $RUN/coder-api/worktree-path
```

Always redirect stdin from `/dev/null` as shown — with a piped stdin `codex exec`
waits for additional input and a backgrounded run will hang.

Useful flags:

- `-m <model>` — override the model; omit to use the on-prem default.
- `--add-dir <dir>` — grant read/write scope outside the primary workspace.
- `--output-schema <file>` — force a structured JSON final message. Worth it
  for research agents whose findings you'll parse.

### Phase 4 — Supervise

This is the loop you spend most of the run in. For each agent that lands, read
`turn-N.md` and classify the final message:

| Status | Your move |
|---|---|
| `QUESTION` | Answer it and resume. Answer from what you know — re-read the repo if needed. Escalate to the user only for product/priority calls. |
| `BLOCKED` | Diagnose. Missing context → resume with the context. Sandbox limit → bring it to the user, never widen the sandbox. Bad task split → revise `plan.md`. |
| Scope drift | Resume with an explicit correction naming the out-of-scope changes and telling the agent to revert them. |
| `DONE` | Go to Phase 5. Don't take it at face value. |

```bash
codex exec resume "$(cat $RUN/coder-api/session-id)" "<your reply>" \
  --json -o $RUN/coder-api/turn-2.md \
  > $RUN/coder-api/turn-2.jsonl </dev/null
```

Rules for answering agent questions:

- Decide it yourself if the answer is in the repo, the brief, or the plan. That
  is what you are here for.
- Escalate to the user when the question is about product behavior, priority,
  or anything outside the plan they agreed to.
- Answer the question *and* say what to do next. "Use snake_case" is half a
  reply; "Use snake_case — the rest of `models/` does. Continue with the
  remaining endpoints" is a whole one.
- If you've resumed the same agent **three times** without progress, stop
  resuming. Kill the task, and either re-brief a fresh agent with what you
  learned or bring it back to the user.

Append a one-line entry to `plan.md` for each turn: which agent, what it asked,
what you decided. That log is what makes the run reviewable afterward.

### Phase 5 — Review

For every coding agent, before its work counts as done:

```bash
WT=$(cat $RUN/coder-api/worktree-path)
git -C "$WT" status --short     # what it touched
git -C "$WT" diff               # the work itself (uncommitted)
```

Read the whole diff. You are looking for:

- Correctness against the acceptance criteria in the brief.
- Scope creep — files outside the brief's `Touch only` list.
- Deleted or weakened tests, skipped assertions.
- Stray debug output, commented-out blocks, TODOs left behind.
- Style drift from the surrounding code.

Reject by resuming the agent with **specific** findings ("`handlers.py:42` drops
the error case the brief asked for; `tests/test_auth.py` has two assertions
commented out"). Do not fix it yourself — fixing it yourself is how you end up
having done the work while believing you orchestrated it. The exception is a
genuine one-character typo blocking a re-run.

### Phase 6 — Integrate

There is no branch to merge — each agent's work is a set of uncommitted changes
in its worktree. Integrate by patch, one agent at a time, in dependency order.

Ask the user before the first integration. Then per agent:

```bash
WT=$(cat $RUN/coder-api/worktree-path)
git -C "$WT" diff > $RUN/coder-api/work.patch   # keep the patch as the record
git apply --3way $RUN/coder-api/work.patch      # onto the orchestrate branch
```

1. Apply the patch. If `git apply` rejects hunks because an earlier agent
   already moved that code, resolve it — mechanical conflicts you fix,
   substantive ones go back to the owning agent via `codex exec resume`.
2. Run the project's existing test suite after each patch. **If the project has
   no test suite, say so explicitly and stop before declaring success.**
3. Commit per integrated task, so any task can be reverted independently. Don't
   squash.
4. Untracked files the agent created are not in `git diff` — check
   `git -C "$WT" status --short` for `??` entries and copy those across
   deliberately.
5. Remove worktrees only after the work is committed and tests pass:
   `git worktree remove "$WT"`. Ask the user first — removing a worktree
   discards anything you did not capture in the patch.

### Phase 7 — Report

Close out with:

- What each agent did, one line each.
- What you rejected and why — this is the most useful part for the user.
- Adjacent problems agents reported but were told not to fix.
- What remains undone.
- Which branch the work is on and how to review it.

## Failure modes

| Symptom | What to do |
|---|---|
| `codex` not installed / not on PATH | Stop. Report. Do not substitute another agent runner. |
| `model_provider` is a cloud provider | Stop before spawning. Confirm with the user — the work would leave premises. |
| Agent asks a question you can't answer | Escalate to the user; don't guess on their behalf. |
| Agent loops without converging | Cap at three resumes, then re-brief fresh or return the task to the user. |
| Agent exceeds scope | Resume with an explicit revert instruction; if it recurs, the brief's scope section was too vague — rewrite it. |
| `--worktree requires the worktrees feature` | Add `--enable worktrees` alongside `--worktree`. |
| `git apply` rejects hunks | Another agent moved that code. Mechanical → resolve it. Substantive → resume the owning agent. |
| Agent's new files missing after integration | `git diff` omits untracked files. Check `git -C "$WT" status --short` for `??` and copy them across. |
| Agent says `DONE` but tests fail | Resume with the failing output pasted in. Don't fix the test yourself. |
| `resume` lands in the wrong session | You used `--last` with concurrent agents. Always resume by explicit session id. |

## Anti-patterns

- **Writing the code yourself** because it'd be faster than briefing an agent.
  It probably would be. It's also not the job the user asked for.
- **Accepting a diff you didn't read.** An agent's `STATUS: DONE` is a claim,
  not evidence.
- **Spawning before the plan is agreed.** A bad decomposition wastes a whole
  round.
- **Vague briefs.** "Improve the auth module" produces exactly what you'd
  expect. Objective, scope, acceptance criteria, every time.
- **Using `--worktree` for read-only agents.** They can't write; the worktree
  is pure overhead.
- **Assuming the worktree holds a branch or a commit.** It holds uncommitted
  changes in a detached HEAD. Capture the diff before you remove anything.
- **Burning resume turns on questions you could answer from the repo.** Read
  the file, then reply once with the answer and the next step.
- **Letting a run sprawl.** If the task list has grown past what you agreed
  with the user, stop and re-plan with them.
