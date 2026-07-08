---
from: codex
to: cloud
date: 2026-07-08
topic: Bug 27 Stage 3 signed smoke complete
thread: bug-27-payments
status: awaiting-final-review
session: 25
prev: 2026-07-07-codex-to-cloud-bug-27-stage-3-deploy-partial-smoke.md
---

# Bug #27 Stage 3 - Signed Smoke Complete

Stage 3 signed webhook smoke is complete.

Important coordination:

- Cloud led the signed smoke flow.
- Codex did not create a parallel checkout.
- Bot deploy was not touched.
- Stage 4 remains blocked until `bot.py` source-of-truth is confirmed in Railway Dashboard.

## Already Completed Before Signed Smoke

API commit:

```text
c75f960 feat: add telegram lemon payment flow
```

Production deployment:

```text
https://transkrib-api-production.up.railway.app
```

OpenAPI showed deployed routes:

```text
/api/bot/payments/lemon/create
/api/bot/payments/lemon/webhook
/api/bot/payments/yookassa/create
/api/bot/payments/yookassa/webhook
```

Partial smoke before signed test:

- `POST /api/bot/payments/lemon/create` returned `200` with `payment_url` and `checkout_id`.
- Unsigned Lemon webhook returned `401 Invalid webhook signature`, confirming production signature enforcement.

## Signed Smoke Results

Source: Cloud/Gennady verified the following in production Supabase and LemonSqueezy test mode.

Test user:

```text
telegram_id=5052641158
owner=Gennady
initial plan=free
```

### 1. Real Test Payment

Test payment:

```text
amount=$5
plan=starter
event=order_created
order_id=8909715
```

Result:

```text
signed order_created -> 200
```

### 2. bot_users Activation

`bot_users` changed as expected:

```text
plan: free -> starter
plan_expires_at: 2026-07-18
last_payment_source: lemonsqueezy
videos_limit: 3 -> 9999
videos_used: 0
```

Notes:

- `2026-07-18` is correct for starter 10-day duration.
- `videos_used=0` is correct for this activation path.

### 3. payment_events Insert

`payment_events` contains exactly one event for the paid order:

```text
provider=lemonsqueezy
order_id=8909715
plan=starter
count=1
```

### 4. Duplicate Webhook

Duplicate signed webhook was sent for:

```text
order_id=8909715
telegram_id=5052641158
plan=starter
```

Result:

```text
{"ok": true}
```

Verification:

```text
plan_expires_at remained 2026-07-18
payment_events count remained 1
```

So duplicate handling is idempotent and did not activate twice.

### 5. Unknown telegram_id

Signed webhook was sent with:

```text
telegram_id=9999999999
```

Result:

```text
{"ok": true}
```

Verification:

```text
bot_users row count for 9999999999 = 0
payment_events row count for unknown event = 0
```

This matches Design A: unknown-id anomalies are acknowledged but not persisted.

### 6. YooKassa Regression

Codex ran:

```powershell
py -m unittest test_bot_payments.py test_bot_tasks.py
```

Result:

```text
Ran 23 tests in 1.841s
OK
```

This includes YooKassa route/activation regression coverage.

## Cleanup

Gennady's test plan was returned to:

```text
plan=free
```

## Stage 3 Status

All six Stage 3 checks are green:

1. signed `order_created` -> `200`
2. `bot_users` activated correctly
3. `payment_events` inserted exactly once
4. duplicate webhook -> `200`, no second activation
5. unknown `telegram_id` -> `200`, no user/event creation
6. YooKassa regression tests OK

Stage 3 is ready for Cloud final review.

## Remaining Blocker Before Stage 4

`bot.py` source-of-truth has not been confirmed yet.

Needed manual check:

```text
Railway -> service transkrib-bot -> Settings -> Source
```

Record:

- source repository
- branch
- auto-deploy on/off

Until this is confirmed, bot deploy remains blocked.
