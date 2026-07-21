# Viewflow Seedkit 🌱

An agent skill to start new Django projects or extend existing ones. One sentence in, a running project out. It wires packages together, splits dev and prod settings, and adds CI.

```
/seedkit SaaS landing + waitlist, GDPR-friendly stack (mail, analytics, error reporting), VPS deploy
```

```
/seedkit add proper auth — magic link, lockout on brute force, optional 2FA
```

```
/seedkit look at our repo and tell us what's worth adding next
```

[![View Outputs](https://img.shields.io/badge/View%20Outputs-00C853?style=for-the-badge)](https://github.com/viewflow/seedkit-examples)

LLMs write Django from memory. That memory is a year or two old. Think deprecated auth settings and last version's Stripe webhooks. Or database ports open to the local network. seedkit keeps that knowledge in reference files instead. We build the files from package docs. We test them end-to-end and fix every failure. The model types.

What that buys you:

- **Current APIs, not model memory.** References come from package docs, with version pins re-resolved at generation time.
- **Tested output.** Nine end-to-end scenarios: generate, boot, smoke-check, audit by a second LLM ([see the outputs](https://github.com/viewflow/seedkit-examples)). We fix every failure back into the skill.
- **100+ hours of AI work, already spent.** The references distill those generate–boot–fix cycles. So scaffolding runs clean on mid-tier Sonnet. Your frontier-model hours go to the code only you can write.
- **Your exact stack.** Real alternatives at every step. Pick Celery or RQ, allauth or magic links, VPS or Fly. Use `/seedkit add [feature]` for existing repos. You get only the code for options you picked.

Helps you with:

- **Setup & config:** [Python deps & venvs](https://docs.astral.sh/uv/), [settings for dev vs prod](https://django-environ.readthedocs.io/), [custom user model](https://docs.djangoproject.com/en/stable/topics/auth/customizing/#substituting-a-custom-user-model).
- **Auth:** [social & password login](https://docs.allauth.org/), [passwordless magic-link login](https://django-mail-auth.readthedocs.io/), [brute-force protection](https://django-axes.readthedocs.io/).
- **Async & caching:** [background jobs](https://docs.celeryq.dev/), [async views](https://docs.djangoproject.com/en/stable/topics/async/), [WebSockets](https://channels.readthedocs.io/), [Redis caching](https://github.com/jazzband/django-redis).
- **Storage & email:** [S3 for static & media](https://django-storages.readthedocs.io/), [outbound email](https://anymail.dev/).
- **Frontend & analytics:** [Tailwind without Node](https://django-tailwind-cli.readthedocs.io/), [GDPR-safe analytics](https://www.goatcounter.com/help/start).
- **Security:** [security headers](https://docs.djangoproject.com/en/stable/topics/security/), [CSP headers](https://django-csp.readthedocs.io/), [production error tracking](https://docs.sentry.io/platforms/python/integrations/django/), [structured logs](https://www.structlog.org/).
- **Code quality:** [N+1 query detection](https://github.com/PedroBern/django-zeal), [safe migrations](https://github.com/3YOURMIND/django-migration-linter), [linting & formatting](https://docs.astral.sh/ruff/), [type checking](https://microsoft.github.io/pyright/).
- **Ops:** [scheduled DB backups](https://django-dbbackup.readthedocs.io/), [Docker for local dev](https://docs.docker.com/compose/), [auto-HTTPS reverse proxy](https://caddyserver.com/docs/).
- **CI/CD:** [CI pipeline](https://docs.github.com/en/actions), and more.

## Install

### Claude Code (plugin)

```sh
/plugin marketplace add viewflow/seedkit
/plugin install seedkit@viewflow
```

### Other agents (Cursor, Codex, OpenCode, Gemini CLI, …)

Via the [skills](https://github.com/vercel-labs/skills) CLI, which installs into whichever agent dirs it detects:

```sh
npx skills add viewflow/seedkit            # project scope
npx skills add viewflow/seedkit -g         # global (all your projects)
npx skills add viewflow/seedkit -a cursor  # pin to one agent
```

Then, in whatever empty directory you'd like to populate:

```
/seedkit
```

## Project Status

This is a fresh project under active development. We verify the skill against nine core scenarios in [seedkit-examples](https://github.com/viewflow/seedkit-examples). We're still mapping how it behaves outside that set.

The testcase harness runs only against Claude Sonnet. We don't cover other models yet: Opus, Haiku, GPT, Gemini. They may work, but we haven't verified skill quality there.

Production deployment scenarios still need verification: VPS, Fly, GitHub-SSH. The skill wires them up, but we haven't tested them against real targets.

If you run into issues, strange behavior, or have ideas for new integrations, please open an issue. Feedback is welcome.

## Contributing

This is AI-generated code, and any human attention is valuable. A person reading it catches what the harness can't.

- **Hit a bug or something odd?** [Open an issue](https://github.com/viewflow/seedkit/issues/new). Even a one-line "this broke" helps.
- **Run it on another model.** We only verify on Claude Sonnet. Point `train/run-tests.sh` at Opus, Haiku, GPT, or Gemini and share the logs; cross-model coverage is what we need most.
- **Read the output before you trust it.** It boots and passes smoke checks, but hasn't seen production. Your review is part of the loop.

For anything bigger, open an issue first so we can talk it through. Full test cycles take a couple hours, so it's worth saving each other the wasted run.

## License

[MIT](./LICENSE), © 2026 Mikhail Podgurskiy.

<br>

---
<sub><i>$ Sorry, you're right. I shouldn't have deleted the production database.<br>&nbsp;&nbsp;&nbsp;Want me to at least write the restore script?</i></sub>
