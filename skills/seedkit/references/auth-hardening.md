# Auth hardening — django-axes + 2FA

Two follow-up questions to Auth. Fire **only when** `auth ≠ none` (without an auth flow there's nothing to harden).

- `axes`: brute-force / lockout protection. Default **yes** when auth is selected.
- `2fa`: TOTP via `allauth-2fa` (paired with `django-allauth`) or `django-otp` (paired with `django-mail-auth` or stock auth). Default **no** — opt-in.

## django-axes (lockout / brute-force)

Wraps the login path. Records every authentication attempt; locks an IP / username after N failures for a cool-off window.

### Install

```sh
uv add django-axes
```

### `base.py`

```python
INSTALLED_APPS = [
    # ...
    'axes',
]

MIDDLEWARE = [
    # ...
    # MUST be the last entry — wraps every other middleware's auth attempts.
    'axes.middleware.AxesMiddleware',
]

AUTHENTICATION_BACKENDS = [
    'axes.backends.AxesBackend',
    # then the project's existing backends, e.g.:
    'django.contrib.auth.backends.ModelBackend',
    # 'allauth.account.auth_backends.AuthenticationBackend',
    # 'mailauth.backends.MailAuthBackend',
]

# Sane defaults — adjust per project tolerance.
AXES_FAILURE_LIMIT = 5
AXES_COOLOFF_TIME = 1          # hours
AXES_LOCKOUT_PARAMETERS = ['ip_address', 'username']
AXES_RESET_ON_SUCCESS = True
```

`AxesBackend` must be **first** in `AUTHENTICATION_BACKENDS`. Wrong order silently disables lockout.

### Migrate

`axes` ships its own models:

```sh
uv run manage.py migrate
```

### Pitfalls

- Behind a reverse proxy the `IP_address` lockout dimension only works if `IPWARE` (or the equivalent) is reading `X-Forwarded-For`. With WhiteNoise / Caddy / nginx wired correctly this is fine; check by hitting the login form from a known IP and reading `axes_accessattempt`.
- `axes` writes one DB row per failed attempt. On heavy public-facing forms, switch the cache backend (`AXES_HANDLER = 'axes.handlers.cache.AxesCacheHandler'`) — requires Redis (`references/redis.md`).
- Don't skip `AxesMiddleware` — without it, lockouts are recorded but never enforced on response.

## 2FA

Pick the matching package for the chosen Auth flow:

- **`django-allauth`** → use **`allauth-2fa`** (`uv add allauth-2fa`). Adds a TOTP-based second factor inside the existing allauth login pipeline. Adds `allauth_2fa` to `INSTALLED_APPS`, `allauth_2fa.middleware.AllauthTwoFactorMiddleware` to `MIDDLEWARE`, and `allauth_2fa.urls` to the `accounts/` include. Users opt in from the account page.
- **`django-mail-auth` or stock auth** → use **`django-otp`** + `django-otp-totp` (`uv add 'django-otp[qrcode]'`). Adds `django_otp`, `django_otp.plugins.otp_totp` to `INSTALLED_APPS`, `django_otp.middleware.OTPMiddleware` to `MIDDLEWARE` (after `AuthenticationMiddleware`). Wire admin login through `django_otp.admin.OTPAdminSite` if 2FA on `/admin/` is wanted.

For both: run migrations, then ship a UI flow for the user to enrol a TOTP secret (allauth-2fa includes templates; `django-otp` does not — wire your own).

### Settings — `production.py` only when allauth-2fa is used

```python
# Only force 2FA in prod; leave it optional in dev so seeded superusers can log in.
ACCOUNT_LOGIN_BY_2FA_REQUIRED = True   # allauth-2fa specific
```

### Pitfalls

- 2FA + axes together: failed 2FA codes count as auth failures in axes. That's usually correct — verify the `AXES_LOCKOUT_PARAMETERS` aren't so aggressive that legitimate users get locked out for one mistyped code.
- `allauth-2fa` requires `django-allauth >= 0.55`; pin both.
- TOTP requires accurate server time. Clock skew > 30s on the host = users fail to authenticate. Document NTP requirement in `README.md`.
