# django-allauth

```toml
# pyproject.toml
dependencies = [
    "django-allauth[mfa]",  # drop [mfa] if 2FA not selected
]
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    "allauth",
    "allauth.account",
    "allauth.mfa",  # only if 2FA selected
]

MIDDLEWARE = [
    # ...
    "allauth.account.middleware.AccountMiddleware",  # required, must come after AuthenticationMiddleware
]

# allauth 0.65+ — the old ACCOUNT_AUTHENTICATION_METHOD / ACCOUNT_EMAIL_REQUIRED /
# ACCOUNT_USERNAME_REQUIRED keys are deprecated and emit warnings on startup.
ACCOUNT_LOGIN_METHODS = {"email"}
ACCOUNT_SIGNUP_FIELDS = ["email*", "password1*", "password2*"]
```

```python
# config/urls.py
urlpatterns = [
    # ...
    path("accounts/", include("allauth.urls")),  # auto-mounts MFA at accounts/2fa/ when allauth.mfa is installed — no separate include
]
```
