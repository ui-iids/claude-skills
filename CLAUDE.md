# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A curated collection of **Claude Code skills** authored for University of Idaho RCDS/IIDS
(Research Computing and Data Services). It contains no application code, build system, or
test suite — each top-level entry under `skills/` is one skill, and the deliverable is the
prose in its `SKILL.md`. "Working in this repo" means authoring, editing, and reviewing
these skill documents, not running software.

## Layout and authoring conventions

Each skill lives at `skills/<skill-name>/SKILL.md`. The directory name, the `name:` field,
and the invocation slug (`/<skill-name>`) are expected to match.

A `SKILL.md` has two parts:

1. **YAML frontmatter** with `name` and `description`. The `description` is the most
   important line in the file — it is what Claude reads to decide whether to load the skill,
   so it must be self-contained and end with explicit trigger phrases ("Use when the user
   invokes `/clean-code`, asks to 'clean up the codebase', ..."). Write descriptions as a
   single dense sentence-or-two that both summarizes the behavior *and* lists the user
   phrasings that should activate it. Study `clean-code`, `test-coverage`, and
   `rcds-brand-design` for the established style.
2. **Markdown body** — the actual instructions Claude follows when the skill is active,
   written as second-person directives ("You are doing...", "Never delete...").

Existing skills encode strong **operational guardrails** worth matching when you add or edit
one: work on a dedicated branch rather than `main`, commit per phase, ask before destructive
or non-trivial changes, and verify against the existing test suite before declaring success
(see `clean-code` and `test-coverage`). Keep that safety-first tone consistent across skills.

## Known inconsistencies (fix-on-touch, don't assume they're intentional)

- `git status` shows `skills/SKILL.md` as deleted and every skill directory as untracked —
  the repo was reorganized from a single root `SKILL.md` into per-skill directories that have
  not been committed yet.

## Commit messages

The git user is `Jarred6068`. Branch-name and date conventions referenced inside skills (e.g.
`refactor/<git-user>-cleanup-MM-DD-YYYY`) derive `<git-user>` from `git config user.name`
lowercased with spaces replaced by hyphens.
