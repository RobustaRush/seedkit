# GitHub Actions — Tests

## .github/workflows/test.yml

```yaml
name: test

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
      DJANGO_SECRET_KEY: test-key
      DJANGO_DEBUG: "False"
      DJANGO_ALLOWED_HOSTS: "*"

    steps:
      - uses: actions/checkout@v4

      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - run: uv sync --frozen

      - run: uv run pytest
```

Don't add a `manage.py migrate` step before `pytest`. `pytest-django` builds its own `test_<dbname>` database from migrations on each run — a pre-step migrate touches the service DB, doesn't affect the test DB at all, and pollutes state for any later step that reuses the same connection.
