# CSP — Django 6 core

Django ≥ 6.0 ships CSP in core — don't install `django-csp`.

```python
# settings — import at the top of the file
from django.utils.csp import CSP

MIDDLEWARE += ["django.middleware.csp.ContentSecurityPolicyMiddleware"]

SECURE_CSP = {
    "default-src": [CSP.SELF],
    "script-src": [CSP.SELF, CSP.NONCE],
    "style-src": [CSP.SELF, CSP.UNSAFE_INLINE],
    "img-src": [CSP.SELF, "data:"],
    "frame-ancestors": [CSP.NONE],
}
```

Nonces: add `"django.template.context_processors.csp"` to the `TEMPLATES` context processors and mark inline scripts with `<script nonce="{{ csp_nonce }}">` — not `request.csp_nonce` (that's django-csp's API, not core's).

First deploy: use `SECURE_CSP_REPORT_ONLY` (same dict shape) with a `report-uri` directive, enforce after the reports run dry.
