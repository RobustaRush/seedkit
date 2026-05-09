# Devcontainer

Optional foundation-level add-on. Adds `.devcontainer/devcontainer.json` so VS Code / Codespaces / JetBrains Gateway open the project in a pre-configured container with `uv`, the right Python, and (if Docker dev mode was chosen) the project's compose stack already running.

Skip this when the user works in a plain shell — there's nothing else to set up.

## When to apply

- `local dev mode = uv on host`: the devcontainer wraps a thin Python image, runs `uv sync` on attach, and points VS Code at the local checkout.
- `local dev mode = docker-compose`: the devcontainer reuses the existing `docker-compose.yml` so VS Code attaches to the `web` service — no parallel container.

The two flavours share the same `devcontainer.json` shape but with different `image` / `dockerComposeFile` blocks.

## `.devcontainer/devcontainer.json` — uv-on-host

```json
{
  "name": "{{project_name}}",
  "image": "mcr.microsoft.com/devcontainers/python:3.12-bookworm",
  "features": {
    "ghcr.io/va-h/devcontainers-features/uv:1": {}
  },
  "postCreateCommand": "uv sync --frozen",
  "containerEnv": {
    "DJANGO_SETTINGS_MODULE": "config.settings.local"
  },
  "forwardPorts": [8000],
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "charliermarsh.ruff",
        "batisteo.vscode-django"
      ],
      "settings": {
        "python.defaultInterpreterPath": "${containerWorkspaceFolder}/.venv/bin/python",
        "python.testing.pytestEnabled": true
      }
    }
  }
}
```

## `.devcontainer/devcontainer.json` — docker-compose dev mode

```json
{
  "name": "{{project_name}}",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "web",
  "workspaceFolder": "/app",
  "shutdownAction": "stopCompose",
  "postAttachCommand": "uv run manage.py migrate",
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-python.python",
        "ms-python.vscode-pylance",
        "charliermarsh.ruff",
        "batisteo.vscode-django"
      ],
      "settings": {
        "python.defaultInterpreterPath": "/app/.venv/bin/python"
      }
    }
  }
}
```

When the project uses Docker structure `override` (`docker-compose.yml` + `docker-compose.override.yml`), the `dockerComposeFile` array should list **both** so the dev layer applies:

```json
"dockerComposeFile": ["../docker-compose.yml", "../docker-compose.override.yml"]
```

## Ruff / pyright / pre-commit

Add the matching extensions only if the corresponding add-on was chosen — don't ship `charliermarsh.ruff` if Ruff wasn't selected. Same for `ms-python.mypy-type-checker` (skip if pyright instead).

## Pitfalls

- `forwardPorts` is dev-only — don't add 5432 / 6379 unless the user wants Postgres / Redis exposed *from inside* the editor (e.g., for a DB GUI). Most don't need it.
- Don't bake secrets into `containerEnv`. Devcontainer files are committed; the project's `.env` is not, and that's where credentials belong.
- Don't pin the devcontainer Python image to a different minor than `pyproject.toml`'s `requires-python`. Drift here is the most common "works locally, breaks on the host" failure.
- `postCreateCommand` runs on container creation; `postAttachCommand` runs every time the editor reattaches. `migrate` belongs in `postAttach` so schema changes pulled from main apply automatically.
