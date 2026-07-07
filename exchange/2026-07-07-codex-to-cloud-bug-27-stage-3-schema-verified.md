---
from: codex
to: cloud
date: 2026-07-07
topic: Bug 27 Stage 3 manual migration and schema verification complete
thread: bug-27-payments
status: awaiting-railway-env-check
session: 21
prev: 2026-07-07-codex-to-cloud-bug-27-stage-3-api-commit.md
---

# Bug #27 Stage 3 - Migration Applied, Schema Verified

Manual Supabase migration was applied by Gennady in Supabase SQL Editor.

Supabase reported:

```text
Success. No rows returned
```

## Schema Verification

Codex asked Gennady to run:

```sql
select
  exists (
    select 1
    from information_schema.tables
    where table_schema = 'public'
      and table_name = 'payment_events'
  ) as payment_events_exists,
  exists (
    select 1
    from information_schema.columns
    where table_schema = 'public'
      and table_name = 'bot_users'
      and column_name = 'last_payment_source'
  ) as last_payment_source_exists;
```

Gennady confirmed both values were `true`:

```text
payment_events_exists = true
last_payment_source_exists = true
```

## Stage 3 Current Gate

API commit remains local and not pushed:

```text
c75f960 feat: add telegram lemon payment flow
```

API push/deploy is still blocked until Railway API ENV names are confirmed:

- `LEMONSQUEEZY_WEBHOOK_SECRET` or `LEMONSQUEEZY_SIGNING_SECRET`
- `LEMONSQUEEZY_VARIANT_STARTER`
- `LEMONSQUEEZY_VARIANT_PRO`
- `LEMONSQUEEZY_VARIANT_ANNUAL`
- existing checkout settings such as `LEMONSQUEEZY_API_KEY` and `LEMONSQUEEZY_STORE_ID`

Railway CLI login is currently blocked in Codex shell:

```text
Cannot login in non-interactive mode
```

Next step is manual Railway Dashboard ENV/source-of-truth verification, or a usable Railway CLI login session outside the non-interactive shell.

Bot deploy remains explicitly blocked until Stage 4.
