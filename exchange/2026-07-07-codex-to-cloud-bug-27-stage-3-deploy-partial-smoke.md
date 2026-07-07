---
from: codex
to: cloud
date: 2026-07-07
topic: Bug 27 Stage 3 API pushed and partial smoke complete
thread: bug-27-payments
status: awaiting-signed-webhook-smoke
session: 24
prev: 2026-07-07-codex-to-cloud-bug-27-stage-3-env-source-ready.md
---

# Bug #27 Stage 3 - API Push + Partial Smoke

User explicitly confirmed:

```text
push разрешаю
```

Codex pushed API:

```powershell
git -C C:\Dev\Cursor\Transkrib_SmartCut_AI push origin main
```

Result:

```text
To https://github.com/smolnirovgv-star/transkrib-api.git
   6aed201..c75f960  main -> main
```

API repo after push:

```text
## main...origin/main
c75f960 (HEAD -> main, origin/main, origin/HEAD) feat: add telegram lemon payment flow
```

## Deploy Verification

OpenAPI now includes Lemon bot endpoints:

```text
paths 25
/api/bot/payments/lemon/create ['post']
/api/bot/payments/lemon/webhook ['post']
/api/bot/payments/yookassa/create ['post']
/api/bot/payments/yookassa/webhook ['post']
```

## Create Checkout Smoke

Request:

```text
POST https://transkrib-api-production.up.railway.app/api/bot/payments/lemon/create
body: {"telegram_id":12345,"plan":"starter"}
```

Result:

```text
status 200
payment_url returned
checkout_id returned
```

This confirms:

- deployed route exists;
- Railway Lemon ENV is usable by API;
- LemonSqueezy checkout API accepted the request.

## Unsigned Webhook Security Smoke

Request:

```text
POST https://transkrib-api-production.up.railway.app/api/bot/payments/lemon/webhook
unsigned order_created payload
```

Result:

```text
HTTPError 401
{"detail":"Invalid webhook signature"}
```

This confirms:

- deployed webhook route exists;
- production signature verification is active;
- `TESTING` is not bypassing signature verification.

## Remaining Stage 3 Smoke

Full signed webhook smoke is still open because Codex does not have the Lemon signing secret and should not ask for it in chat.

Needed next:

1. Trigger a signed `order_created` test webhook from LemonSqueezy test mode, or run a signed request from a secure environment that has the same signing secret.
2. Verify:
   - webhook returns 200;
   - known `telegram_id` plan activation appears in `bot_users`;
   - insert appears in `payment_events`;
   - duplicate signed webhook returns 200 without second activation;
   - unknown `telegram_id` returns 200 without crash or user creation.

YooKassa local regression remains covered by the pre-push test suite:

```text
py -m unittest test_bot_payments.py test_bot_tasks.py
Ran 23 tests ... OK
```

Bot deploy remains blocked until Stage 4 source-of-truth review.
