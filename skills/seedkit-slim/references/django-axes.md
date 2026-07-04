# django-axes

Brute-force lockout. Wire in `base.py`:

```python
INSTALLED_APPS += ['axes']

# AxesMiddleware must be the LAST middleware — wraps every other middleware's auth attempts.
MIDDLEWARE += ['axes.middleware.AxesMiddleware']

# AxesBackend must be FIRST — wrong order silently disables lockout.
AUTHENTICATION_BACKENDS = [
    'axes.backends.AxesBackend',
    'django.contrib.auth.backends.ModelBackend',
]

AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = 1  # hours
# Nested list = AND (lock the attacking ip+username pair). A flat
# ['ip_address', 'username'] is OR — anyone can lock a known username out.
AXES_LOCKOUT_PARAMETERS = [['ip_address', 'username']]
AXES_RESET_ON_SUCCESS = True
```

Don't set `AXES_LOCKOUT_CALLABLE` — `axes.helpers.lockout_response` was removed in v8. The default lockout response is correct.

With allauth, the username dimension never fires out of the box (allauth posts the identifier as `login`, axes reads `username`) — either wire axes' allauth integration steps or lock on `['ip_address']` only. Plain `django-axes` resolves every request to the proxy IP behind Caddy/nginx — install `'django-axes[ipware]'` on proxied deploys.

In `production.py`, when Redis is in scope:

```python
AXES_HANDLER = 'axes.handlers.cache.AxesCacheHandler'
```
