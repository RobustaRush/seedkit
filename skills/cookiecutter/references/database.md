# Database

`django-environ` parses `DATABASE_URL` directly — no extra Django config beyond `DATABASES = {"default": env.db("DATABASE_URL")}`.

## SQLite (default, zero-setup)

`.env`:

```sh
DATABASE_URL=sqlite:///db.sqlite3
```

No extra dependency. The file lives next to `manage.py`. Add `db.sqlite3` to `.gitignore`.

## PostgreSQL

```sh
uv add 'psycopg[binary]'
```

### Variant A — Postgres on host

Create the DB once:

```sh
createdb {project_slug}
# or:  psql -c "CREATE DATABASE {project_slug};"
```

`.env`:

```sh
DATABASE_URL=postgres://{user}:{password}@localhost:5432/{project_slug}
```

### Variant B — Postgres in Docker, Django on host

Run only the `db` service from `references/docker.md`:

```sh
docker compose up -d db
```

`.env` (host connects to the published port):

```sh
DATABASE_URL=postgres://postgres:postgres@localhost:5432/postgres
```

### Variant C — full stack in docker-compose

See `references/docker.md` "Local development". `.env` uses the service hostname:

```sh
DATABASE_URL=postgres://postgres:postgres@db:5432/postgres
```
