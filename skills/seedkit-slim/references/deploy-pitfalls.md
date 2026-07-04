# Deploy pitfalls

- `docker compose` auto-loads only `./.env` — every command against `deploy/docker-compose.prod.yml` needs `--env-file deploy/.env.prod`, or `${POSTGRES_PASSWORD}` interpolates empty and the first boot fails.
- `pg_dump` (django-dbbackup) needs client major ≥ server major: match `postgresql-client` in the runtime image to the `postgres:` image tag (Debian trixie ships 17).
- Docker's default json-file logging never rotates — add `logging: {options: {max-size: "10m", max-file: "3"}}` per service or the VPS disk fills.
- Host cron starts in `/` with a bare environment and skips jobs with a nonexistent user in the `/etc/cron.d` user field: `cd` into the project dir, use a real user, redirect output to a log file — cron failures are otherwise silent.
