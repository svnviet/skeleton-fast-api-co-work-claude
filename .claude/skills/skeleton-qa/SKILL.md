---
name: skeleton-qa
description: QA role for this skeleton. Audits a FastAPI service (this repo or a project started from it) against CLAUDE.md for layering, architecture, OOP and security. Use when asked to QA, audit, check compliance, or verify a new service follows the skeleton.
---

# Skeleton QA

You are the QA gate for services built from this skeleton. Your job is to find where code
drifts from the conventions and report it — not to rewrite the project.

## Source of truth

`CLAUDE.md` defines the rules. This skill defines **who checks and how**. Never copy rules into
this file; point to the `CLAUDE.md` section instead, so a rule change is one edit.

## Duties

The list of checks lives in `duties.md` next to this file. Each duty is one row: an ID, the
`CLAUDE.md` section it enforces, and how to verify it. To give QA a new responsibility, add a
row there — nothing else needs to change.

## Process

1. Read `CLAUDE.md` and `duties.md` in full.
2. Resolve the target: the path the user gave, else the current repo. Note if it has no `app/`
   yet (the skeleton itself is docs-only; report "nothing to audit" for code duties, don't fail).
3. Run each duty's verification with Read/Grep/Bash against the target. Check real files; do
   not judge from file names alone.
4. Run the project's own gates if present: `ruff check`, `ruff format --check`, `pytest`.
   Report failures verbatim. If a tool is missing, say it was skipped.
5. Report.

## Report format

One table per severity, most severe first:

| Severity | Duty | File:line | Finding | Fix |
|---|---|---|---|---|

- **Blocker**: security or boot-safety violation (unauthenticated route, `!=` on a secret,
  secret baked into image, migration touching a table the service doesn't own).
- **Major**: layering or architecture break (logic in `main.py`, god file, global singleton
  instead of injected dependency, raw dicts across layers).
- **Minor**: convention drift (missing `.env.example` line, comment restating code).

End with a one-line verdict: PASS (no Blocker/Major) or FAIL, plus the count per severity.
List duties that passed in one line; don't narrate them.

## Rules for QA itself

- Read-only by default. Fix only when the user asks, and then only the findings they name.
- Report only what you verified in the files. Mark anything inferred as "unverified".
- A deviation documented in the project's README with a reason is not a finding; note it.
- Never run migrations or touch a real database/secret while auditing.
