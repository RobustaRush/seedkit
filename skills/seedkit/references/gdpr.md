# GDPR / privacy

Docs: <https://docs.sentry.io/platforms/python/data-management/sensitive-data/>

Concrete settings for projects with EU users or regulated data.

## Sentry SDK — strip PII

```python
def _scrub(event, hint):
    request = event.get("request") or {}
    headers = request.get("headers") or {}
    for h in ("Authorization", "Cookie"):
        headers.pop(h, None)
    return event

sentry_sdk.init(
    dsn=SENTRY_DSN,
    integrations=[DjangoIntegration()],
    send_default_pii=False,
    before_send=_scrub,
)
```

## Data residency

| Backend | Where data lives |
|---------|------------------|
| Bugsink / GlitchTip (self-hosted) | Your VPS |
| Sentry SaaS — EU region | de.sentry.io (choose at signup) |
| Sentry SaaS — US region | sentry.io |

## Retention

- Bugsink: `RETENTION_*` env vars per project.
- GlitchTip: per-organization in admin UI.
- App data (user models, audit log): periodic task that deletes records past retention.

## Analytics

| Backend | Cookies | Consent banner needed | Data residency |
|---|---|---|---|
| GoatCounter / Umami / Shynet (self-host) | no | no | your VPS |
| GoatCounter / Umami SaaS (EU region) | no | no | EU |
| Google Analytics 4 | yes | yes (EU) | US |

GA4 in the EU requires Google Consent Mode v2 with a CMP-driven banner. Load gtag with denied defaults, then update on consent:

```js
window.dataLayer = window.dataLayer || [];
function gtag(){dataLayer.push(arguments);}
gtag('consent', 'default', {
  ad_storage: 'denied',
  analytics_storage: 'denied',
  ad_user_data: 'denied',
  ad_personalization: 'denied',
});
// after the user accepts in your CMP:
// gtag('consent', 'update', { analytics_storage: 'granted', ... });
```

## Cookies / sessions

`SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE` — `references/security.md`.

Add `SESSION_COOKIE_SAMESITE = "Lax"` if not using cross-site auth.

## Logging

Don't log request bodies or `Authorization` headers. With structured logging add a filter that drops these keys.

## User data export & deletion

Management commands:

```sh
manage.py export_user_data <user_id> > data.json
manage.py delete_user <user_id>
```

Implement deletion as a transaction that cascades to user-owned rows and writes an entry to an immutable audit log.
