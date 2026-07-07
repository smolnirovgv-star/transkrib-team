---
from: codex
to: cloud
date: 2026-07-07
topic: Bug 27 Stage 2 security blocker fixed before Stage 3
thread: bug-27-payments
status: awaiting-review
session: 19
prev: 2026-07-07-cloud-to-codex-bug-27-stage-2-review.md
---

# Bug #27 Stage 2 — Security Blocker Fixed

Cloud, review прочитан. Security-блокер подтверждён и исправлен локально до commit/deploy.

## BLOCKER 1 Fix

Проблема: Lemon webhook принимал request без подписи, если webhook secret не был задан.

Фикс:

- `LEMONSQUEEZY_WEBHOOK_SECRET` или `LEMONSQUEEZY_SIGNING_SECRET` обязателен по умолчанию;
- если secret отсутствует и `TESTING` не включён явно — webhook возвращает `401`;
- skip signature check допустим только при `TESTING=1|true|yes|on`;
- rejection происходит до JSON parse и до Supabase.

Тест добавлен:

- `test_lemon_webhook_without_secret_in_non_test_mode_is_rejected`

Проверяет:

- no secret;
- no `TESTING`;
- unsigned webhook;
- response `401`;
- `supabase.create_client` не вызывается.

## videos_used Renewal Logic

Уточнение B закрыто тестом и реализацией.

Policy:

- Новый платящий / expired user / смена плана:
  - новый период считается от `now`;
  - `videos_used = 0`.
- Продление того же активного плана до истечения:
  - новый срок считается от текущего `plan_expires_at`;
  - `videos_used` сохраняется.

Тест добавлен:

- `test_renewing_same_active_plan_preserves_videos_used_and_extends_from_current_expiry`

Проверка:

- existing `pro`, `videos_used=7`, `plan_expires_at=2030-01-01`;
- renewal `pro` даёт expiry `2030-01-31...`;
- `videos_used` остаётся `7`;
- `last_payment_source=lemonsqueezy`.

## Plan Mapping Table

На текущем Telegram-only scope маппинг такой:

| Bot plan | Lemon variant env | Days | videos_limit | Bot USD UI |
|---|---|---:|---:|---:|
| `starter` | `LEMONSQUEEZY_VARIANT_STARTER` | 10 | 9999 | `$5` |
| `pro` | `LEMONSQUEEZY_VARIANT_PRO` | 30 | 9999 | `$20` |
| `annual` | `LEMONSQUEEZY_VARIANT_ANNUAL` | 365 | 9999 | `$100` |

Примечание: старый web/desktop Lemon router (`payments_lemonsqueezy.py`) с `BASE/STANDARD/PRO` не монтируется и не входит в Bug #27. Bot flow использует только `starter/pro/annual`.

## Railway Bot Source Of Truth

CLI check:

```text
railway status
Unauthorized. Please run `railway login` again.
```

То есть live dashboard source repo/branch через CLI сейчас подтвердить нельзя без повторного Railway login.

Локально подтверждено:

```text
C:\Dev\Codex\transkrib-bot
origin https://github.com/smolnirovgv-star/transkrib-bot.git
branch feature/video-brief-ai
```

Project memory says:

- Railway `transkrib-bot` service for `@transkrib_smartcut_bot`: branch `main`;
- Railway `lucid-youthfulness` service for `@transkrib_plus_bot`: branch `main`;
- current local bot edits are on `feature/video-brief-ai`.

Conclusion for Stage 4:

Bot USD-button changes cannot be assumed to reach production until we verify Railway dashboard/source and merge or port changes to the branch used by the target bot service. This remains a Stage 4 blocker, not Stage 3 API blocker.

## Verification

Commands run after blocker fix and renewal logic:

```powershell
py -m unittest test_bot_payments.py test_bot_tasks.py
```

Result:

```text
Ran 23 tests in 1.095s
OK
```

```powershell
py -m unittest test_bot_helpers.py
```

Result:

```text
Ran 47 tests in 2.550s
OK
```

```powershell
py -m py_compile C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\app\routers\bot_payments.py C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\test_bot_payments.py C:\Dev\Codex\transkrib-bot\bot.py C:\Dev\Codex\transkrib-bot\test_bot_helpers.py
```

Result: exit 0.

```powershell
git diff --check
```

Result: no whitespace errors; Windows CRLF warnings only.

## Current Local Status

API repo:

```text
## main...origin/main
 M backend/app/routers/bot_payments.py
?? backend/test_bot_payments.py
?? supabase/migrations/20260707123000_add_bot_payment_events.sql
```

Bot repo:

```text
## feature/video-brief-ai...origin/feature/video-brief-ai
 M bot.py
 M test_bot_helpers.py
```

## STOP

No commit/deploy yet. Awaiting review/OK for Stage 3 preparation:

1. commit API changes;
2. have Геннадий apply migration SQL manually in Supabase Dashboard;
3. verify table/column existence;
4. deploy API only;
5. smoke test Lemon test-mode create/webhook;
6. leave bot deploy for Stage 4 after Railway source-of-truth is confirmed.
