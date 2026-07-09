---
from: codex
to: cloud
date: 2026-07-09
topic: Bug 27 Stage 4 implementation ready for review - STOP before bot push
thread: bug-27-payments
status: stage-4-review-requested
session: 17
prev: 2026-07-08-cloud-to-codex-bug-27-stage-3-approved.md
---

# Bug #27 Stage 4 - implementation ready for review

## STOP status

No bot push and no bot deploy were performed.

- Source repository: `smolnirovgv-star/transkrib-bot`
- Production branch: `main`
- Auto deploy: ON
- Local branch: `bug-27-stage-4-lemon-button`
- Base: `92f0cc0a43955ee68f3b05c4db1bd695bd9bfdd2`
- Local commit: `ea4298cdf3d313090f29be07cc7461261f7f0e82`

## Exact scope

`main..stage4` contains exactly:

- `bot.py`
- `test_bot_helpers.py`

Implemented:

- removed static `LEMON_LINKS` usage from the bot;
- added `LEMONSQUEEZY_BOT_ALLOWED_IDS`;
- missing/empty ENV defaults to `ADMIN_ID=5052641158`;
- comma-separated IDs are supported;
- `*` opens Lemon checkout to all users;
- startup logs restricted user count or warns `OPEN TO ALL (*)`;
- denied users do not see the USD button;
- forged/stale USD callbacks are blocked before the API call;
- allowed USD callbacks create checkout through
  `/api/bot/payments/lemon/create`;
- the RUB/YooKassa implementation block was not changed.

No video-brief code and no statistics changes were transferred.

## TDD evidence

RED before implementation:

- 10 tests run;
- expected result: 1 failure and 7 errors;
- missing helpers: `_lemon_checkout_allowed`,
  `_log_lemon_whitelist`, `_create_lemon_checkout_url`;
- denied user still saw USD before the implementation;
- pre-existing YooKassa regression test already passed.

GREEN after implementation:

```text
Ran 10 tests in 0.049s
OK
```

Compile check:

```text
py -m py_compile bot.py billing.py claude_assistant.py
exit code 0
```

Diff hygiene:

```text
git diff --check
exit code 0

git diff --name-only origin/main...HEAD
bot.py
test_bot_helpers.py
```

## Pre-push gate

Before any push to `main`:

1. Cloud reviews the complete diff below.
2. Gennadiy gives explicit approval.
3. Confirm Railway `transkrib-bot` has
   `LEMONSQUEEZY_BOT_ALLOWED_IDS=5052641158`.
4. Only then push to `main`; Railway will auto-deploy immediately.

## Complete diff (`origin/main...ea4298c`)

```diff
diff --git a/bot.py b/bot.py
index 3358d3f..b2d9a34 100644
--- a/bot.py
+++ b/bot.py
@@ -1,119 +1,167 @@
 import os
 import io
 import json
 import asyncio
 import time
 import httpx
 from dotenv import load_dotenv
 load_dotenv()
-from billing import can_process, increment_usage, get_status_text, PLAN_PRICES, LEMON_LINKS, can_process_video, log_usage
+from billing import can_process, increment_usage, get_status_text, PLAN_PRICES, can_process_video, log_usage
 from claude_assistant import ask_claude
 from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup, BotCommand
 from telegram.error import BadRequest, Conflict
 from telegram.ext import Application, ApplicationBuilder, CommandHandler, MessageHandler, CallbackQueryHandler, ConversationHandler, filters, ContextTypes
 
 import logging
 import sys
 
 logging.basicConfig(
     level=logging.INFO,
     format="%(asctime)s [%(levelname)s] %(name)s - %(message)s",
     stream=sys.stdout,
     force=True,
 )
 logging.getLogger("httpx").setLevel(logging.WARNING)
 logging.getLogger("telegram").setLevel(logging.INFO)
 logging.getLogger("telegram.ext").setLevel(logging.INFO)
 logger = logging.getLogger("transkrib_bot")
 
 logger.info("=== bot.py module loaded ===")
 
 BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN", "")
 _daily_usage_store = []  # list of {date, input_tokens, output_tokens, cost_usd}
 API_URL = os.environ.get("TRANSKRIB_API_URL", "https://transkrib-api.onrender.com")
 BOT_NAME = os.getenv("BOT_NAME", "Transkrib")
 BOT_MODE = os.getenv("BOT_MODE", "phone")  # phone | youtube -- reserved for future use
 
 ADMIN_ID = 5052641158
 FREE_CHAT_LIMIT = 10  # messages per day for free users
 
+
+def _lemon_whitelist():
+    raw = os.environ.get("LEMONSQUEEZY_BOT_ALLOWED_IDS", "").strip()
+    if not raw:
+        return False, {ADMIN_ID}
+    if raw == "*":
+        return True, set()
+
+    allowed_ids = set()
+    for value in raw.split(","):
+        try:
+            allowed_ids.add(int(value.strip()))
+        except ValueError:
+            continue
+    return False, allowed_ids
+
+
+def _lemon_checkout_allowed(telegram_id: int) -> bool:
+    open_to_all, allowed_ids = _lemon_whitelist()
+    return open_to_all or telegram_id in allowed_ids
+
+
+def _log_lemon_whitelist() -> None:
+    open_to_all, allowed_ids = _lemon_whitelist()
+    if open_to_all:
+        logger.warning("[LEMON] whitelist: OPEN TO ALL (*)")
+        return
+    logger.info("[LEMON] whitelist: %d user(s)", len(allowed_ids))
+
+
+async def _create_lemon_checkout_url(telegram_id: int, plan: str) -> str:
+    async with httpx.AsyncClient(timeout=60.0) as client:
+        response = await client.post(
+            f"{API_URL}/api/bot/payments/lemon/create",
+            json={"telegram_id": telegram_id, "plan": plan},
+            timeout=60.0,
+        )
+        response.raise_for_status()
+
+    data = response.json()
+    if "error" in data:
+        raise RuntimeError(data["error"])
+    payment_url = data.get("payment_url")
+    if not payment_url:
+        raise RuntimeError("Payment API did not return payment_url")
+    return payment_url
+
+
 WAITING_CUT = 1
 WAITING_FORMAT = 2
 WAITING_LANG = 3
 
 CUT_LABELS = {'cut_5': '5 мин', 'cut_10': '10 мин', 'cut_15': '15 мин', 'cut_20': '20 мин', 'cut_30': '30 мин', 'cut_no': 'Без сокращения'}
 FMT_LABELS = {'fmt_text': 'Только транскрипция', 'fmt_cut': 'Транскрипция + нарезка', 'fmt_srt': 'SRT субтитры', 'fmt_md': 'Markdown (.md)', 'fmt_cut_srt': 'Нарезка + SRT'}
 LANG_LABELS = {'lang_auto': '🔄 Авто', 'lang_ru': '🇷🇺 Русский', 'lang_en': '🇬🇧 English'}
 
 MAX_VIDEO_SIZE_BYTES = 20 * 1024 * 1024  # 20 MB Telegram Bot API limit
 
 VIDEO_FALLBACK_MESSAGE = (
     "Сейчас бот принимает только видеофайлы напрямую с устройства. "
     "Пришлите ролик с телефона (до 20 МБ) — получите транскрипт, ключевые моменты и видео-нарезку."
 )
 
 LANG_CONFIRM = {
     'lang_ru': {'flag': '🇷🇺', 'confirmed': 'Русский выбран', 'prompt': 'Пришли видео-файл с телефона (до 20 МБ) — я сделаю транскрипцию и умную нарезку!'},
     'lang_en': {'flag': '🇬🇧', 'confirmed': 'English selected', 'prompt': 'Upload a video file from your phone (up to 20 MB) and I will transcribe it!'},
     'lang_hi': {'flag': '🇮🇳', 'confirmed': 'हिन्दी चुनी गई', 'prompt': 'अपने फ़ोन से वीडियो फ़ाइल अपलोड करें (20 MB तक) — मैं ट्रांस्क्रिप्शन और स्मार्ट कट बनाऊँगा!'},
     'lang_zh': {'flag': '🇨🇳', 'confirmed': '已选择中文', 'prompt': '请从手机上传视频文件（最大20 MB）— 我会为您生成转录和智能剪辑！'},
     'lang_ko': {'flag': '🇰🇷', 'confirmed': '한국어 선택됨', 'prompt': '휴대폰에서 동영상 파일을 업로드하세요 (최대 20 MB) — 트랜스크립션과 스마트 컷을 만들어드립니다!'},
     'lang_pt': {'flag': '🇧🇷', 'confirmed': 'Português selecionado', 'prompt': 'Faça upload de um vídeo do celular (até 20 MB) — vou gerar transcrição e corte inteligente!'},
 }
 
 
 async def _log_user_message(update, message_type: str, message_text: str = None,
                              file_id: str = None, file_size: int = None,
                              duration: int = None, mime_type: str = None):
     """Логирует сообщение пользователя в user_feedback таблицу. Не логирует ADMIN_ID."""
     user = update.effective_user
     if user.id == ADMIN_ID:
         return
     try:
         from claude_assistant import supabase
         raw = update.message.to_dict() if update.message else {}
         supabase.table("user_feedback").insert({
             "bot_name": BOT_NAME,
             "user_id": user.id,
             "username": user.username,
             "first_name": user.first_name,
             "last_name": user.last_name,
             "language_code": user.language_code,
             "message_type": message_type,
             "message_text": message_text,
             "file_id": file_id,
             "file_size": file_size,
             "duration": duration,
             "mime_type": mime_type,
             "raw_payload": raw,
         }).execute()
     except Exception as e:
         print(f"[user_feedback] log error: {e}")
 
 
 async def handle_media_or_other(update, context):
     """Логирует медиа/прочие сообщения + acknowledgment на языке пользователя."""
     msg = update.message
     if not msg:
         return
 
     file_id = None
     file_size = None
     duration = None
     mime_type = None
     text = None
     msg_type = 'other'
 
     if msg.voice:
         msg_type = 'voice'
         file_id = msg.voice.file_id
         file_size = msg.voice.file_size
         duration = msg.voice.duration
         mime_type = msg.voice.mime_type
     elif msg.audio:
         msg_type = 'audio'
         file_id = msg.audio.file_id
         file_size = msg.audio.file_size
         duration = msg.audio.duration
         mime_type = msg.audio.mime_type
     elif msg.photo:
@@ -1058,191 +1106,212 @@ async def process_video(chat_id, url, context):
                         pass
                     total = int(time.monotonic() - start_time)
                     mm, ss = divmod(total, 60)
                     try:
                         await timer_msg.edit_text(f"⚠️ <b>Остановлено на {mm:02d}:{ss:02d}</b>", parse_mode="HTML")
                     except Exception as e:
                         logger.warning("[timer] cancelled edit failed: %s", e)
                     return
                 elif status == "error":
                     await _update_progress(context, chat_id, msg_id, 'error')
                     total = int(time.monotonic() - start_time)
                     mm, ss = divmod(total, 60)
                     try:
                         await timer_msg.edit_text(f"⚠️ <b>Остановлено на {mm:02d}:{ss:02d}</b>", parse_mode="HTML")
                     except Exception as e:
                         logger.warning("[timer] error edit failed: %s", e)
                     error = data.get("error", "Неизвестная ошибка")
                     # Detect YouTube-specific errors
                     yt_keywords = ["youtube", "sign in", "bot", "cookie", "yt-dlp", "403"]
                     is_yt_error = any(kw in error.lower() for kw in yt_keywords)
                     if is_yt_error:
                         kb = InlineKeyboardMarkup([[
                             InlineKeyboardButton("🔄 Перезапустить чат", callback_data="retry_fresh")
                         ]])
                         await context.bot.send_message(
                             chat_id=chat_id,
                             text=(
                                 f"❌ Ошибка: {error[:300]}\n\n"
                                 "⚠️ Не удалось скачать видео. Возможные причины:\n• Видео удалено или закрыто\n• Временная блокировка\n\nПопробуй позже или отправь другое видео."
                             ),
                             reply_markup=kb
                         )
                     else:
                         await context.bot.send_message(
                             chat_id=chat_id, text=f"❌ Ошибка: {error}"
                         )
                     return
 
             await _update_progress(context, chat_id, msg_id, 'error')
             total = int(time.monotonic() - start_time)
             mm, ss = divmod(total, 60)
             try:
                 await timer_msg.edit_text(f"⚠️ <b>Остановлено на {mm:02d}:{ss:02d}</b>", parse_mode="HTML")
             except Exception as e:
                 logger.warning("[timer] timeout edit failed: %s", e)
             await context.bot.send_message(
                 chat_id=chat_id,
                 text=f"⏱ Превышено время ожидания ({max_attempts * 10 // 60} мин). Видео слишком длинное или произошла ошибка обработки. Попробуй ещё раз или свяжись с поддержкой.",
             )
 
     except Exception as e:
         await context.bot.send_message(chat_id=chat_id, text=f"❌ Ошибка: {str(e)[:200]}")
 
 
 async def cmd_plan(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "cmd_plan", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     tid = update.effective_user.id
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(tid)
     text += nl + nl + "📦 *Тарифы:*" + nl
     text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
     text += "🚀 Стандарт — 1500₽ / $15 — безлимит, 30 дней" + nl
     text += "👑 Годовой — 8900₽ / $100 — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
         InlineKeyboardButton("👑 Годовой", callback_data="buy_annual"),
     ]])
     await update.message.reply_text(text, parse_mode="Markdown", reply_markup=kb)
 
 
 async def handle_buy(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "handle_buy", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     query = update.callback_query
     await query.answer()
     plan = query.data.replace("buy_", "")
     p = PLAN_PRICES.get(plan)
     if not p:
         return
     nl = chr(10)
-    kb = InlineKeyboardMarkup([[
+    buttons = [
         InlineKeyboardButton(f"🇷🇺 {p['rub']}₽ — Оплатить (ЮКасса)", callback_data=f"currency_rub_{plan}"),
-        InlineKeyboardButton(f"🇺🇸 ${p['usd']} — Pay (LemonSqueezy)", callback_data=f"currency_usd_{plan}"),
-    ]])
+    ]
+    if _lemon_checkout_allowed(query.from_user.id):
+        buttons.append(
+            InlineKeyboardButton(f"🇺🇸 ${p['usd']} — Pay (LemonSqueezy)", callback_data=f"currency_usd_{plan}")
+        )
+    kb = InlineKeyboardMarkup([buttons])
     await query.edit_message_text(
         f"💳 *{p['name']}*" + nl + nl
         + f"🇷🇺 {p['rub']}₽ / 🇺🇸 ${p['usd']}" + nl + nl
         + "Выбери валюту оплаты:",
         parse_mode="Markdown",
         reply_markup=kb,
     )
 
 
 async def handle_currency(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "handle_currency", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     query = update.callback_query
     await query.answer()
     _, currency, plan = query.data.split("_", 2)
     p = PLAN_PRICES.get(plan)
     if not p:
         return
     nl = chr(10)
 
     if currency == "usd":
-        link = LEMON_LINKS.get(plan, "")
-        await query.edit_message_text(
-            f"💳 *{p['name']}* — ${p['usd']}/мес" + nl + nl
-            + f"[Pay with LemonSqueezy]({link})" + nl + nl
-            + "После оплаты напиши /plan для проверки.",
-            parse_mode="Markdown",
-        )
+        tid = query.from_user.id
+        if not _lemon_checkout_allowed(tid):
+            logger.warning("[LEMON] blocked checkout callback for telegram_id=%s", tid)
+            await query.edit_message_text("Оплата в USD временно недоступна.")
+            return
+
+        await query.edit_message_text("⏳ Создаю ссылку на оплату...")
+        try:
+            link = await _create_lemon_checkout_url(tid, plan)
+            await context.bot.send_message(
+                chat_id=tid,
+                text=(
+                    f"💳 *{p['name']}* — ${p['usd']}" + nl + nl
+                    + f"[Pay with LemonSqueezy]({link})" + nl + nl
+                    + "После оплаты тариф активируется автоматически."
+                ),
+                parse_mode="Markdown",
+            )
+        except Exception as e:
+            logger.error("Lemon payment API error: %s", e)
+            await context.bot.send_message(
+                chat_id=tid,
+                text=f"❌ Ошибка LemonSqueezy: {str(e)[:200]}",
+            )
         return
 
     # ₽ — create YooKassa payment via API
     tid = query.from_user.id
     await query.edit_message_text("⏳ Создаю ссылку на оплату...")
     import httpx as _httpx
     try:
         # Wake up Render (free tier sleeps)
         async with _httpx.AsyncClient(timeout=60.0) as client:
             for _attempt in range(5):
                 try:
                     _ping = await client.get(f"{API_URL}/api/health", timeout=15.0)
                     if _ping.status_code < 500:
                         break
                 except Exception:
                     pass
                 await asyncio.sleep(8)
             resp = await client.post(
                 f"{API_URL}/api/bot/payments/yookassa/create",
                 json={"telegram_id": tid, "plan": plan},
                 timeout=60.0,
             )
             resp.raise_for_status()
         data = resp.json()
         if "error" in data:
             await context.bot.send_message(chat_id=tid, text=f"❌ Ошибка: {data['error']}")
             return
         payment_url = data.get("payment_url")
         if not payment_url:
             logger.error("No payment_url in response: %s", data)
             await context.bot.send_message(
                 chat_id=tid,
                 text="Ошибка платежа: сервер не вернул ссылку. Попробуйте позже или напишите в поддержку."
             )
             return
         await context.bot.send_message(
             chat_id=tid,
             text=(
                 f"💳 *{p['name']}* — {p['rub']}₽" + nl + nl
                 + f"[🔗 Перейти к оплате]({payment_url})" + nl + nl
                 + "После оплаты тариф активируется автоматически."
             ),
             parse_mode="Markdown",
         )
     except _httpx.HTTPStatusError as e:
         logger.error("Payment API error %s: %s", e.response.status_code, e.response.text[:200])
         await context.bot.send_message(
             chat_id=tid,
             text=f"Ошибка платежа: сервер вернул {e.response.status_code}. Попробуйте позже."
         )
     except Exception as e:
         await context.bot.send_message(chat_id=tid, text=f"❌ Ошибка платежа: {str(e)[:200]}")
 
 
 async def handle_show_plan(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "handle_show_plan", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     query = update.callback_query
     await query.answer()
     tid = query.from_user.id
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(tid)
     text += nl + nl + "📦 *Тарифы:*" + nl
     text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
     text += "🚀 Стандарт — 1500₽ / $15 — безлимит, 30 дней" + nl
     text += "👑 Годовой — 8900₽ / $100 — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
         InlineKeyboardButton("👑 Годовой", callback_data="buy_annual"),
     ]])
     await query.edit_message_reply_markup(reply_markup=None)
     await context.bot.send_message(chat_id=tid, text=text, parse_mode="Markdown", reply_markup=kb)
 
 
 async def cmd_help(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "cmd_help", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     text = (
         f"🤖 *{BOT_NAME}* — что умеет бот:\n\n"
         "📹 *Загрузи видео с телефона* (до 20 МБ):\n"
         "Отправь видео-файл прямо в чат или сними через 📎 → Камера\n\n"
@@ -1357,158 +1426,160 @@ async def handle_chat(update, context):
 
 
 async def _daily_cost_summary(context):
     """Send daily Claude cost summary to admin at 20:00 MSK."""
     from datetime import datetime, timezone, timedelta
     msk_now = datetime.now(timezone.utc) + timedelta(hours=3)
     today = msk_now.strftime("%Y-%m-%d")
     total_inp = total_out = total_cost = 0.0
     for rec in list(_daily_usage_store):
         if rec.get("date") == today:
             total_inp += rec.get("input_tokens", 0)
             total_out += rec.get("output_tokens", 0)
             total_cost += rec.get("cost_usd", 0.0)
     warning = "\n⚠️ Высокие расходы!" if total_cost > 5 else ""
     await context.bot.send_message(
         chat_id=ADMIN_ID,
         text=(
             f"📊 Сводка за {today}:\n"
             f"Input: {int(total_inp):,} tok\n"
             f"Output: {int(total_out):,} tok\n"
             f"Стоимость: ${total_cost:.4f}{warning}"
         ),
     )
 
 
 async def cmd_activate(update, context):
     """Активировать инвайт-код."""
     if not context.args:
         await update.message.reply_text(
             "Использование: /activate TRANSKRIB-TEST-XXXX-YYYY\n"
             "Получи код у администратора."
         )
         return
     code = context.args[0].strip().upper()
     user_id = update.effective_user.id
     from billing import supabase as _supabase
     result = _supabase.table("invite_codes").select("*").eq("code", code).execute()
     if not result.data:
         await update.message.reply_text(f"❌ Код {code} не найден")
         return
     row = result.data[0]
     if row.get("revoked"):
         await update.message.reply_text(f"❌ Код {code} отозван")
         return
     if row.get("used_by"):
         if row["used_by"] == user_id:
             await update.message.reply_text("ℹ️ Ты уже активировал этот код")
         else:
             await update.message.reply_text(f"❌ Код {code} уже использован")
         return
     from datetime import datetime, timezone, timedelta
     now = datetime.now(timezone.utc)
     expires_at = now + timedelta(days=row["duration_days"])
     _supabase.table("invite_codes").update({
         "used_by": user_id, "used_at": now.isoformat(),
     }).eq("code", code).execute()
     _supabase.table("bot_users").upsert({
         "telegram_id": user_id, "plan": row["plan"],
         "plan_expires_at": expires_at.isoformat(),
         "videos_limit": 9999,
     }).execute()
     await update.message.reply_html(
         f"✅ <b>Тестовый доступ активирован!</b>\n\n"
         f"План: <b>{row['plan']}</b> (безлимит как Pro)\n"
         f"До: <b>{expires_at.strftime('%d.%m.%Y')}</b>\n"
         f"Срок: {row['duration_days']} дней\n\n"
         f"Пришли видео-файл с телефона (до 20 МБ)."
     )
 
 async def post_init(app):
     await app.bot.set_my_commands([
         BotCommand("start",    "🚀 Главная — выбор языка"),
         BotCommand("plan",     "💳 Мой тариф и подписка"),
         BotCommand("help",     "❓ Помощь и инструкция"),
         BotCommand("cancel",   "❌ Отменить обработку"),
         BotCommand("activate", "🎟 Активировать инвайт-код"),
     ])
 
 
 def main():
+    _log_lemon_whitelist()
+
     # Initialize static ffmpeg/ffprobe binaries (for phone-video duration detection)
     try:
         import static_ffmpeg
         static_ffmpeg.add_paths()
         logger.info("static-ffmpeg ready: ffprobe available")
     except Exception as _sfe:
         logger.warning("static-ffmpeg init failed (%s) — phone video duration check disabled", _sfe)
 
     app = ApplicationBuilder().token(BOT_TOKEN).post_init(post_init).build()
 
     async def _global_error_handler(update, context):
         logger.exception(
             "Unhandled exception in handler | update=%s | error=%r",
             getattr(update, "to_dict", lambda: update)() if update else None,
             context.error,
         )
 
     app.add_error_handler(_global_error_handler)
     logger.info("Global error handler registered")
 
     conv_handler = ConversationHandler(
         per_message=False,
         entry_points=[
             MessageHandler(filters.Regex(r"https?://"), handle_url_reject),
             MessageHandler(filters.VIDEO | filters.Document.VIDEO, handle_video_start),
         ],
         states={
             WAITING_CUT: [CallbackQueryHandler(handle_cut, pattern="^cut_")],
             WAITING_FORMAT: [CallbackQueryHandler(handle_format, pattern="^fmt_")],
             WAITING_LANG: [CallbackQueryHandler(handle_lang_choice, pattern="^lang_")],
         },
         fallbacks=[CommandHandler("cancel", cmd_cancel)],
     )
 
     media_filter = (filters.VOICE | filters.AUDIO | filters.PHOTO |
                     filters.Sticker.ALL | filters.VIDEO_NOTE |
                     (filters.Document.ALL & ~filters.Document.VIDEO)) & ~filters.COMMAND
     app.add_handler(MessageHandler(media_filter, handle_media_or_other))
     app.add_handler(conv_handler)
     app.add_handler(CommandHandler("start", start))
     app.add_handler(CommandHandler("help", cmd_help))
     app.add_handler(CommandHandler("plan", cmd_plan))
     app.add_handler(CommandHandler("cancel",   cmd_cancel))
     app.add_handler(CommandHandler("stats",    cmd_stats))
     app.add_handler(CommandHandler("activate", cmd_activate))
     app.add_handler(CallbackQueryHandler(handle_buy, pattern="^buy_"))
     app.add_handler(CallbackQueryHandler(handle_currency, pattern="^currency_"))
     app.add_handler(CallbackQueryHandler(handle_recut, pattern="^recut_"))
     app.add_handler(CallbackQueryHandler(handle_stop, pattern="^stop_"))
     app.add_handler(CallbackQueryHandler(handle_retry, pattern="^retry_fresh$"))
     app.add_handler(CallbackQueryHandler(handle_show_plan, pattern="^show_plan$"))
     app.add_handler(CallbackQueryHandler(handle_language, pattern="^lang_(?:ru|en|hi|zh|ko|pt)$"))
     app.add_handler(MessageHandler(
         filters.TEXT & ~filters.COMMAND & ~filters.Regex(r'https?://'),
         handle_chat
     ))
     job_queue = app.job_queue
     if job_queue:
         from datetime import time as _time
         job_queue.run_daily(_daily_cost_summary, time=_time(17, 0, 0))  # 17:00 UTC = 20:00 MSK
     print("Bot started!")
     logger.info("Starting polling loop...")
     import time as _time_module
     while True:
         try:
             app.run_polling(drop_pending_updates=True)
             break
         except Conflict:
             logger.warning("[BOT] Conflict detected, restarting in 5s...")
             _time_module.sleep(5)
             continue
         except Exception as e:
             logger.error(f"[BOT] Fatal error: {e}")
             break
 
 
 if __name__ == "__main__":
     main()
diff --git a/test_bot_helpers.py b/test_bot_helpers.py
new file mode 100644
index 0000000..48c5334
--- /dev/null
+++ b/test_bot_helpers.py
@@ -0,0 +1,225 @@
+import os
+import sys
+import types
+import unittest
+from unittest.mock import AsyncMock, patch
+
+
+fake_billing = types.ModuleType("billing")
+fake_billing.can_process = lambda *_args, **_kwargs: True
+fake_billing.increment_usage = lambda *_args, **_kwargs: None
+fake_billing.get_status_text = lambda *_args, **_kwargs: "free"
+fake_billing.can_process_video = lambda *_args, **_kwargs: (True, "")
+fake_billing.log_usage = lambda *_args, **_kwargs: None
+fake_billing.PLAN_PRICES = {
+    "starter": {
+        "rub": "450",
+        "usd": "5",
+        "days": 10,
+        "videos_limit": 9999,
+        "name": "Базовый",
+    },
+}
+fake_billing.LEMON_LINKS = {"starter": "https://old.invalid/checkout"}
+sys.modules["billing"] = fake_billing
+
+fake_claude_assistant = types.ModuleType("claude_assistant")
+fake_claude_assistant.ask_claude = AsyncMock(return_value="ok")
+sys.modules["claude_assistant"] = fake_claude_assistant
+
+import bot
+
+
+class FakeResponse:
+    def __init__(self, data, status_code=200):
+        self._data = data
+        self.status_code = status_code
+        self.text = str(data)
+
+    def raise_for_status(self):
+        return None
+
+    def json(self):
+        return self._data
+
+
+class FakeAsyncClient:
+    calls = []
+    response_data = {"payment_url": "https://checkout.example/test"}
+
+    def __init__(self, *args, **kwargs):
+        self.args = args
+        self.kwargs = kwargs
+
+    async def __aenter__(self):
+        return self
+
+    async def __aexit__(self, exc_type, exc, tb):
+        return None
+
+    async def get(self, url, **kwargs):
+        self.calls.append(("get", url, kwargs))
+        return FakeResponse({"ok": True})
+
+    async def post(self, url, **kwargs):
+        self.calls.append(("post", url, kwargs))
+        return FakeResponse(self.response_data)
+
+
+class FakeQuery:
+    def __init__(self, data, telegram_id):
+        self.data = data
+        self.from_user = types.SimpleNamespace(id=telegram_id)
+        self.message = types.SimpleNamespace(chat_id=telegram_id)
+        self.edit_calls = []
+
+    async def answer(self, **kwargs):
+        return None
+
+    async def edit_message_text(self, text, **kwargs):
+        self.edit_calls.append((text, kwargs))
+
+
+class FakeUpdate:
+    def __init__(self, data, telegram_id):
+        self.callback_query = FakeQuery(data, telegram_id)
+        self.effective_user = self.callback_query.from_user
+        self.effective_chat = types.SimpleNamespace(id=telegram_id)
+        self.message = None
+
+
+class FakeBot:
+    def __init__(self):
+        self.messages = []
+
+    async def send_message(self, **kwargs):
+        self.messages.append(kwargs)
+
+
+class FakeContext:
+    def __init__(self):
+        self.bot = FakeBot()
+
+
+class LemonWhitelistTests(unittest.TestCase):
+    def tearDown(self):
+        os.environ.pop("LEMONSQUEEZY_BOT_ALLOWED_IDS", None)
+
+    def test_missing_env_allows_only_admin(self):
+        self.assertTrue(bot._lemon_checkout_allowed(bot.ADMIN_ID))
+        self.assertFalse(bot._lemon_checkout_allowed(123456))
+
+    def test_comma_separated_ids_are_allowed(self):
+        os.environ["LEMONSQUEEZY_BOT_ALLOWED_IDS"] = "111, 222"
+
+        self.assertTrue(bot._lemon_checkout_allowed(111))
+        self.assertTrue(bot._lemon_checkout_allowed(222))
+        self.assertFalse(bot._lemon_checkout_allowed(333))
+
+    def test_star_opens_checkout_to_all(self):
+        os.environ["LEMONSQUEEZY_BOT_ALLOWED_IDS"] = "*"
+
+        self.assertTrue(bot._lemon_checkout_allowed(123456))
+
+    def test_startup_log_reports_restricted_count(self):
+        os.environ["LEMONSQUEEZY_BOT_ALLOWED_IDS"] = "111,222"
+
+        with patch.object(bot.logger, "info") as info:
+            bot._log_lemon_whitelist()
+
+        info.assert_called_once_with("[LEMON] whitelist: %d user(s)", 2)
+
+    def test_startup_log_warns_when_open_to_all(self):
+        os.environ["LEMONSQUEEZY_BOT_ALLOWED_IDS"] = "*"
+
+        with patch.object(bot.logger, "warning") as warning:
+            bot._log_lemon_whitelist()
+
+        warning.assert_called_once_with("[LEMON] whitelist: OPEN TO ALL (*)")
+
+
+class LemonButtonTests(unittest.IsolatedAsyncioTestCase):
+    def tearDown(self):
+        os.environ.pop("LEMONSQUEEZY_BOT_ALLOWED_IDS", None)
+
+    async def test_denied_user_sees_only_rub_button(self):
+        update = FakeUpdate("buy_starter", 123456)
+
+        await bot.handle_buy(update, FakeContext())
+
+        keyboard = update.callback_query.edit_calls[0][1]["reply_markup"]
+        callbacks = [
+            button.callback_data
+            for row in keyboard.inline_keyboard
+            for button in row
+        ]
+        self.assertEqual(callbacks, ["currency_rub_starter"])
+
+    async def test_allowed_user_sees_rub_and_usd_buttons(self):
+        update = FakeUpdate("buy_starter", bot.ADMIN_ID)
+
+        await bot.handle_buy(update, FakeContext())
+
+        keyboard = update.callback_query.edit_calls[0][1]["reply_markup"]
+        callbacks = [
+            button.callback_data
+            for row in keyboard.inline_keyboard
+            for button in row
+        ]
+        self.assertEqual(
+            callbacks,
+            ["currency_rub_starter", "currency_usd_starter"],
+        )
+
+    async def test_denied_forged_usd_callback_does_not_call_api(self):
+        update = FakeUpdate("currency_usd_starter", 123456)
+
+        with patch.object(
+            bot,
+            "_create_lemon_checkout_url",
+            new=AsyncMock(),
+        ) as create_checkout:
+            await bot.handle_currency(update, FakeContext())
+
+        create_checkout.assert_not_awaited()
+
+
+class PaymentApiTests(unittest.IsolatedAsyncioTestCase):
+    def setUp(self):
+        FakeAsyncClient.calls = []
+
+    def tearDown(self):
+        os.environ.pop("LEMONSQUEEZY_BOT_ALLOWED_IDS", None)
+
+    async def test_lemon_checkout_uses_bot_payment_api(self):
+        with patch.object(bot.httpx, "AsyncClient", FakeAsyncClient):
+            url = await bot._create_lemon_checkout_url(bot.ADMIN_ID, "starter")
+
+        self.assertEqual(url, "https://checkout.example/test")
+        method, endpoint, kwargs = FakeAsyncClient.calls[0]
+        self.assertEqual(method, "post")
+        self.assertTrue(endpoint.endswith("/api/bot/payments/lemon/create"))
+        self.assertEqual(
+            kwargs["json"],
+            {"telegram_id": bot.ADMIN_ID, "plan": "starter"},
+        )
+
+    async def test_rub_callback_still_uses_yookassa_api(self):
+        update = FakeUpdate("currency_rub_starter", bot.ADMIN_ID)
+        context = FakeContext()
+
+        with patch.object(bot.httpx, "AsyncClient", FakeAsyncClient):
+            await bot.handle_currency(update, context)
+
+        post_calls = [call for call in FakeAsyncClient.calls if call[0] == "post"]
+        self.assertEqual(len(post_calls), 1)
+        _, endpoint, kwargs = post_calls[0]
+        self.assertTrue(endpoint.endswith("/api/bot/payments/yookassa/create"))
+        self.assertEqual(
+            kwargs["json"],
+            {"telegram_id": bot.ADMIN_ID, "plan": "starter"},
+        )
+
+
+if __name__ == "__main__":
+    unittest.main()
```
