---
name: refresh-my-skills
description: Sync the user's installed Codex skills in `~/.codex/skills/` with the canonical University of Idaho RCDS/IIDS Codex skill collection at `ui-iids/claude-skills` on GitHub (the `codex/skills/` directory). Shallow-clones the canonical repo, compares every canonical Codex skill against `~/.codex/skills/`, reports which skills are missing or out of date, then (after showing the summary) copies the canonical versions in, adding missing skills and replacing stale ones, and finishes by reminding the user to restart Codex so the refreshed skills load. Use when the user invokes `/refresh-my-skills`, asks to "refresh my skills", "update my skills", "sync skills from ui-iids", "pull the latest codex skills", "get the newest skills", or similar.
---

# Refresh my skills

You are syncing the user's installed **Codex** skills at `~/.codex/skills/`
with the canonical RCDS/IIDS Codex skill collection in
`https://github.com/ui-iids/claude-skills`, under its `codex/skills/`
directory. The canonical repo is the source of truth, and its version of a
skill always wins. Your job is to add what's missing, replace what's stale,
and leave everything else alone.

The same repo also has a `skills/` directory. Those are the **Claude Code**
versions (they reference `CLAUDE.md`, `TaskCreate`, and so on). Never install
them into Codex. Only `codex/skills/` is the source for this sync.

## Non-negotiables

- **The canonical repo wins.** When an installed skill differs from the
  canonical `codex/skills/` version, the canonical version replaces it. Never
  merge the two or hand-edit skill content.
- **Never delete local-only skills.** A skill in `~/.codex/skills/` that is not
  in the canonical `codex/skills/` is the user's own skill or one Codex ships.
  Leave it untouched and don't report it as a problem. In particular, never
  touch hidden directories such as `~/.codex/skills/.system/`, which Codex
  manages itself.
- **Show the summary before writing.** Present the add/replace list and get
  the user's go-ahead before copying anything. Replacing a locally modified
  skill destroys those local edits, so call out any skill whose installed
  copy differs from canonical.
- **Copy whole skill directories**, not just `SKILL.md`. A skill may ship
  supporting files (references, templates), so the directory is the unit.
- **Only write inside `~/.codex/skills/`.** Don't modify `~/.codex/config.toml`,
  `AGENTS.md`, or anything else under `~/.codex/`.
- **Never modify the canonical repo.** The clone is read-only scratch: no
  commits and no pushes, and it gets cleaned up at the end.

## Procedure

### 1. Locate the installed skills

1. The target is `~/.codex/skills/` (expand `~` to the user's home directory,
   e.g. `$HOME/.codex/skills`). If `CODEX_HOME` is set, use
   `$CODEX_HOME/skills` instead and say so.
2. Each installed skill is a directory containing a `SKILL.md`. If the skills
   directory doesn't exist yet, note that every canonical skill will be an
   **add**, and create the directory only when the user confirms the sync
   (step 4).

### 2. Fetch the canonical skills

1. Shallow-clone the canonical repo into a temporary directory outside any
   project:
   `git clone --depth 1 https://github.com/ui-iids/claude-skills.git <tmpdir>/claude-skills-canonical`
2. If the clone fails (offline, auth, repo moved), report the exact error
   and stop. Do not fall back to stale copies or guesswork.
3. The canonical Codex skills are the directories under
   `<clone>/codex/skills/<skill-name>/`, each containing a `SKILL.md`. If
   `codex/skills/` is missing from the clone, stop and tell the user. Don't
   substitute the Claude `skills/` directory.

### 3. Compare

For each canonical skill directory `codex/skills/<name>/`:

1. **Missing locally** (`~/.codex/skills/<name>/` doesn't exist) → mark **add**.
2. **Present locally** → compare the directory contents recursively
   (e.g. `diff -rq <clone>/codex/skills/<name> ~/.codex/skills/<name>`):
   - Identical → mark **up to date**, no action.
   - Any file differs, or files exist on only one side → mark **replace**.

Then present a summary table: skill name, status (add / replace / up to
date), and for replacements a one-line note of what differs (e.g. "installed
SKILL.md differs from canonical"). If everything is up to date, say so,
clean up the clone, and stop. No restart reminder is needed.

### 4. Sync

After the user confirms:

1. Create `~/.codex/skills/` if it doesn't exist.
2. For each **add**: copy the canonical skill directory into
   `~/.codex/skills/<name>/`.
3. For each **replace**: remove the installed skill directory, then copy the
   canonical one in its place, so files deleted upstream don't linger.
4. Do not touch skills marked up to date, local-only skills, or hidden
   directories.
5. `~/.codex/skills/` is not a git repo, so list exactly what was written:
   each added or replaced directory with its path.

### 5. Clean up and remind

1. Delete the temporary clone.
2. Report the final tally: N added, N replaced, N already up to date,
   N local-only skills left alone.
3. **Always end with the restart reminder** (whenever anything was added or
   replaced). Tell the user to restart Codex (exit and relaunch `codex`, or
   start a new session in the IDE extension or app), because skills are
   loaded at session start and the refreshed skills won't be available until
   they do.
