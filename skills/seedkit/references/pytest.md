# Test runner — pytest

Stock Django ships `manage.py test` (unittest-based). `pytest-django` swaps in pytest: shorter test files, fixtures (`db`, `client`, `admin_client`, `django_user_model`), parametrization, better failure output. Adds two dev deps; doesn't change production code.

If you skip this, write tests with `django.test.TestCase` and run `uv run manage.py test`.

## Install

```sh
uv add --dev pytest pytest-django
```

## Config

`pytest.ini` (or `[tool.pytest.ini_options]` in `pyproject.toml`):

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.local
python_files = tests.py test_*.py *_tests.py
```

For a single-settings layout, point at `config.settings`.

## Run

```sh
uv run pytest
uv run pytest -k user        # only tests with "user" in the name
uv run pytest path/to/tests.py::test_signup
```

`pytest-django` builds a fresh `test_<dbname>` from migrations on first run, then reuses it. Add `--create-db` to force a rebuild after a schema change.

## Sample test

```python
# users/tests.py
import pytest
from django.urls import reverse

@pytest.mark.django_db
def test_admin_login(client, admin_user):
    client.force_login(admin_user)
    response = client.get(reverse("admin:index"))
    assert response.status_code == 200
```

## CI

`references/ci.md`'s GitHub Actions workflow runs `uv run pytest` after `uv sync --frozen`. The dev deps installed above are what makes that step actually succeed.
