---
name: seedkit
description: Set up a new Django project with only the components you need.
---

## Prerequisites

```sh
uv --version
```

## Reference files

### Foundation

- `references/uv.md` — uv installation and commands
- `references/new-project.md` — Two Scoops layout, django-environ, uv
- `references/database.md` — SQLite vs PostgreSQL (host or Docker)
- `references/custom-user.md` — custom `AUTH_USER_MODEL` (set before first migrate)
- `references/docker.md` — local docker-compose dev + production image
- `references/lint.md` — Ruff (Django-aware rules, optional pre-commit)
- `references/pytest.md` — pytest + pytest-django (replaces `manage.py test`); default no

### Add-ons

- `references/auth.md` — `django-allauth` (passwords, verification, social) or `django-mail-auth` (passwordless magic-link); ask, or none for stock auth
- `references/debug.md` — orbit / silk dispatcher → loads `debug-orbit.md` or `debug-silk.md`
- `references/redis.md` — Redis cache (django-redis)
- `references/storage-whitenoise.md` — static via WhiteNoise + media volume on VPS
- `references/storage-s3.md` — static + media on S3-compatible storage
- `references/tasks-celery.md` — Celery + Redis
- `references/tasks-django.md` — Django Tasks dispatcher → loads `tasks-django-db.md` or `tasks-django-rq.md`; optional `tasks-django-cron.md` for periodic
- `references/email.md` — transactional email (console / SMTP / Mailpit)
- `references/logging.md` — `structlog` JSON-in-prod / pretty-in-dev; ask yes / no
- `references/analytics.md` — dispatcher → loads `analytics-goatcounter.md` / `-umami.md` / `-shynet.md` / `-ga4.md`

### Production

- `references/security.md` — Django security settings
- `references/error-reporting.md` — Bugsink / Sentry SaaS / GlitchTip (sentry-sdk)
- `references/gdpr.md` — PII scrubbing, retention, user data export/delete
- `references/ci.md` — GitHub Actions test workflow
- `references/deploy-vps.md` — VPS with Docker + Caddy
- `references/deploy-managed.md` — Fly.io / Railway / Render
- `references/deploy-github-ssh.md` — GitHub Actions deploy via SSH

## Instructions

### 1. Overview

Before any question, send one short message summarising — in your own words, drawn from the Reference files list above — what the skill can set up: foundation, add-ons, production. Phrase it fresh each time. End with an invitation to begin.

### 2. Foundation — one question at a time, in order

For any question involving a third-party package, brief 1–2 sentences on *what it adds beyond stock Django* (from the reference's intro). Then ask.

1. Project name and a one-line purpose.
2. Settings layout: single `settings.py` or split `base/local/production`.
3. Database: SQLite or PostgreSQL.
4. Local dev mode: uv on host or docker-compose.
   - Postgres + uv-on-host → host Postgres or Postgres-only in Docker.
   - docker-compose → full stack.
   - SQLite + Docker — warn the file lives in a container volume.
5. Custom user model: yes / no. Decide now (see `references/custom-user.md`).
6. Lint with Ruff: yes / no.
7. Test runner: pytest + pytest-django, or stock `manage.py test`? **Default no** (stock). Apply `references/pytest.md` only if user picks pytest.

Never bundle questions.

### 3. Apply the foundation

Generate files from the matching references. `.env` `DATABASE_URL` must match DB + dev mode (sqlite path, host Postgres URL, or `db` service host).

If the user opted into a custom user model, apply `references/custom-user.md` **before** the boot check.

### 4. Boot check — mandatory

In the chosen mode:

- `migrate`
- `createsuperuser` (interactive)
- `collectstatic --noinput` only if a static-files add-on was applied; skip for the bare foundation (`runserver` serves statics in DEBUG).

Do not move on until the user confirms `/admin/` login works.

### 5. Add-ons — one question at a time, in this order

Ask every question below. Don't drop any. Same briefing rule as foundation: 1–2 sentences from the reference intro on what the package adds beyond stock Django, then the question. `none` is always a valid answer.

1. Auth: `django-allauth` / `django-mail-auth` / `none` (`references/auth.md`).
2. Debug: `django-orbit` / `django-silk` / `none` (`references/debug.md`).
3. Redis cache: yes / no (`references/redis.md`).
4. Static + media storage: `whitenoise` / `s3` / `none` (`references/storage-whitenoise.md`, `references/storage-s3.md`).
5. Background tasks: `celery` / `django-tasks-db` / `django-tasks-rq` / `none` (`references/tasks-celery.md`, `references/tasks-django.md`).
6. **Email backend: `console` / `smtp` / `mailpit` / `none` (`references/email.md`).** Always ask — every project sends mail eventually (password resets, error reports, allauth verification).
7. Structured logging (`structlog`): yes / no (`references/logging.md`).
8. Analytics: `goatcounter` / `umami` / `shynet` / `ga4` / `none` (`references/analytics.md`).

### 6. Production

Ask only after add-ons. Same briefing-then-ask rule for each.

1. Security settings: yes / no (`references/security.md`).
2. Error reporting: `bugsink` / `sentry` / `glitchtip` / `none` (`references/error-reporting.md`).
3. GDPR helpers: yes / no (`references/gdpr.md`).
4. CI on GitHub Actions: yes / no (`references/ci.md`).
5. Deploy target: `vps` / `managed` / `github-ssh` / `none` (`references/deploy-vps.md` / `-managed.md` / `-github-ssh.md`).

### 7. README

After any setup step, update `README.md` with the key decisions (stack, DB, dev mode, add-ons, deploy target) and the main commands (install, test, migrate, run, deploy). Don't hardcode dependency versions — read them from `pyproject.toml`.

### Don't improvise

Use reference snippets as written.

- Env vars: always `DJANGO_DEBUG` / `DJANGO_SECRET_KEY` / `DJANGO_ALLOWED_HOSTS`.
- Don't add packages the user didn't ask for (`django-extensions`, etc.).
- Don't create an app dir named after the project unless asked.
- Don't restate values in `local.py` / `production.py` that `base.py` already sets.
- Don't reimplement what `django-environ` does (no manual `.split(",")`, no leftover `import os`).
- After `uv init`, set `requires-python = ">=3.12"` in `pyproject.toml` (Django 6 supports 3.12+; the auto-detected pin from the host interpreter is too tight).
- When adding an add-on, append every env var its reference reads to `.env.example` so the file stays the canonical list.
- After inserting the env-driven `DATABASES = {...}` line in Option A, **delete** the original hardcoded `DATABASES` block + `# Database` comment that `startproject` emitted. Bottom wins; leaving both makes `DATABASE_URL` dead code. (Option B writes `base.py` from scratch, so this only applies to Option A.)
- `tasks.py` must live inside a registered Django app, never at project root and never under `config/`. Both Celery autodiscovery and `django-tasks` only scan `INSTALLED_APPS`. If no app exists yet, create one (`uv run manage.py startapp jobs`) before placing `tasks.py`.
- Use snippets verbatim. Don't drop lines that look "obvious" or "redundant" (e.g. `DEFAULT_AUTO_FIELD`, gated env defaults, `JOB_CLASS` in `RQ_QUEUES`). If a value looks unnecessary, the reviewer caught the same thing in the previous five runs — keep it.
