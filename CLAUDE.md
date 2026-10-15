# CLAUDE.md — Rules for agents working in this repo

This project is **document-driven**. The documents in `docs/` are the source of truth.
If code and a document disagree, the document wins (or the document is updated first — never silently).

## Mandatory reading order (before touching code)

1. `docs/01-PRD.md` — WHAT we are building and WHY. Scope and what is NOT included.
2. `docs/02-TRD.md` — WHAT WITH. Stack, APIs, closed technical decisions.
3. `docs/03-APP-FLOW.md` — HOW it is used. CLI commands, report screens and what each one leads to.
4. `docs/04-DESIGN-BRIEF.md` — HOW it looks. Report color tokens, typography, components.
5. `docs/05-BACKEND-SCHEMA.md` — WHERE data lives. Files and/or tables, fields, relationships.
6. `docs/06-IMPLEMENTATION-PLAN.md` — IN WHAT ORDER. Numbered steps with a "done" criterion.

## Working rules

- **Everything in the repo is written in English** — code, comments, docs, commit messages, report copy,
  test names, judge prompts and the reports the harness produces. (The owner may chat in Spanish;
  the repository is still English-only.)
- **Work one plan step at a time.** State which step (`P3.2`, etc.) you are executing.
- **Do not invent.** If something is not defined in the docs, STOP and ask. Do not add libraries,
  commands, files or features that are not in the PRD/TRD.
- **When a step is finished:** tick its checkbox in `06-IMPLEMENTATION-PLAN.md`, verify its
  "done" criterion and summarize what changed in 2-3 lines.
- **If a decision changes** (e.g. a different library), first propose the change in the relevant doc,
  add an entry to the TRD "Decision log", and only then change the code.
- **Commits:** Conventional Commits, one commit per plan step, descriptive body, every commit green
  (tests + lint).
- **This repository is PUBLIC.** Before every commit, review `git diff --cached` and make sure none of
  these get in: keys or tokens (`sk-`…), `.env*` files (except the empty `.env.example`), local run
  data (`data/`, `runs/`, `traces/`, `reports/`), logs, absolute paths from my machine, emails or
  personal data. If in doubt, do not commit — ask.
- **Secrets:** never in code, tests, fixtures or commits. Only in `.env.local` (and `.env.example`
  without values). Model responses recorded as fixtures are reviewed before committing.
- **Money:** ask before any command that spends API money in bulk (full eval runs, judge calibration).
- **Tests:** every step with logic gets at least one test. TDD whenever feasible.

## Workflow with the ECC plugin

| When | ECC command / skill |
|---|---|
| Plan a specific step before coding | `/ecc:plan` (waits for your confirmation) |
| Implement tests-first | `ecc:tdd-workflow` skill |
| Review work before committing | `/ecc:code-review` |
| Security review (keys, inputs) and before every push | `ecc:security-reviewer` agent |
| Review the storage schema | `ecc:database-reviewer` agent |
| Accessibility of the HTML report | `ecc:a11y-architect` agent |
| Save state at the end of a session | `/ecc:save-session` |

## Project commands

Defined in `docs/02-TRD.md` once the stack is set up.
