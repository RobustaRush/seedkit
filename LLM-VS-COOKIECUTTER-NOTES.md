# Notes: can an LLM skill replace cookiecutter?

Raw material pulled from seedkit's own git history for a future article. Collected, not interpreted — every claim below cites a commit hash + date so it can be checked against `git show <hash>`. seedkit is a Claude Code skill that scaffolds Django projects conversationally (the LLM reads Markdown "references" and writes files) instead of stamping a Jinja template. Its history is unusually good source material because every bug the LLM shipped got caught by a *second*, context-free LLM review pass and turned into a commit message that names the exact failure.

**Dataset:** 228 commits, 2026-04-24 → 2026-07-05 (~10 weeks). A 9-case domain testcase suite (blog, shop, jobs-board, media-vault, mailer, crm, saas, startup, internal-ops) is run through `claude -p`, each output audited by a fresh `claude -p` that never saw the build conversation. Findings become commits. `REVIEW.md` (2026-07-04) is a later, more adversarial 8-pass audit of all 45 reference files against live upstream docs — it's the single richest document in the repo for this article.

---

## 1. Stale training knowledge — the LLM ships APIs that used to exist

The model's priors are frozen at training time; library APIs aren't.

- **allauth 0.65+**: LLM defaults to deprecated `ACCOUNT_AUTHENTICATION_METHOD` / `ACCOUNT_EMAIL_REQUIRED` / `ACCOUNT_USERNAME_REQUIRED` instead of current `ACCOUNT_LOGIN_METHODS` / `ACCOUNT_SIGNUP_FIELDS`. (`26.20.3`, 2026-05-13)
- **dj-stripe 2.9+**: whole webhook model changed from settings-based `DJSTRIPE_WEBHOOK_SECRET` + `WEBHOOK_SIGNALS` to admin-created DB rows with UUID URLs + `djstripe_receiver`. LLM wrote the pre-2.9 flow with total confidence. `REVIEW.md §1.2`: "the project is billing.md's stated Option B... webhook URL 404s, billing state never syncs." (`ad62202`, `bf64e05`, 2026-07-04)
- **django-csp**: legacy flat `CSP_*` keys raise `csp.E001` under django-csp 4.0+, which wants nested `CONTENT_SECURITY_POLICY = {"DIRECTIVES": {...}}`. Superseded again a version later when Django 6 shipped CSP in core — the fix for the fix was "stop installing the third-party package at all." (`26.20.3`; `26.27.6`, 2026-07-04)
- **Stripe SDK**: `stripe.error.SignatureVerificationError` (deprecated shim) instead of `stripe.SignatureVerificationError`. (`2a5f77a`, 2026-05-09)
- **allauth-2fa**: recommended an unmaintained third-party package that's incompatible with allauth ≥ 0.58, instead of the built-in `allauth.mfa`. (`2a5f77a`, 2026-05-09)
- **uvicorn**: `uvicorn.workers.UvicornWorker` is deprecated; current form is the standalone `uvicorn-worker` package. Caught in the systemic staleness sweep, not a targeted fix. (`REVIEW.md §2.1`, `ad62202`, 2026-07-04)
- **Litestream**: pinned to an abandoned 0.3.13 with a changed replica config shape (`replica:` singular in 0.5.x, new `linux-x86_64` asset naming). (`REVIEW.md §2.1`, `26.27.6`)
- **django-modern-rest**: entire guessed API was wrong — import name, `INSTALLED_APPS` entry, router shape. Real library: `import dmr`, no `INSTALLED_APPS` entry, `dmr.routing.Router` + `Controller.as_view()`. (`26.27.6`, 2026-07-04)
- **GitHub Actions**: `checkout@v4`, `setup-uv@v3`, `build-push-action@v5`, `codecov-action@v4` — each a major version behind, riding a soon-deprecated Node 20 runner. (`REVIEW.md §2.1`)

## 2. Hallucinated APIs — confidently wrong, not just outdated

Not drift from a real prior API — these never existed in any version.

- `Model.objects.afilter(...)` — Django has no such async ORM method. (`REVIEW.md §2.3`; fixed `ad62202`)
- `ZEAL_RAISE_ON_VIOLATION` / `ZEAL_SILENCED_WARNINGS` — real settings are `ZEAL_RAISE` / `ZEAL_ALLOWLIST`. Compounding error: the reference also claimed "zeal works automatically in pytest," which is false as wired — it never runs in tests at all. (`REVIEW.md §2.3`, `1030551`)
- "daphne is needed for management commands and signal hooks" — false, and self-contradicting: with daphne actually in `INSTALLED_APPS`, `runserver` *does* serve WebSockets, directly contradicting another line in the same file. (`REVIEW.md §2.3`)
- "Caddy's default `read_timeout` is 60s" — no such default exists. (`REVIEW.md §2.3`)
- "`WHITENOISE_USE_FINDERS` doesn't exist" — it does; the reference asserted a real setting was fake. (`REVIEW.md §2.3`)
- `migrator.reset()` "rolls back the database" — it only rebuilds the in-memory migration graph. A test built on this belief would pass without ever exercising the `backwards()` path it claimed to test. (`0c89c4a`, 2026-05-10)
- "pytest-django builds the test DB once, then reuses it" — backwards; default behavior is create+destroy every run, and the shipped snippet was missing `--reuse-db`. (`REVIEW.md §2.3`)

## 3. Snippet drift — the model "improves" a verbatim instruction and breaks it

The most severe class, because it survives review passes that only check *presence* of a feature, not *fidelity* to the snippet.

- **Critical, security-grade**: a custom-user-model admin form set `field_classes = {}` to strip the `username` mapping, but that also stripped `ReadOnlyPasswordHashField` — the admin's password input silently became a writable plain `CharField`. Saving the user form in admin overwrote `user.password` in cleartext and broke login. (`bb937a7`, 2026-05-09 — filed under "CRITICAL")
- **Settings redeclaration**: agents repeatedly rewrote `MIDDLEWARE` / `EMAIL_BACKEND` / `DATABASES` wholesale in `production.py` instead of appending deltas to the imported `base.py` list — silently dropping WhiteNoise's insertion, CSP middleware, or earlier settings. Recurring enough that the fix had to show three separate append forms (`+=`, `.insert(idx, ...)`, explicit rationale) instead of one prose sentence. (`4cd93c3`, 2026-05-08; recurs `8ab0911`, 2026-05-11)
- **`.env.example` inline comments treated as data**: the model wrote `redis://redis:6379 # cache /0`, and `django-environ` parsed the comment as part of the literal URL string — because nothing in the snippet said comments aren't supported. (`bb937a7`, 2026-05-09)
- **Missing the third file of three**: split-settings changes updated `manage.py` and `wsgi.py` but consistently forgot `asgi.py` — because the instruction was one prose line naming "the entry points" instead of three separate snippets. (`9683da7`, 2026-05-10)
- **`JOB_CLASS` misplaced**: agent nested a django-rq setting one level too deep (`RQ_QUEUES[queue]["JOB_CLASS"]` instead of top-level `settings.RQ`) — plausible-looking, but `get_job_class()` only reads the top-level key, so the worker silently fetched jobs as the wrong class and every task exploded with `'Task' object is not callable`. (`bb937a7`, 2026-05-09 — filed "CRITICAL")
- **Celery autodiscovery smoke check false-passed**: an agent-improvised check printed `celery_app.tasks` without first calling `import_default_modules()`, so it only ever listed built-in `celery.*` tasks and reported success even when the real task wasn't registered. (`42ff778`, 2026-05-11)

## 4. Security defaults the model reaches for by default

Not malice — just the path of least resistance in a scaffold-writing context.

- `docker-compose`'s bare `"5432:5432"` port publish resolves to `0.0.0.0`, not loopback — exposes `postgres/postgres` to the whole LAN on a dev machine. Same pattern independently caught for Mailpit's SMTP port (open relay risk). (`42ff778`, 2026-05-11; `aaa65d0`)
- `django-axes` lockout key as a flat list `['ip_address', 'username']` (OR semantics) instead of the nested `[['ip_address', 'username']]` (AND) — the flat form lets an attacker lock out an arbitrary username by hitting it from many IPs (username-lockout DoS). (`26.27.6`, 2026-07-04)
- `SECURE_PROXY_SSL_HEADER` set unconditionally in front of a directly-exposed gunicorn lets any client spoof `X-Forwarded-Proto: https` and bypass the HTTPS redirect. Fixed by gating it behind an explicit `DJANGO_BEHIND_PROXY` env var. (`bb937a7`, 2026-05-09)
- `send_mail(recipient_list=[...])` exposes every recipient's address to every other recipient in the `To:` header — flagged by a Gemini review pass specifically as "a critical privacy leak" that the Sonnet-authored reference had shipped. (`bb937a7`, 2026-05-09)
- Django's own `get_random_secret_key()` draws from `string.printable[:64]`, which freely emits `$ & ( ) ' " \`. Those characters break shell-sourced `.env` files, Docker Compose `${VAR}` interpolation, and any `sed`-based key substitution — in one run a literal placeholder string ended up baked into a "real" secret key because the substitution step corrupted on a special character. Fixed by switching key generation to `secrets.token_urlsafe(50)` (alphabet `[A-Za-z0-9_-]`, no quoting ever needed). (`3ffd61e`, 2026-05-09)

## 5. Whole-system consistency — the model can't hold every file in view at once

`REVIEW.md §2.4` ("Cross-file contradictions") is entirely this category, caught in one late audit pass across 45 files that individually looked correct:

- `cors.md` sets `*_COOKIE_SECURE = not DEBUG`; `security.md` sets the same settings unconditionally `True` in `production.py` — no stated precedence, "whichever pastes last wins."
- `new-project.md` says point all three entry points (`manage.py`/`wsgi.py`/`asgi.py`) at production settings; `async.md`'s own table says only the deployed one should — leaving the untouched file pointed at an empty settings package (import-error trap).
- `deploy-vps.md` hardcodes `image: ghcr.io/{owner}/{project_slug}:latest`; `deploy-github-ssh.md` assumes `image: ghcr.io/${GITHUB_REPOSITORY}:latest` — mutually exclusive, one is always wrong for a given project. (Independently rediscovered and fixed later: `2add197`, 2026-07-05 — "point compose image at `${GITHUB_REPOSITORY}`, not scaffold-time `{owner}` placeholder.")
- `healthcheck.md` mandates a Fly `[checks]` block + `/readyz` gate in CI; `deploy-managed.md`'s own `fly.toml` sample has no checks section and the workflow polls `/healthz` instead.
- `tasks-celery.md`'s Beat schedule enqueues `{project_slug}.tasks.example_task` — a task the skill's own app-layout rule says is never created at that path. Worker logs unregistered-task errors from the first boot.

Two more of this shape from earlier in the history, each needing an explicit "don't restate — append" rule before it stopped recurring: WhiteNoise middleware position lost on settings redeclaration (§3 above), and a celery config comment saying "production" where it should say "local," which made *dev* celery workers boot against prod env vars and crash on missing `SECRET_KEY`. (`bb937a7`, 2026-05-09)

## 6. Env-var / settings footguns — silent wrong-value vs loud crash

The natural Python idiom (`env(default=None)`) is actively wrong for a settings-gating pattern, and the model reaches for it anyway:

- `env.db(default=None)` and `env.email_url(default=None)` don't return `None` — they crash on `AttributeError`/`TypeError` because they try to URL-parse `None`. Plain `env(default=None)` silently propagates `None` straight into a Django setting with no error at all. Fixed by adopting `env.NOTSET` everywhere in the gated-default idiom, which raises `ImproperlyConfigured` and *names* the missing variable. (`69550a4`, 2026-05-09 — became the canonical idiom referenced in every later reference file)
- An unconditional `INSTALLED_APPS += ["dbbackup"]` reads `env("AWS_ACCESS_KEY_ID")` with no default at *import* time — which fires during `collectstatic` in the Docker build stage (which runs with `DEBUG=True` and no real secrets yet), crashing the image build itself, not the running app. (`edf3c8e`, 2026-05-10)
- CI's placeholder `DJANGO_SECRET_KEY = "test-key"` tripped Django's own `security.W009` check inside the same workflow's `check --deploy --fail-level WARNING` step — the placeholder needed to be 50+ characters to pass the check it was only there to unblock. (`6396ad9`, 2026-07-05)
- A project shipping only empty `startapp` stubs makes `pytest` exit 5 ("no tests collected"), which some CI configurations treat as a hard failure — turning a perfectly fine fresh scaffold red on its first push. (`144aeac`, 2026-07-05)

## 7. Agentic execution failures — the model as an *operator*, not just an author

Distinct from knowledge gaps: these are the model driving a shell inside the build/test loop and getting process semantics wrong.

- **`kill %1` doesn't propagate.** An agent tested Celery with `uv run celery ... & ; sleep 8 ; kill %1 ; wait`. `kill %1` only signals the `uv run` wrapper process; `uv` does not forward signals to the forked 10-worker pool underneath it, which reparents to PID 1 and keeps running. `wait` blocked forever — the harness had to move cleanup out of the agent entirely, into a session-leader + PGID-sweep mechanism (`kill -- -$pgid`), because no prompt-level instruction fixes a wrong belief about POSIX signal propagation. (`e22ba8e`, 2026-05-10)
- **Compose polling loop livelocked for 27 minutes.** An agent, mid-build, improvised `docker compose ps --format json | python3 -c "json.load(sys.stdin)..."` in a retry loop. Compose v2.6+ emits newline-delimited JSON, not a JSON array; `json.load` raised `Extra data`, `2>/dev/null` silently swallowed the error, the loop's exit code stayed 1 forever, and the build phase hung against the watchdog ceiling. Root cause was invisible from outside the shell call — only a live debug of the running session caught it. (`323c088`, 2026-05-10)
- **Boot-check race.** `runserver &` followed immediately by `curl` fires before the WSGI listener is actually up, producing a flaky false-negative boot check — needed `--noreload` plus an actual poll loop, not a fixed `sleep`. (`a733a0e`, 2026-07-05; an earlier version of the same fix at `SKILL.md` §4, 2026-05-11)
- **Bare `python -c "..."` mail smoke test.** Runs without `DJANGO_SETTINGS_MODULE` set; `django.setup()` fails silently, `send_mail()` no-ops, and the downstream Mailpit-count assertion then looks like a mailer bug rather than a test-harness bug. (`5553043`, 2026-05-11)

## 8. Model capability varies — this isn't one failure profile

From the project's own README ("Known failure modes") and testcase-triage commits — the same skill, same references, different models:

> "Smaller models drift back to the old API even when told not to. Haiku, in particular, will write `STATICFILES_STORAGE = "..."` instead of the `STORAGES = {...}` dict the skill explicitly demands... Sonnet does it about half as often. Opus rarely." (README, commit `817f583`, 2026-05-09)

> "Smaller models don't enumerate the menu... Weaker models skip the brief, skip whole categories, and present truncated menus that quietly hide options the user might have wanted." (same commit)

- Weaker/smaller-context models activated the skill but never actually read its reference files before writing code — fixed only by making the preflight rule explicit ("name the references the agent must read before the first tool call"), not by better prose. (`SKILL.md` fix, 26.20.2-era, 2026-05-12)
- A prose instruction ("ask about add-ons, then production") got read by smaller models as *discretion* and entire categories — email most consistently — were skipped even though the reference existed. The fix wasn't clearer prose; it was converting the section into an explicit numbered question list. (`25a8cbf`, 2026-05-09)
- The reviewer model is itself unreliable in the same way: one review pass flagged a correct `django.tasks.backends.immediate` import path as wrong, based on a stale belief about Django's task API — the maintainer had to override the reviewer, not the reference. (`8184787`, 2026-05-11)
- The harness's own status line: "The testcase harness currently runs only against Claude Sonnet... they may work, but skill quality on those models is not verified." (README, current)

## 9. Instruction-following: prose gets ignored, structure gets followed

A pattern the project converged on independently, then wrote down as house style (`seedkit/CLAUDE.md`): **negative instructions next to a correct positive sample get dropped** — the correct sample is followed, the paired warning is redundant weight the model doesn't need and sometimes second-guesses. Concretely:

- `20728c1`, 2026-05-10 — "drop don'ts where the correct sample is already shown": removed a `default=None` warning paragraph once the `env.NOTSET` snippet above it already showed the right form; same for pyright config warnings, a "don't delete tests.py" note once the snippet showed `git mv` instead.
- `6a8b665`, later — same principle applied project-wide: "drop redundant 'Don't X' warnings in references."
- The counter-evidence for *not* dropping negative guidance: it's kept only when there's no positive snippet to anchor it — e.g. "don't strip the Host-header check globally," a behavior rule with nothing to demonstrate.
- Reviewer calibration went through the same arc in the other direction — vague ("and Django best practices") review prompts produced speculative nitpicks (retries, defensive validation, "consider adding X" on a starter scaffold); tightening to explicit, bounded rules ("report only issues that prevent boot, fail a smoke check, or open a security hole... 'No issues found.' is a valid report") is what got six consecutive clean review passes on stable references. (`aac392d`, 2026-05-09)
- The same over-flagging problem recurred for *intentional* design choices (gated defaults, `wsgi.py` pointing at `production` while `manage.py` points at `local`) being repeatedly reported as bugs — needed an explicit "reviewer-silence preamble" listing them, and even with it, "about a third of audits flag at least one of them" anyway. (README "Known failure modes," `817f583`)

## 10. Version rot — the model's knowledge has a clock, and it doesn't know

`REVIEW.md §2.1` names this as *systemic*, not incidental — "every version pin is 1-2 years old; several snippets target APIs that have since been removed or deprecated" — bookworm→trixie base images, Redis 7→8, GitHub Actions majors behind, pre-commit hook IDs renamed (`ruff` → `ruff-check`), DaisyUI pinned to a tag likely to vanish. The fix wasn't re-pinning once — it was adding a standing rule to `SKILL.md`: *resolve current releases at generation time* instead of trusting the reference's literal pin. (`26.20.3`, `ad62202`, 2026-07-04)

## 11. What the skill's architecture does about all of the above

The structural responses, in the order they appeared in history:

- **Split references by chosen option**, not one omnibus file — cuts what the model has to hold in context per question from ~250-290 lines to ~30-95. Explicit rationale: smaller footprint means "local 8B models can run this without hating you." (`9e5e3a7`, 2026-05-08)
- **`Docs:` link on every reference** pointing at the upstream project's real documentation, so the agent has a way to check a claim against ground truth instead of trusting a possibly-stale snippet. (`3002ba8`, 2026-05-10)
- **"Snippet integrity" as an explicit rule**, not a hope: "the agent uses snippets verbatim... anything not in the snippet won't make it into the generated project" (seedkit/CLAUDE.md). Directly answers §3 above.
- **Two-phase build/review with no shared context** — the review agent never sees the build conversation, so it can't rationalize the build agent's shortcuts; this is the mechanism that surfaced nearly every bug cited above.
- **Harness-side process control, not agent-side** — moved cleanup (§7) entirely out of prompt instructions into deterministic session-leader + PGID sweeps once it became clear no wording fixes a wrong belief about signal propagation.
- **Generation-time freshness rule** for version pins (§10) — tells the agent to look something up rather than trust a static reference value, an explicit admission that a written-down pin *will* be stale by the time it's used.
- **Reviewer-silence preamble + explicit boot/security/smoke-only scope** — a calibration mechanism for the failure mode where an LLM reviewer either drowns real bugs in nitpicks or repeatedly re-flags intentional design.

## 12. Open, unresolved, admitted in the project's own words

Not solved yet — worth keeping honest in the article:

- **Reproducibility is explicitly given up, not hidden.** The README's own comparison table: cookiecutter-django and copier are "reproducible (deterministic)"; this skill is "no (model + phrasing affect output)." Framed as a trade: copier gives determinism, "you pay for it by living inside the template author's mental model forever."
- **No re-run/drift-detection flow yet.** "If your project has drifted (renamed apps, edited settings layout) the agent will read the drift, ask about it, and sometimes guess wrong. We're collecting cases where that happens." (README)
- **Cross-file contradictions (§5) are flagged but not all fixed** — `REVIEW.md §2.4` items are open as of the last recorded status update (2026-07-04): "§2.2–§2.5, §3, §4 remain open."
- **The REST add-on choice is itself unstable ground**: `REVIEW.md §2.5` — the skill offers only two pre-1.0 REST frameworks (django-modern-rest at ~39k total downloads, django-bolt "under active development"), no DRF, no django-ninja — "a skill claiming senior judgment shouldn't put churn risk on the most load-bearing dependency without at least saying so."
- **Defaults still contradict the "production-ready" promise** in places: security settings, lint, and CI all default to *no* in the questionnaire, meaning a user who accepts every default gets a project that fails `manage.py check --deploy` out of the box. (`REVIEW.md §2.2`)
