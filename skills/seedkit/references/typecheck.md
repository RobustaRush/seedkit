# Type checking — pyright + django-stubs

Stock Django runs without static types. Pyright (the type checker that powers VS Code's Pylance) plus `django-stubs` adds inferred types to the ORM, querysets, request/response cycle, and common third-party packages. Same `pyrightconfig.json` is read by both CI and the editor.

Pick pyright over mypy because it's roughly 10× faster on a typical Django project and has stricter inference defaults; the trade is a less mature plugin ecosystem and slightly different rules around descriptors.

Skip this if your project doesn't use type hints — checking untyped code only produces noise.

## Install

```sh
uv add --dev pyright django-stubs
```

## Config

`pyrightconfig.json`:

```json
{
  "include": ["."],
  "exclude": ["**/migrations/**", "**/__pycache__", ".venv", "staticfiles"],
  "venvPath": ".",
  "venv": ".venv",
  "reportMissingImports": "error",
  "reportGeneralTypeIssues": "warning",
  "reportOptionalMemberAccess": "warning",
  "strictListInference": true
}
```

`pyproject.toml` — tell `django-stubs` which settings module to load:

```toml
[tool.django-stubs]
django_settings_module = "config.settings.local"   # or "config.settings" for single-file
```

## Run

```sh
uv run pyright
uv run pyright path/to/file.py    # one file
```

## CI

In `.github/workflows/test.yml`, before `pytest`:

```yaml
      - run: uv run pyright
```

Failing pyright should fail the build — type errors don't surface at test time.

## Pre-commit hook

If `references/pre-commit.md` is applied, add:

```yaml
  - repo: https://github.com/RobertCraigie/pyright-python
    rev: v1.1.380
    hooks:
      - id: pyright
```

## Pragmatics

- Migrations rarely type-check cleanly — already excluded above.
- Admin `ModelAdmin` subclasses often need `# type: ignore[attr-defined]` on lines that touch dynamically-added attributes.
- Use `if TYPE_CHECKING:` to import heavy types (factories, test fixtures) without runtime cost.
- `django-stubs` covers Django core, auth, contenttypes, sessions; most third-party packages don't ship stubs. Pyright will warn `reportMissingTypeStubs` — leave at `none` until the noise is worth solving.
