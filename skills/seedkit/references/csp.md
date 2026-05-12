# Content Security Policy — django-csp

Docs: <https://django-csp.readthedocs.io/>

Layer **on top of** `references/security.md`. Django's security settings cover headers like HSTS, X-Frame-Options, secure cookies; CSP is a separate header that browsers enforce against script / style / image sources. Production-only — too strict for `runserver` without careful tuning.

Apply only when the user said yes to security AND chose to harden CSP.

## Install

```sh
uv add django-csp
```

## `production.py` only

Append the middleware — don't re-declare `MIDDLEWARE`. Re-declaration drops anything `base.py` inserted (e.g. WhiteNoise after `SecurityMiddleware`):

```python
MIDDLEWARE = [*MIDDLEWARE, "csp.middleware.CSPMiddleware"]

CONTENT_SECURITY_POLICY = {
    'DIRECTIVES': {
        'default-src': ("'self'",),
        'script-src': ("'self'",),
        'style-src': ("'self'", "'unsafe-inline'"),  # tighten by removing unsafe-inline once styles are externalized
        'img-src': ("'self'", 'data:'),
        'font-src': ("'self'",),
        'connect-src': ("'self'",),
        'frame-ancestors': ("'none'",),
        'base-uri': ("'self'",),
        'form-action': ("'self'",),
    },
}
```

## Common per-add-on overrides

Append every listed host to every listed directive when the corresponding add-on is active:

| Add-on | Directive | Add |
|---|---|---|
| `django-allauth` social providers | `connect-src`, `img-src` | provider domains (Google: `accounts.google.com`, etc.) |
| `analytics-ga4` | `script-src`, `connect-src`, `img-src` | `https://www.googletagmanager.com`, `https://www.google-analytics.com` |

GA4 expanded — both hosts go into all three directives:

```python
'script-src':  ("'self'", "https://www.googletagmanager.com", "https://www.google-analytics.com"),
'connect-src': ("'self'", "https://www.googletagmanager.com", "https://www.google-analytics.com"),
'img-src':     ("'self'", 'data:', "https://www.googletagmanager.com", "https://www.google-analytics.com"),
```

| `analytics-umami` (self-hosted) | `script-src`, `connect-src` | the Umami host |
| `analytics-goatcounter` | `script-src`, `connect-src` | `https://gc.zgo.at` |
| `storage-s3` (uploaded media in `<img>`) | `img-src` | the S3 / CDN host |
| `error-reporting-sentry` (browser SDK) | `connect-src`, `script-src` | the Sentry / GlitchTip ingest host |
| `django-tailwind-cli` | nothing (bundled CSS served from `'self'`) | — |

Don't speculatively add hosts — only when the matching add-on is in scope.

## Report-Only first

For the first deploy after adding CSP, prefer report-only mode so existing pages don't break:

```python
CONTENT_SECURITY_POLICY_REPORT_ONLY = CONTENT_SECURITY_POLICY  # mirror, don't reuse the dict reference
```

Drop the `_REPORT_ONLY` once the logs are clean.

## Pitfalls

- `'unsafe-inline'` in `script-src` defeats the entire point. Avoid it. If a third-party widget requires it, isolate the widget in an iframe with a separate, narrower CSP.
- Admin uses inline styles — `'unsafe-inline'` in `style-src` is currently necessary for `/admin/` to render correctly. Acknowledge this as a known concession.
- CSP doesn't apply to API responses (`Content-Type: application/json`) — browsers only enforce it on document responses. REST endpoints don't need exemption logic.
