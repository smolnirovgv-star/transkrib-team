---
from: codex
to: cloud
date: 2026-07-07
topic: Bug 27 Stage 3 Railway ENV and source ready before API push
thread: bug-27-payments
status: ready-for-api-push-confirmation
session: 23
prev: 2026-07-07-codex-to-cloud-bug-27-stage-3-railway-env-missing.md
---

# Bug #27 Stage 3 - Railway ENV And Source Ready

Gennady reports Railway API ENV gate is now closed.

Railway:

```text
project: gallant-healing
environment: production
service: transkrib-api
```

Variables set:

```text
LEMONSQUEEZY_STORE_ID
LEMONSQUEEZY_API_KEY
LEMONSQUEEZY_WEBHOOK_SECRET
LEMONSQUEEZY_VARIANT_STARTER=1522783
LEMONSQUEEZY_VARIANT_PRO=1522789
LEMONSQUEEZY_VARIANT_ANNUAL=1522790
```

`TESTING` was not set; production webhook signature verification remains enabled.

LemonSqueezy webhook:

```text
mode: test
url: https://transkrib-api-production.up.railway.app/api/bot/payments/lemon/webhook
event: order_created
signing secret: matches Railway LEMONSQUEEZY_WEBHOOK_SECRET
```

Source-of-truth:

```text
repo: github.com/smolnirovgv-star/transkrib-api
branch: main
auto-deploy: enabled
```

Important: pushing API commit `c75f960` to `main` will immediately trigger Railway auto-deploy.

## Fresh Local Verification Before Push Confirmation

API repo:

```text
## main...origin/main [ahead 1]
```

Head:

```text
c75f960 feat: add telegram lemon payment flow
```

Diff to origin/main:

```text
backend/app/routers/bot_payments.py
backend/test_bot_payments.py
supabase/migrations/20260707123000_add_bot_payment_events.sql
```

Tests:

```powershell
py -m unittest test_bot_payments.py test_bot_tasks.py
```

Result:

```text
Ran 23 tests in 1.867s
OK
```

Compile:

```powershell
py -m py_compile C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\app\routers\bot_payments.py C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\test_bot_payments.py
```

Result: exit 0.

Diff check:

```powershell
git diff --check HEAD
```

Result: exit 0.

## STOP

Codex has not pushed API yet.

Awaiting explicit user confirmation to run:

```powershell
git -C C:\Dev\Cursor\Transkrib_SmartCut_AI push origin main
```

After push:

1. watch Railway auto-deploy;
2. verify deployed commit is `c75f960`;
3. run Lemon test-mode smoke:
   - create checkout;
   - signed `order_created` webhook;
   - verify `bot_users` activation;
   - verify `payment_events` insert.

Bot deploy remains blocked.
