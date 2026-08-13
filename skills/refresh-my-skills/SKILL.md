---
name: refresh-my-skills
description: Sync the current project's local Claude Code skills with the canonical University of Idaho RCDS/IIDS skill collection at `ui-iids/claude-skills` on GitHub. Shallow-clones the canonical repo, compares every canonical skill against the project's `.claude/skills/`, reports which skills are missing or out of date, then (after showing the summary) copies the canonical versions in — adding missing skills and replacing stale ones — and finishes by reminding the user to restart their Claude session so the refreshed skills load. Use when the user invokes `/refresh-my-skills`, asks to "refresh my skills", "update my skills", "sync skills from ui-iids", "pull the latest claude skills", "get the newest skills into this project", or similar.
---

# Refresh my skills

You are syncing the skills in the user's **current project repo** with the
canonical RCDS/IIDS skill collection at
`https://github.com/ui-iids/claude-skills`. The canonical repo is the source
of truth: its version of a skill always wins. Your job is to add what's
missing, replace what's stale, and leave everything else alone.

## Non-negotiables

- **The canonical repo wins.** When a local skill differs from the
  `ui-iids/claude-skills` version, the canonical version replaces it. Never
  merge the two or hand-edit skill content.
- **Never delete local-only skills.** A skill that exists in the project but
  not in the canonical repo is a project-specific skill — leave it untouched
  and do not report it as a problem.
- **Show the summary before writing.** Present the add/replace list and get
  the user's go-ahead before copying anything. Replacing a locally modified
  skill destroys those local edits, so call out any skill whose local copy
  differs from canonical.
- **Copy whole skill directories**, not just `SKILL.md`. A skill may ship
  supporting files (references, templates); the directory is the unit.
- **Never modify the canonical repo.** The clone is read-only scratch — no
  commits, no pushes, and it gets cleaned up at the end.

## Procedure

### 1. Locate the project's local skills

1. Confirm you are inside a project repo (`git rev-parse --show-toplevel`).
   If not in a git repo, use the current working directory as the project
   root and say so.
2. The project's skills live at `<project-root>/.claude/skills/`, one
   directory per skill. If that directory doesn't exist yet, note that every
   canonical skill will be an **add**, and create the directory only when
   the user confirms the sync (step 4).

   > Special case: if the project root *is* a checkout of
   > `ui-iids/claude-skills` itself (check `git remote -v`), stop and tell
   > the user — this repo is the source of truth and syncing it onto itself
   > is a no-op. Suggest `git pull` instead.

### 2. Fetch the canonical skills

1. Shallow-clone the canonical repo into a temp/scratch directory:
   `git clone --depth 1 https://github.com/ui-iids/claude-skills.git <scratch>/claude-skills-canonical`
2. If the clone fails (offline, auth, repo moved), report the exact error
   and stop. Do not fall back to stale copies or guesswork.
3. The canonical skills are the directories under
   `<clone>/skills/<skill-name>/`, each containing a `SKILL.md`.

### 3. Compare

For each canonical skill directory `skills/<name>/`:

1. **Missing locally** (`.claude/skills/<name>/` doesn't exist) → mark **add**.
2. **Present locally** → compare the directory contents recursively
   (e.g. `diff -rq <canonical>/skills/<name> <project>/.claude/skills/<name>`):
   - Identical → mark **up to date**, no action.
   - Any file differs, or files exist on only one side → mark **replace**.

Then present a summary table: skill name, status (add / replace / up to
date), and for replacements a one-line note of what differs (e.g. "local
SKILL.md differs from canonical"). If everything is up to date, say so,
clean up the clone, and stop — no restart reminder needed.

### 4. Sync

After the user confirms:

1. Create `.claude/skills/` if it doesn't exist.
2. For each **add**: copy the canonical skill directory into
   `.claude/skills/<name>/`.
3. For each **replace**: remove the local skill directory, then copy the
   canonical one in its place — so files deleted upstream don't linger
   locally.
4. Do not touch skills marked up to date or local-only skills.
5. If the project is a git repo, show `git status` for `.claude/skills/` so
   the user can see exactly what changed. Do **not** commit — leave staging
   and committing to the user unless they ask.

### 5. Clean up and remind

1. Delete the scratch clone.
2. Report the final tally: N added, N replaced, N already up to date,
   N local-only skills left alone.
3. **Always end with the restart reminder** (whenever anything was added or
   replaced): tell the user to restart their Claude session — e.g. exit and
   relaunch `claude`, or run `/exit` and start a new session — because
   skills are loaded at session start and the refreshed skills won't be
   available until they do.
