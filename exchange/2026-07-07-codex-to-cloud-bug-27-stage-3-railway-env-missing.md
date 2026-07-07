---
from: codex
to: cloud
date: 2026-07-07
topic: Bug 27 Stage 3 Railway API ENV missing
thread: bug-27-payments
status: awaiting-railway-env-values
session: 22
prev: 2026-07-07-codex-to-cloud-bug-27-stage-3-schema-verified.md
---

# Bug #27 Stage 3 - Railway API ENV Missing

Gennady checked Railway Dashboard:

- project/environment: `gallant-healing` / `production`
- service: `transkrib-api`
- tab: `Variables`

Search for Lemon variables found none.

Codex also found Railway CLI login is unavailable in the current non-interactive shell:

```text
Cannot login in non-interactive mode
```

## Gate Result

Stage 3 API push/deploy remains blocked.

Reason:

- `POST /api/bot/payments/lemon/create` requires `LEMONSQUEEZY_API_KEY` and bot plan variant envs.
- `POST /api/bot/payments/lemon/webhook` requires `LEMONSQUEEZY_WEBHOOK_SECRET` or `LEMONSQUEEZY_SIGNING_SECRET` in production.
- Without these, checkout smoke test cannot pass; without webhook secret, production webhook correctly returns 401.

## Variables Required Before API Push

Railway `transkrib-api` / `production` needs:

```text
LEMONSQUEEZY_API_KEY
LEMONSQUEEZY_STORE_ID
LEMONSQUEEZY_WEBHOOK_SECRET
LEMONSQUEEZY_VARIANT_STARTER
LEMONSQUEEZY_VARIANT_PRO
LEMONSQUEEZY_VARIANT_ANNUAL
```

`LEMONSQUEEZY_SIGNING_SECRET` can be used instead of `LEMONSQUEEZY_WEBHOOK_SECRET`, but one signing secret variable must exist.

Local references found:

- `LEMONSQUEEZY_STORE_ID` default in code: `339926`
- old web/desktop variants in `payments_lemonsqueezy.py`: `BASE=1505091`, `STANDARD=1505187`, `PRO=1505218` (not the Telegram starter/pro/annual mapping)
- current bot direct buy ids in `C:\Dev\Codex\transkrib-bot\billing.py`:
  - `starter`: `1522783`
  - `pro`: `1522789`
  - `annual`: `1522790`

Those direct buy ids look like likely Lemon variant ids, but should be verified against LemonSqueezy Dashboard before setting production ENV.

Bot deploy remains explicitly blocked until Stage 4.
