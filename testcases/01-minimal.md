# 01 — Minimal example

Smallest path that boots a working Django project. Baseline — if this fails, everything else is moot.

## Prompt

```
/cookiecutter

Project name: 01-minimal-blog
Purpose: a tiny blog to verify the skill works end-to-end.

Settings layout: single file (`config/settings.py`).
Database: SQLite.
Local dev mode: uv on host.
Lint with Ruff: no.
Add-ons: none.
Production setup: skip.

Run the foundation, the boot check (migrate + createsuperuser), and confirm /admin/ login works.
```

## Expected outcome

- `uv run manage.py runserver` boots without errors.
- `/admin/` renders and the superuser can log in.
- Files present: `pyproject.toml`, `uv.lock`, `manage.py`, `config/settings.py`, `db.sqlite3`, `.env`, `.gitignore`.
- No Docker, no Postgres deps, no Ruff config.

## Run

```sh
# Run from a scratch parent dir; the skill creates `01-minimal-blog/` via `uv init`.
# AI executes the skill here, then:
cd 01-minimal-blog
uv run manage.py runserver &
curl -sf http://127.0.0.1:8000/admin/login/ > /dev/null
```

## Check report

**Execute this command yourself before stopping. Do not present it as a "next step" for the user — the testcase isn't done until the review file exists.** It runs an independent review (the model that built the project shouldn't grade its own output) and writes the result to `REVIEW.md` in the project dir.

```sh
claude -p \
  --model claude-opus-4-7 \
  --allowedTools "Read,Grep,Glob,Bash(ls:*),Bash(cat:*),Bash(rg:*)" \
  "Audit the existing code in this directory. Do NOT create, generate, or modify any files — read-only review only. Do NOT invoke any skill (especially cookiecutter). List bugs, inconsistencies with Django best practices, and concrete fixes. Be brief, top issues first." \
  | tee REVIEW.md
```

Paste the output below.

- What worked out of the box:
- What broke:
- Fixes applied:
- Suggested skill changes:

## Cleanup

Leave the code in place; no external resources to remove (SQLite file lives inside the project dir).
