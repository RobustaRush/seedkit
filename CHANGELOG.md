# Changelog

Versioned `YY.WW.D` — `date +%y.%V.%u` — year / ISO week / ISO weekday. One section per day; all of a day's commits collapse into one block. Trim to ≤ 200 lines; git keeps the rest.

## 26.20.1 — 2026-05-11

### Changed
- SQLite mini-prod defaults: when DB=SQLite, Foundation §3 writes the WAL/IMMEDIATE PRAGMAs into `production.py`; §5.3 cache backend defaults to `sqlite` (separate `cache.sqlite3` + `CacheRouter`); §5.4 background tasks default to `django-tasks-db`. The "optional" framing on the SQLite-in-production block in `database.md` is gone.
- Testcase 07 rewritten as the SQLite mini-prod exemplar (VPS + Caddy + single-stage Dockerfile + Litestream + `cache.sqlite3` + `django-tasks-db` + Sentry). Postgres/Redis/Celery coverage stays via cases 02/03/04/06/08/09. README adds a `Cache backend` variation dimension.
- §6 deploy: the `django-dbbackup` question now applies to `github-ssh` too (both `vps` and `github-ssh` deploy to self-managed hosts); skip for `managed` only, or for SQLite-on-VPS when Litestream is wired. Testcase 09 turns dbbackup on to keep coverage that moved out of 07.
- Boot-check flow split into §4 Foundation smoke (agent-driven: `migrate` + curl `/admin/login/` — no user keystrokes) and §7 Final smoke (user-driven: `createsuperuser` + browser login at the end, using task-runner names from §5.1 if one was picked). Avoids teaching `uv run manage.py …` muscle memory that §5.1 immediately replaces.
- SQLite + Docker: `docker.md` wires a named `sqlite_data:/data` volume on `web` (simple and override paths) with `DATABASE_URL=sqlite:////data/site.sqlite3`; the Foundation-step-4 warn is gone.
- Bootstrap is uv-on-host only; Docker is a runtime layer (`docker.md`).
- Settings split ships `base / local / production / test.py` from day one.
- `env.NOTSET` replaces `default=None` for the prod branch across env-driven settings.
- Skill prose trimmed: positive samples replace "don't" warnings; per-reference `Docs:` links instead of usage tutorials.

### Added
- Two-phase testcase runner: build and review agents see different contexts.
- `review-logs.sh` harness that loops over per-run logs, applies short skill fixes, and rolls a daily changelog bullet.
- `seedkit-examples` sibling submodule for committable generated reference projects.
- Task runner question (`mise` / `just` / `make` / `poe`) under Developer Experience.

### Fixed
- Wait-for-services rules: `docker compose up -d --wait` rather than hand-rolled `docker compose ps --format json` polling.
- CSP middleware appends to `MIDDLEWARE`; re-declaration drops WhiteNoise.
- Sentry init lives in `production.py`, not `base.py`.
- `django-bolt` builder image: full uv bookworm + `build-essential` (no aarch64-linux wheel).
- `setup.cfg [django_migration_linter] exclude_apps` uses app labels, not INSTALLED_APPS names.
