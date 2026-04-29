# Analytics

Privacy-respecting site analytics. Pick one backend at setup time — only that snippet ships in templates. Empty `ANALYTICS_ID` disables tracking (e.g. in dev).

| Backend | Hosting | Cookies | Consent banner | Stack |
|---|---|---|---|---|
| GoatCounter (recommended) | self-host or SaaS | no | no | Go + SQLite |
| Umami | self-host or SaaS | no | no | Node + Postgres |
| Shynet | self-host | no | no | Django + Postgres |
| Google Analytics 4 | SaaS (US) | yes | **yes (EU)** | — |

GDPR / consent specifics — see `references/gdpr.md`.

## Django wiring

### .env

```sh
ANALYTICS_ID=
ANALYTICS_HOST=     # only for self-hosted GoatCounter / Umami / Shynet
```

### Settings

In `config/settings.py` (or `config/settings/base.py`):

```python
ANALYTICS_ID   = env("ANALYTICS_ID", default="")
ANALYTICS_HOST = env("ANALYTICS_HOST", default="")   # omit for GA4
```

### Context processor

`config/context_processors.py`:

```python
from django.conf import settings

def analytics(request):
    return {
        "ANALYTICS_ID":   settings.ANALYTICS_ID,
        "ANALYTICS_HOST": settings.ANALYTICS_HOST,
    }
```

Register in `TEMPLATES[0]["OPTIONS"]["context_processors"]`:

```python
"config.context_processors.analytics",
```

### Template

`templates/_analytics.html` — wraps the chosen backend's snippet. The `not debug` gate guarantees no beacons in dev even if `ANALYTICS_ID` is set:

```django
{% if ANALYTICS_ID and not debug %}
  {# backend snippet here — see one of the sections below #}
{% endif %}
```

In `templates/base.html`, before `</body>`:

```django
{% include "_analytics.html" %}
```

---

## Backend A — GoatCounter (recommended)

Tiny single Go binary, SQLite, cookieless. Lowest ops cost.

**SaaS:** free at goatcounter.com — `ANALYTICS_HOST=https://<code>.goatcounter.com`, `ANALYTICS_ID=<code>`.

**Self-host (docker-compose.prod.yml):**

```yaml
services:
  goatcounter:
    image: arp242/goatcounter:latest
    restart: unless-stopped
    environment:
      GOATCOUNTER_LISTEN: ":8080"
    volumes:
      - goatcounter_data:/home/user/db

volumes:
  goatcounter_data:
```

Reverse-proxy `stats.example.com` → `goatcounter:8080` in the Caddyfile.

**Snippet** (`templates/_analytics.html`):

```html
<script data-goatcounter="{{ ANALYTICS_HOST }}/count"
        async src="//gc.zgo.at/count.js"></script>
```

---

## Backend B — Umami

Polished UI, cookieless. Node + Postgres.

**SaaS:** umami.is.

**Self-host (docker-compose.prod.yml):**

```yaml
services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://umami:${UMAMI_DB_PASSWORD}@umami-db:5432/umami
      DATABASE_TYPE: postgresql
      APP_SECRET: ${UMAMI_APP_SECRET}
    depends_on: [umami-db]

  umami-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: ${UMAMI_DB_PASSWORD}
    volumes:
      - umami_pgdata:/var/lib/postgresql/data

volumes:
  umami_pgdata:
```

Reverse-proxy `stats.example.com` → `umami:3000`. Create a website in the Umami UI to get the website ID.

**Snippet:**

```html
<script defer src="{{ ANALYTICS_HOST }}/script.js"
        data-website-id="{{ ANALYTICS_ID }}"></script>
```

---

## Backend C — Shynet

Django + Postgres. Same stack as the project; sessions tracked without cookies via short-lived heartbeats.

**Self-host (docker-compose.prod.yml):**

```yaml
services:
  shynet:
    image: milesmcc/shynet:latest
    restart: unless-stopped
    environment:
      DATABASE_URL: postgres://shynet:${SHYNET_DB_PASSWORD}@shynet-db:5432/shynet
      DJANGO_SECRET_KEY: ${SHYNET_SECRET_KEY}
      ALLOWED_HOSTS: stats.example.com
    depends_on: [shynet-db]

  shynet-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shynet
      POSTGRES_USER: shynet
      POSTGRES_PASSWORD: ${SHYNET_DB_PASSWORD}
    volumes:
      - shynet_pgdata:/var/lib/postgresql/data

volumes:
  shynet_pgdata:
```

Reverse-proxy `stats.example.com` → `shynet:8080`. Create a Service in the Shynet admin; it generates the snippet — paste it into `_analytics.html` (the service ID becomes `ANALYTICS_ID`).

---

## Backend D — Google Analytics 4

SaaS, US-hosted. Cookies required. **EU users must show a consent banner before loading gtag** — see `references/gdpr.md`.

`ANALYTICS_ID=G-XXXXXXX` from analytics.google.com. `ANALYTICS_HOST` unused.

**Snippet:**

```html
<script async src="https://www.googletagmanager.com/gtag/js?id={{ ANALYTICS_ID }}"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', '{{ ANALYTICS_ID }}');
</script>
```
