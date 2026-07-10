# Brief/SmartCut Bot - Stage 2 rebuilt diff for Cloud review

Date: 2026-07-10
From: Codex
To: Cloud
Topic: rebuilt Stage 2 after Plus shutdown and brief USD decision

## Status

Implemented locally only. No product repo commits, no product pushes, no deploys.

Previous Stage 2 diff is obsolete because `transkrib-bot/main` was confirmed to serve both SmartCut and Plus. Since Геннадий disabled `@transkrib_plus_bot`, SmartCut branding is now safe in `main`.

## Scope

### Brief bot

Repo: `C:\Dev\Codex\transkrib-bot`
Branch: `feature/video-brief-ai`
Files changed: `bot.py`, `billing.py`, `test_bot_helpers.py`

Changes:

- `/start` approved copy, including big-upload button mention.
- 4h limit -> 3h: `LONG_VIDEO_HARD_LIMIT_MIN = 180` and user-facing error text.
- `PLAN_PRICES` pro price fixed to `1500/$15`.
- annual plan name changed from `Про` to `Годовой`.
- `/plan`, `show_plan`, `/help` use current `PLAN_PRICES` values.
- `/help` contacts include `transkrib.su`, `info@transkrib.su`, `schwed.2000@mail.ru`.
- `LEMON_USD_ENABLED` added, default false.
- USD/Lemon button hidden by default.
- Direct `currency_usd_*` callback is blocked when USD is disabled, so old static `LEMON_LINKS` cannot be reached accidentally.

### SmartCut bot

Repo: `C:\Dev\Cursor\transkrib-bot`
Branch: `main`
Files changed: `bot.py`, `test_smartcut_texts.py`

Changes:

- `/start` approved SmartCut copy with em dashes and no brief/test text.
- `VIDEO_FALLBACK_MESSAGE` removes text `@transkrib_brief_test_bot`.
- `BRIEF_BOT_URL` env added with default `https://t.me/transkrib_brief_test_bot`.
- fallback reply now uses inline button `Открыть бота для ссылок` with URL from env.
- `/help` contacts include `transkrib.su`, `info@transkrib.su`, `schwed.2000@mail.ru`.

## Explicitly not done

- No language auto-detection.
- No brief rename.
- No one-off minute payments.
- No cost logging.
- No Bug #27 Lemon checkout transfer to brief; instead USD is hidden.
- No Railway deploy.
- No product repo commits/pushes.

## Verification

```text
C:\Users\Admin\OneDrive\Desktop\Cursor\Transkrib_Bot\.venv\Scripts\python.exe -m unittest test_bot_helpers.py
Ran 52 tests in 2.414s
OK
```

```text
C:\Users\Admin\OneDrive\Desktop\Cursor\Transkrib_Bot\.venv\Scripts\python.exe -m unittest test_smartcut_texts.py
Ran 3 tests in 0.041s
OK
```

Syntax check without writing `__pycache__`:

```text
syntax ok
syntax ok
```

## STOP

Awaiting Cloud review and Геннадий explicit OK before product commits/push/deploy.

Railway CLI remains logged out in this environment; brief deploy is still blocked until Геннадий performs Railway login/service selection.

## Diff - brief bot (`C:\Dev\Codex\transkrib-bot`)

```diff
diff --git a/billing.py b/billing.py
index 7aec18b..5c0a03f 100644
--- a/billing.py
+++ b/billing.py
@@ -19,8 +19,8 @@ PLANS = {
 
 PLAN_PRICES = {
     "starter": {"rub": "450",  "usd": "5",  "days": 10,   "videos_limit": 9999, "name": "⭐ Базовый"},
-    "pro":     {"rub": "1700", "usd": "20", "days": 30,  "videos_limit": 9999, "name": "🚀 Стандарт"},
-    "annual":  {"rub": "8900", "usd": "100","days": 365, "videos_limit": 9999, "name": "👑 Про"},
+    "pro":     {"rub": "1500", "usd": "15", "days": 30,  "videos_limit": 9999, "name": "🚀 Стандарт"},
+    "annual":  {"rub": "8900", "usd": "100","days": 365, "videos_limit": 9999, "name": "👑 Годовой"},
 }
 
 LEMON_LINKS = {
diff --git a/bot.py b/bot.py
index 9c4f8b1..487457b 100644
--- a/bot.py
+++ b/bot.py
@@ -57,6 +57,10 @@ def _video_billing_relaxed() -> bool:
         return True
     return "test" in BOT_NAME.lower()
 
+
+def _lemon_usd_enabled() -> bool:
+    return os.getenv("LEMON_USD_ENABLED", "").strip().lower() in {"1", "true", "yes", "on"}
+
 WAITING_CUT = 1
 WAITING_VIDEO_BRIEF_TEXT = 10
 WAITING_VIDEO_BRIEF_CONFIRM = 11
@@ -686,6 +690,8 @@ LANG_LABELS = {
 }
 
 MAX_VIDEO_SIZE_BYTES = 20 * 1024 * 1024  # 20 MB Telegram Bot API limit
+LONG_VIDEO_HARD_LIMIT_MIN = 180
+LONG_VIDEO_WARN_MIN = 120
 
 VIDEO_FALLBACK_MESSAGE = (
     "🎬 <b>Это видео больше лимита прямой отправки Telegram-боту.</b>\n\n"
@@ -1047,16 +1053,16 @@ async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
     ]]
     await _safe_reply_text(
         update.message,
-        f"👋 Привет! Я {BOT_NAME}!\n\n"
-        "🧪 <b>Тестовый бот</b>\n"
-        "Сейчас идёт закрытая проверка для 2-3 пользователей. Возможны задержки и ошибки.\n"
-        "Не отправляйте конфиденциальные видео.\n"
-        "Пожалуйста, не передавайте ссылку на бота третьим лицам.\n"
-        "Если бот завис или выдал ошибку, пришлите время, действие и скрин/текст сообщения.\n\n"
-        "📹 Большие файлы загружайте через кнопку «Загрузить большое видео», "
-        "но вы можете прислать видео-файл с телефона прямо в чат (до 20 МБ).\n\n"
-        "💡 <i>Совет: можно снять прямо в чате — жми 📎 → Камера</i>\n\n"
-        "🌐 transkrib.su · ✉️ info@transkrib.su\n\n"
+        "Привет! Я Transkrib Brief.\n\n"
+        "Превращаю видео в текст: с пунктуацией, заглавными буквами\n"
+        "и разбивкой на смысловые блоки.\n\n"
+        "Что умею:\n"
+        "- Ссылки: YouTube, VK, прямые файлы\n"
+        "- Длинные видео: до 3 часов\n"
+        "- Форматы: текст, ключевые моменты, субтитры, нарезка\n\n"
+        "Большие файлы — через кнопку «Загрузить большое видео».\n\n"
+        "Первые 3 видео — бесплатно. Карта не нужна.\n\n"
+        "Просто пришлите ссылку или файл.\n\n"
         "🌍 Выберите язык / Choose your language:",
         reply_markup=InlineKeyboardMarkup(keyboard),
         parse_mode="HTML",
@@ -1817,9 +1823,6 @@ async def process_video(chat_id, url, context):
     else:
         max_wait_seconds = 1800
 
-    LONG_VIDEO_HARD_LIMIT_MIN = 240
-    LONG_VIDEO_WARN_MIN = 180
-
     if duration_minutes > LONG_VIDEO_HARD_LIMIT_MIN:
         hours = duration_minutes // 60
         mins = duration_minutes % 60
@@ -1827,8 +1830,8 @@ async def process_video(chat_id, url, context):
             chat_id=chat_id,
             text=(
                 f"🎬 Видео: <b>{hours} ч {mins:02d} мин</b>\n\n"
-                "⛔ <b>Превышен лимит 4 часа</b>\n\n"
-                "Сейчас бот обрабатывает видео до 4 часов за один раз. "
+                "⛔ <b>Превышен лимит 3 часа</b>\n\n"
+                "Сейчас бот обрабатывает видео до 3 часов за один раз. "
                 "Для более длинных видео разбейте материал на части и пришлите их по очереди."
             ),
             parse_mode="HTML",
@@ -2206,13 +2209,13 @@ async def cmd_plan(update, context):
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(uid)
     text += nl + nl + "📦 *Тарифы:*" + nl
-    text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
-    text += "🚀 Стандарт — 1700₽ / $20 — безлимит, 30 дней" + nl
-    text += "👑 Про — 8900₽ / $100 — безлимит, 1 год"
+    text += f"⭐ Базовый — {PLAN_PRICES['starter']['rub']}₽ / ${PLAN_PRICES['starter']['usd']} — безлимит, 10 дней" + nl
+    text += f"🚀 Стандарт — {PLAN_PRICES['pro']['rub']}₽ / ${PLAN_PRICES['pro']['usd']} — безлимит, 30 дней" + nl
+    text += f"{PLAN_PRICES['annual']['name']} — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']} — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
-        InlineKeyboardButton("👑 Про", callback_data="buy_annual"),
+        InlineKeyboardButton("👑 Годовой", callback_data="buy_annual"),
     ]])
     await update.message.reply_text(text, parse_mode="Markdown", reply_markup=kb)
 
@@ -2226,15 +2229,15 @@ async def handle_buy(update, context):
     if not p:
         return
     nl = chr(10)
-    kb = InlineKeyboardMarkup([[
-        InlineKeyboardButton(f"🇷🇺 {p['rub']}₽ — оплатить (ЮKassa)", callback_data=f"currency_rub_{plan}"),
-        InlineKeyboardButton(f"🇺🇸 ${p['usd']} — Pay (LemonSqueezy)", callback_data=f"currency_usd_{plan}"),
-    ]])
+    buttons = [InlineKeyboardButton(f"🇷🇺 {p['rub']}₽ — оплатить (ЮKassa)", callback_data=f"currency_rub_{plan}")]
+    if _lemon_usd_enabled():
+        buttons.append(InlineKeyboardButton(f"🇺🇸 ${p['usd']} — Pay (LemonSqueezy)", callback_data=f"currency_usd_{plan}"))
+    kb = InlineKeyboardMarkup([buttons])
     await _safe_edit_message_text(
         query,
         f"💳 *{p['name']}*" + nl + nl
-        + f"🇷🇺 {p['rub']}₽ / 🇺🇸 ${p['usd']}" + nl + nl
-        + "Выбери валюту оплаты:",
+        + (f"🇷🇺 {p['rub']}₽ / 🇺🇸 ${p['usd']}" if _lemon_usd_enabled() else f"🇷🇺 {p['rub']}₽") + nl + nl
+        + "Выбери способ оплаты:",
         parse_mode="Markdown",
         reply_markup=kb,
     )
@@ -2251,6 +2254,12 @@ async def handle_currency(update, context):
     nl = chr(10)
 
     if currency == "usd":
+        if not _lemon_usd_enabled():
+            await _safe_edit_message_text(
+                query,
+                "Оплата в долларах временно недоступна, используйте рубли.",
+            )
+            return
         link = LEMON_LINKS.get(plan, "")
         await _safe_edit_message_text(
             query,
@@ -2318,13 +2327,13 @@ async def handle_show_plan(update, context):
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(tid)
     text += nl + nl + "📦 *Тарифы:*" + nl
-    text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
-    text += "🚀 Стандарт — 1700₽ / $20 — безлимит, 30 дней" + nl
-    text += "👑 Про — 8900₽ / $100 — безлимит, 1 год"
+    text += f"⭐ Базовый — {PLAN_PRICES['starter']['rub']}₽ / ${PLAN_PRICES['starter']['usd']} — безлимит, 10 дней" + nl
+    text += f"🚀 Стандарт — {PLAN_PRICES['pro']['rub']}₽ / ${PLAN_PRICES['pro']['usd']} — безлимит, 30 дней" + nl
+    text += f"{PLAN_PRICES['annual']['name']} — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']} — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
-        InlineKeyboardButton("👑 Про", callback_data="buy_annual"),
+        InlineKeyboardButton("👑 Годовой", callback_data="buy_annual"),
     ]])
     await query.edit_message_reply_markup(reply_markup=None)
     await context.bot.send_message(chat_id=tid, text=text, parse_mode="Markdown", reply_markup=kb)
@@ -2334,20 +2343,24 @@ async def cmd_help(update, context):
     logger.info("handler=%s user=%s chat=%s data=%r", "cmd_help", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     text = (
         f"🤖 *{BOT_NAME}* — что умеет бот:\n\n"
+        "🔗 *Ссылки и длинные видео:*\n"
+        "YouTube, VK и прямые файлы до 3 часов.\n\n"
         "🎬 *Загрузка видео с телефона* (до 20 МБ):\n"
         "Отправь видео-файл прямо в чат или сними через 📎 → Камера.\n\n"
         "📤 *Большие видео*:\n"
         "Нажми «Загрузить большое видео», выбери файл в браузере и вернись в Telegram после загрузки.\n\n"
+        "📄 *Качество текста:*\n"
+        "Текст с пунктуацией, заглавными буквами и смысловыми блоками.\n\n"
         "⚙️ *Режимы обработки:*\n"
         "- автоматическая нарезка;\n"
         "- режим по эпизодам;\n"
         "- только транскрипция;\n"
         "- SRT субтитры и монтажная карта.\n\n"
         "💳 *Тарифы:*\n"
-        "- пробный — бесплатно;\n"
-        "- Базовый — 450₽ / $5, 10 дней;\n"
-        "- Стандарт — 1700₽ / $20, 30 дней;\n"
-        "- Про — 8900₽ / $100, 365 дней.\n\n"
+        "- пробный — первые 3 видео бесплатно;\n"
+        f"- Базовый — {PLAN_PRICES['starter']['rub']}₽ / ${PLAN_PRICES['starter']['usd']}, 10 дней;\n"
+        f"- Стандарт — {PLAN_PRICES['pro']['rub']}₽ / ${PLAN_PRICES['pro']['usd']}, 30 дней;\n"
+        f"- Годовой — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']}, 365 дней.\n\n"
         "📌 *Команды:*\n"
         "/start — главная\n"
         "/plan — тариф и подписка\n"
@@ -2355,7 +2368,7 @@ async def cmd_help(update, context):
         "/cancel — отменить обработку\n\n"
         "📧 *Контакты:*\n"
         "info@transkrib.su\n"
-        "support@transkrib.su\n"
+        "schwed.2000@mail.ru\n"
         "transkrib.su"
     )
     await update.message.reply_text(text, parse_mode="Markdown")
diff --git a/test_bot_helpers.py b/test_bot_helpers.py
index b5c4ef9..52578bf 100644
--- a/test_bot_helpers.py
+++ b/test_bot_helpers.py
@@ -81,6 +81,7 @@ class FakeCallbackQuery:
         self.data = ""
         self.message = FakeMessage()
         self.message.chat_id = 456
+        self.from_user = type("User", (), {"id": 123})()
 
     async def answer(self, **kwargs):
         self.answer_calls.append(kwargs)
@@ -232,21 +233,27 @@ class SafeCallbackButtonTests(unittest.IsolatedAsyncioTestCase):
 
 
 class StartMessageTests(unittest.IsolatedAsyncioTestCase):
-    async def test_start_message_is_readable_russian(self):
+    async def test_start_message_explains_brief_value_and_free_trial(self):
         update = FakeMessageUpdate("/start")
         context = FakeContext()
 
         await bot.start(update, context)
 
         text = update.message.calls[0][0]
-        self.assertIn("Тестовый бот", text)
-        self.assertIn("Не отправляйте конфиденциальные видео", text)
-        self.assertIn("не передавайте ссылку", text)
-        self.assertIn("пришлите время", text)
-        self.assertIn("Привет", text)
+        self.assertIn("Привет! Я Transkrib Brief.", text)
+        self.assertIn("пунктуацией", text)
+        self.assertIn("заглавными буквами", text)
+        self.assertIn("YouTube, VK, прямые файлы", text)
+        self.assertIn("до 3 часов", text)
+        self.assertIn("Большие файлы", text)
         self.assertIn("Загрузить большое видео", text)
-        self.assertIn("но вы можете прислать видео-файл с телефона", text)
+        self.assertIn("Первые 3 видео — бесплатно", text)
+        self.assertIn("Карта не нужна", text)
+        self.assertIn("Просто пришлите ссылку или файл", text)
         self.assertIn("Выберите язык", text)
+        self.assertNotIn("Тестовый бот", text)
+        self.assertNotIn("Не отправляйте конфиденциальные видео", text)
+        self.assertNotIn("до 4 часов", text)
         self.assertNotIn("Пришли видео-файл с телефона (до 20 МБ)", text)
         self.assertNotIn("РЎ", text)
         self.assertNotIn("СЂ", text)
@@ -422,6 +429,81 @@ class BotCommandMenuTests(unittest.IsolatedAsyncioTestCase):
         self.assertFalse(any("Р" in description or "СЂ" in description for description in descriptions))
 
 
+class BillingCopyTests(unittest.IsolatedAsyncioTestCase):
+    def test_plan_prices_match_current_source_of_truth(self):
+        self.assertEqual(bot.PLAN_PRICES["starter"]["rub"], "450")
+        self.assertEqual(bot.PLAN_PRICES["starter"]["usd"], "5")
+        self.assertEqual(bot.PLAN_PRICES["pro"]["rub"], "1500")
+        self.assertEqual(bot.PLAN_PRICES["pro"]["usd"], "15")
+        self.assertEqual(bot.PLAN_PRICES["annual"]["rub"], "8900")
+        self.assertEqual(bot.PLAN_PRICES["annual"]["usd"], "100")
+        self.assertEqual(bot.PLAN_PRICES["annual"]["name"], "👑 Годовой")
+
+    async def test_plan_command_uses_current_pro_price(self):
+        update = FakeMessageUpdate("/plan")
+        context = FakeContext()
+
+        with patch.object(bot, "get_status_text", return_value="🆓 Пробный\nВидео использовано: 0/3"):
+            await bot.cmd_plan(update, context)
+
+        text = update.message.calls[0][0]
+        self.assertIn("🚀 Стандарт — 1500₽ / $15 — безлимит, 30 дней", text)
+        self.assertIn("👑 Годовой — 8900₽ / $100 — безлимит, 1 год", text)
+        self.assertNotIn("1700", text)
+        self.assertNotIn("$20", text)
+
+    async def test_help_text_uses_current_pro_price_and_three_hour_limit(self):
+        update = FakeMessageUpdate("/help")
+        context = FakeContext()
+
+        await bot.cmd_help(update, context)
+
+        text = update.message.calls[0][0]
+        self.assertIn("до 3 часов", text)
+        self.assertIn("- Стандарт — 1500₽ / $15, 30 дней;", text)
+        self.assertIn("- Годовой — 8900₽ / $100, 365 дней.", text)
+        self.assertIn("transkrib.su", text)
+        self.assertIn("info@transkrib.su", text)
+        self.assertIn("schwed.2000@mail.ru", text)
+        self.assertNotIn("1700", text)
+        self.assertNotIn("$20", text)
+        self.assertNotIn("4 часа", text)
+
+
+class BriefPaymentButtonsTests(unittest.IsolatedAsyncioTestCase):
+    async def test_usd_button_hidden_by_default(self):
+        update = FakeCallbackUpdate("buy_starter")
+        context = FakeContext()
+
+        with patch.dict(bot.os.environ, {}, clear=True):
+            await bot.handle_buy(update, context)
+
+        keyboard = update.callback_query.edit_calls[0][1]["reply_markup"].inline_keyboard
+        buttons = [button for row in keyboard for button in row]
+
+        self.assertEqual(len(buttons), 1)
+        self.assertEqual(buttons[0].callback_data, "currency_rub_starter")
+        self.assertIn("ЮKassa", buttons[0].text)
+        self.assertFalse(any("LemonSqueezy" in button.text for button in buttons))
+
+    async def test_usd_callback_blocked_when_disabled(self):
+        update = FakeCallbackUpdate("currency_usd_starter")
+        context = FakeContext()
+
+        with patch.dict(bot.os.environ, {}, clear=True):
+            await bot.handle_currency(update, context)
+
+        text = update.callback_query.edit_calls[0][0]
+        self.assertIn("Оплата в долларах временно недоступна", text)
+        self.assertIn("используйте рубли", text)
+        self.assertNotIn("Pay with LemonSqueezy", text)
+
+
+class LongVideoLimitTests(unittest.TestCase):
+    def test_long_video_limit_is_three_hours(self):
+        self.assertEqual(bot.LONG_VIDEO_HARD_LIMIT_MIN, 180)
+
+
 class ProcessingStartTextTests(unittest.TestCase):
     def test_processing_start_text_shows_clock_timer(self):
         text = bot._processing_start_text()
```

## Diff - smartcut bot (`C:\Dev\Cursor\transkrib-bot`)

```diff
diff --git a/bot.py b/bot.py
index 3358d3f..b7a041a 100644
--- a/bot.py
+++ b/bot.py
@@ -46,12 +46,22 @@ FMT_LABELS = {'fmt_text': 'Только транскрипция', 'fmt_cut': '
 LANG_LABELS = {'lang_auto': '🔄 Авто', 'lang_ru': '🇷🇺 Русский', 'lang_en': '🇬🇧 English'}
 
 MAX_VIDEO_SIZE_BYTES = 20 * 1024 * 1024  # 20 MB Telegram Bot API limit
+BRIEF_BOT_URL = os.getenv("BRIEF_BOT_URL", "https://t.me/transkrib_brief_test_bot")
 
 VIDEO_FALLBACK_MESSAGE = (
-    "Сейчас бот принимает только видеофайлы напрямую с устройства. "
-    "Пришлите ролик с телефона (до 20 МБ) — получите транскрипт, ключевые моменты и видео-нарезку."
+    "Я работаю только с файлами с устройства — ссылки не принимаю.\n\n"
+    "Но у нас есть бот, который умеет ссылки: YouTube, VK, "
+    "длинные видео до 3 часов. Первые 3 видео тоже бесплатно.\n\n"
+    "А если у вас короткий ролик на телефоне (до 20 МБ) — "
+    "пришлите файл сюда, обработаю за минуту."
 )
 
+
+def _fallback_keyboard() -> InlineKeyboardMarkup:
+    return InlineKeyboardMarkup([[
+        InlineKeyboardButton("Открыть бота для ссылок", url=os.getenv("BRIEF_BOT_URL", BRIEF_BOT_URL))
+    ]])
+
 LANG_CONFIRM = {
     'lang_ru': {'flag': '🇷🇺', 'confirmed': 'Русский выбран', 'prompt': 'Пришли видео-файл с телефона (до 20 МБ) — я сделаю транскрипцию и умную нарезку!'},
     'lang_en': {'flag': '🇬🇧', 'confirmed': 'English selected', 'prompt': 'Upload a video file from your phone (up to 20 MB) and I will transcribe it!'},
@@ -185,10 +195,14 @@ async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
         InlineKeyboardButton("💳 Мой тариф", callback_data="show_plan"),
     ]]
     await update.message.reply_text(
-        f"👋 Привет! Я {BOT_NAME}!\n\n"
-        "📹 Пришли видео-файл с телефона (до 20 МБ) — я сделаю транскрипцию и умную нарезку!\n\n"
-        "💡 <i>Совет: можно снять прямо в чате — жми 📎 → Камера</i>\n\n"
-        "🌐 transkrib.su · ✉️ info@transkrib.su\n\n"
+        "Привет! Я Transkrib SmartCut.\n\n"
+        "Работаю с видео прямо с вашего телефона: снял, отправил,\n"
+        "получил текст и нарезку.\n\n"
+        "- Только файлы с устройства, до 20 МБ\n"
+        "- Быстро: короткие ролики за минуту\n"
+        "- Текст с пунктуацией + умная нарезка\n\n"
+        "Первые 3 видео — бесплатно.\n\n"
+        "Совет: можно снять прямо в чате — жми скрепку, Камера.\n\n"
         "🌍 Choose your language:",
         reply_markup=InlineKeyboardMarkup(keyboard),
         parse_mode="HTML"
@@ -199,7 +213,7 @@ async def handle_url_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
     logger.info("handler=%s user=%s chat=%s data=%r", "handle_url_start", update.effective_user.id if update and update.effective_user else None, update.effective_chat.id if update and update.effective_chat else None, (update.message.text if update and update.message else None) or (update.callback_query.data if update and update.callback_query else None))
     url = update.message.text.strip()
     if not url.startswith('http'):
-        await update.message.reply_text(VIDEO_FALLBACK_MESSAGE)
+        await update.message.reply_text(VIDEO_FALLBACK_MESSAGE, reply_markup=_fallback_keyboard())
         return ConversationHandler.END
     context.user_data['url'] = url
     keyboard = [[
@@ -220,7 +234,7 @@ async def handle_url_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
 
 async def handle_url_reject(update: Update, context: ContextTypes.DEFAULT_TYPE):
     """Reject URL входа — phone-video бот не принимает ссылки."""
-    await update.message.reply_text(VIDEO_FALLBACK_MESSAGE)
+    await update.message.reply_text(VIDEO_FALLBACK_MESSAGE, reply_markup=_fallback_keyboard())
 
 
 async def handle_video_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
@@ -1265,7 +1279,7 @@ async def cmd_help(update, context):
         "/cancel — отменить обработку\n\n"
         "📧 *Контакты:*\n"
         "✉️ info@transkrib.su\n"
-        "🛟 support@transkrib.su\n"
+        "✉️ schwed.2000@mail.ru\n"
         "🌐 transkrib.su"
     )
     await update.message.reply_text(text, parse_mode="Markdown")
diff --git a/test_smartcut_texts.py b/test_smartcut_texts.py
new file mode 100644
index 0000000..633dc70
--- /dev/null
+++ b/test_smartcut_texts.py
@@ -0,0 +1,69 @@
+import unittest
+import os
+from unittest.mock import patch
+
+os.environ.setdefault("SUPABASE_URL", "https://example.supabase.co")
+os.environ.setdefault("SUPABASE_KEY", "test-key")
+os.environ.setdefault("TELEGRAM_BOT_TOKEN", "123:test")
+
+import bot
+
+
+class FakeMessage:
+    def __init__(self):
+        self.calls = []
+        self.text = "/start"
+
+    async def reply_text(self, text, **kwargs):
+        self.calls.append((text, kwargs))
+
+
+class FakeUpdate:
+    def __init__(self):
+        self.message = FakeMessage()
+        self.effective_user = type("User", (), {"id": 123})()
+        self.effective_chat = type("Chat", (), {"id": 456})()
+
+
+class SmartCutCopyTests(unittest.IsolatedAsyncioTestCase):
+    def test_url_fallback_redirects_to_brief_bot(self):
+        with patch.dict(bot.os.environ, {"BRIEF_BOT_URL": "https://t.me/example_brief"}):
+            keyboard = bot._fallback_keyboard()
+
+        text = bot.VIDEO_FALLBACK_MESSAGE
+
+        self.assertIn("Я работаю только с файлами с устройства", text)
+        self.assertIn("YouTube, VK, длинные видео до 3 часов", text)
+        self.assertIn("Первые 3 видео тоже бесплатно", text)
+        self.assertNotIn("@transkrib_brief_test_bot", text)
+        self.assertEqual(keyboard.inline_keyboard[0][0].text, "Открыть бота для ссылок")
+        self.assertEqual(keyboard.inline_keyboard[0][0].url, "https://t.me/example_brief")
+        self.assertNotIn("Сейчас бот принимает только видеофайлы", text)
+
+    async def test_start_text_explains_smartcut_and_brief_difference(self):
+        update = FakeUpdate()
+
+        await bot.start(update, object())
+
+        text = update.message.calls[0][0]
+        self.assertIn("Привет! Я Transkrib SmartCut.", text)
+        self.assertIn("Только файлы с устройства, до 20 МБ", text)
+        self.assertIn("Текст с пунктуацией + умная нарезка", text)
+        self.assertIn("Первые 3 видео — бесплатно", text)
+        self.assertNotIn("@transkrib_brief_test_bot", text)
+        self.assertNotIn("---", text)
+        self.assertNotIn("Пришли видео-файл с телефона", text)
+
+    async def test_help_contains_current_contacts(self):
+        update = FakeUpdate()
+
+        await bot.cmd_help(update, object())
+
+        text = update.message.calls[0][0]
+        self.assertIn("transkrib.su", text)
+        self.assertIn("info@transkrib.su", text)
+        self.assertIn("schwed.2000@mail.ru", text)
+
+
+if __name__ == "__main__":
+    unittest.main()
```
