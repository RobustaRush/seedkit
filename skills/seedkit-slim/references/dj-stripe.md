# dj-stripe

dj-stripe ≥ 2.9: webhook endpoints are database rows — `DJSTRIPE_WEBHOOK_SECRET`, `DJSTRIPE_FOREIGN_KEY_TO_FIELD`, and a dashboard-registered `/stripe/webhook/` no longer exist.

- Create endpoints in the admin: dj-stripe → Webhook endpoints → Add (`/admin/djstripe/webhookendpoint/add/`). Saving registers the endpoint on Stripe; the URL gets a UUID suffix (`/stripe/webhook/<uuid>/`) and the signing secret lives on the row.
- Event handlers use the decorator, not the removed `WEBHOOK_SIGNALS` dict:

```python
from djstripe.event_handlers import djstripe_receiver

@djstripe_receiver("customer.subscription.created")
def on_subscription_created(sender, event, **kwargs): ...
```

- dj-stripe never sets the SDK global — before calling `stripe.*` yourself:

```python
from djstripe.settings import djstripe_settings
stripe.api_key = djstripe_settings.STRIPE_SECRET_KEY
```

- Local dev: `stripe listen --forward-to localhost:8000/stripe/webhook/<uuid>/`, then paste the CLI's printed `whsec_...` into the endpoint row's secret field.
