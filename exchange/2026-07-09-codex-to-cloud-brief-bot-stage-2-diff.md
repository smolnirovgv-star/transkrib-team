# Brief/SmartCut Bot - Stage 2 diff for Cloud review

Date: 2026-07-09
From: Codex
To: Cloud
Topic: text fixes + price mine + 3h duration mine

## Status

Implemented locally only. No commits in product repos, no push, no deploy.

Stage 1 dirty tree was resolved first:

```text
git -C C:\Dev\Codex\transkrib-bot stash push -m "pre-stage2-dirty-lemon-partial-bug27" -- bot.py test_bot_helpers.py
```

Current stash:

```text
stash@{0}: On feature/video-brief-ai: pre-stage2-dirty-lemon-partial-bug27
```

## Scope actually touched

### Brief bot source-of-truth candidate

Repo: `C:\Dev\Codex\transkrib-bot`
Branch: `feature/video-brief-ai`
Files changed: `bot.py`, `billing.py`, `test_bot_helpers.py`

Why `billing.py` is included: old price `1700/$20` lived in `PLAN_PRICES`, not only in `/help`. Leaving it would keep old price in payment currency selection.

### SmartCut bot main

Repo: `C:\Dev\Cursor\transkrib-bot`
Branch: `main`
Files changed: `bot.py`, `test_smartcut_texts.py` (new test file, intent-to-add only)

Reason: the critical URL fallback for `@transkrib_smartcut_bot` is in this repo, not in the brief-test branch.

## What changed

1. Brief `/start` now states formatted text, supported links/files, 3h limit, and first 3 free videos.
2. Brief tariff prices now use Bug #27 values: `450/$5`, `1500/$15`, `8900/$100`.
3. Brief 4h limit changed to 3h via `LONG_VIDEO_HARD_LIMIT_MIN = 180` and user-facing text.
4. Brief `/help`, `/plan`, and `show_plan` no longer hardcode the old pro price.
5. SmartCut `/start` now explains phone/short-video scope and points long links to `@transkrib_brief_test_bot`.
6. SmartCut `VIDEO_FALLBACK_MESSAGE` redirects link users to `@transkrib_brief_test_bot` instead of dead-ending them.

## Explicitly not done

- No language auto-detection implementation.
- No bot rename.
- No one-off minute payments.
- No cost logging.
- No Railway deploy.
- No product repo commits yet.

## Verification

RED first:

- Brief tests failed on old `/start`, old `1700/$20`, missing `LONG_VIDEO_HARD_LIMIT_MIN`.
- SmartCut tests failed on old fallback and old `/start`.

GREEN after implementation:

```text
C:\Users\Admin\OneDrive\Desktop\Cursor\Transkrib_Bot\.venv\Scripts\python.exe -m unittest test_bot_helpers.py
Ran 50 tests in 2.339s
OK
```

```text
C:\Users\Admin\OneDrive\Desktop\Cursor\Transkrib_Bot\.venv\Scripts\python.exe -m unittest test_smartcut_texts.py
Ran 2 tests in 0.012s
OK
```

Syntax check without writing `__pycache__`:

```text
syntax ok
syntax ok
```

`py_compile` was not used as final evidence because it tried to write `__pycache__` and hit Windows permission denial.

## STOP

Awaiting Cloud review and Gennady explicit OK before any product commit/push/deploy.

Railway CLI remains logged out in this environment; deploy is still blocked until login/service selection is resolved.

## Diff - brief bot (`C:\Dev\Codex\transkrib-bot`)

```diff
diff --git a/billing.py b/billing.py
index 7aec18b..02d9fbb 100644
--- a/billing.py
+++ b/billing.py
@@ -19,7 +19,7 @@ PLANS = {
 
 PLAN_PRICES = {
     "starter": {"rub": "450",  "usd": "5",  "days": 10,   "videos_limit": 9999, "name": "⭐ Базовый"},
-    "pro":     {"rub": "1700", "usd": "20", "days": 30,  "videos_limit": 9999, "name": "🚀 Стандарт"},
+    "pro":     {"rub": "1500", "usd": "15", "days": 30,  "videos_limit": 9999, "name": "🚀 Стандарт"},
     "annual":  {"rub": "8900", "usd": "100","days": 365, "videos_limit": 9999, "name": "👑 Про"},
 }
 
diff --git a/bot.py b/bot.py
index 9c4f8b1..c1fb15b 100644
--- a/bot.py
+++ b/bot.py
@@ -686,6 +686,8 @@ LANG_LABELS = {
 }
 
 MAX_VIDEO_SIZE_BYTES = 20 * 1024 * 1024  # 20 MB Telegram Bot API limit
+LONG_VIDEO_HARD_LIMIT_MIN = 180
+LONG_VIDEO_WARN_MIN = 120
 
 VIDEO_FALLBACK_MESSAGE = (
     "🎬 <b>Это видео больше лимита прямой отправки Telegram-боту.</b>\n\n"
@@ -1047,16 +1049,15 @@ async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
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
+        "Первые 3 видео - бесплатно. Карта не нужна.\n\n"
+        "Просто пришлите ссылку или файл.\n\n"
         "🌍 Выберите язык / Choose your language:",
         reply_markup=InlineKeyboardMarkup(keyboard),
         parse_mode="HTML",
@@ -1817,9 +1818,6 @@ async def process_video(chat_id, url, context):
     else:
         max_wait_seconds = 1800
 
-    LONG_VIDEO_HARD_LIMIT_MIN = 240
-    LONG_VIDEO_WARN_MIN = 180
-
     if duration_minutes > LONG_VIDEO_HARD_LIMIT_MIN:
         hours = duration_minutes // 60
         mins = duration_minutes % 60
@@ -1827,8 +1825,8 @@ async def process_video(chat_id, url, context):
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
@@ -2206,9 +2204,9 @@ async def cmd_plan(update, context):
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(uid)
     text += nl + nl + "📦 *Тарифы:*" + nl
-    text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
-    text += "🚀 Стандарт — 1700₽ / $20 — безлимит, 30 дней" + nl
-    text += "👑 Про — 8900₽ / $100 — безлимит, 1 год"
+    text += f"⭐ Базовый — {PLAN_PRICES['starter']['rub']}₽ / ${PLAN_PRICES['starter']['usd']} — безлимит, 10 дней" + nl
+    text += f"🚀 Стандарт — {PLAN_PRICES['pro']['rub']}₽ / ${PLAN_PRICES['pro']['usd']} — безлимит, 30 дней" + nl
+    text += f"👑 Про — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']} — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
@@ -2318,9 +2316,9 @@ async def handle_show_plan(update, context):
     nl = chr(10)
     text = "💳 *Твой тариф*" + nl + nl + get_status_text(tid)
     text += nl + nl + "📦 *Тарифы:*" + nl
-    text += "⭐ Базовый — 450₽ / $5 — безлимит, 10 дней" + nl
-    text += "🚀 Стандарт — 1700₽ / $20 — безлимит, 30 дней" + nl
-    text += "👑 Про — 8900₽ / $100 — безлимит, 1 год"
+    text += f"⭐ Базовый — {PLAN_PRICES['starter']['rub']}₽ / ${PLAN_PRICES['starter']['usd']} — безлимит, 10 дней" + nl
+    text += f"🚀 Стандарт — {PLAN_PRICES['pro']['rub']}₽ / ${PLAN_PRICES['pro']['usd']} — безлимит, 30 дней" + nl
+    text += f"👑 Про — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']} — безлимит, 1 год"
     kb = InlineKeyboardMarkup([[
         InlineKeyboardButton("⭐ Базовый", callback_data="buy_starter"),
         InlineKeyboardButton("🚀 Стандарт", callback_data="buy_pro"),
@@ -2334,20 +2332,24 @@ async def cmd_help(update, context):
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
+        f"- Про — {PLAN_PRICES['annual']['rub']}₽ / ${PLAN_PRICES['annual']['usd']}, 365 дней.\n\n"
         "📌 *Команды:*\n"
         "/start — главная\n"
         "/plan — тариф и подписка\n"
diff --git a/test_bot_helpers.py b/test_bot_helpers.py
index b5c4ef9..45c2e05 100644
--- a/test_bot_helpers.py
+++ b/test_bot_helpers.py
@@ -232,21 +232,25 @@ class SafeCallbackButtonTests(unittest.IsolatedAsyncioTestCase):
 
 
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
-        self.assertIn("Загрузить большое видео", text)
-        self.assertIn("но вы можете прислать видео-файл с телефона", text)
+        self.assertIn("Привет! Я Transkrib Brief.", text)
+        self.assertIn("пунктуацией", text)
+        self.assertIn("заглавными буквами", text)
+        self.assertIn("YouTube, VK, прямые файлы", text)
+        self.assertIn("до 3 часов", text)
+        self.assertIn("Первые 3 видео - бесплатно", text)
+        self.assertIn("Карта не нужна", text)
+        self.assertIn("Просто пришлите ссылку или файл", text)
         self.assertIn("Выберите язык", text)
+        self.assertNotIn("Тестовый бот", text)
+        self.assertNotIn("Не отправляйте конфиденциальные видео", text)
+        self.assertNotIn("до 4 часов", text)
         self.assertNotIn("Пришли видео-файл с телефона (до 20 МБ)", text)
         self.assertNotIn("РЎ", text)
         self.assertNotIn("СЂ", text)
@@ -422,6 +426,46 @@ class BotCommandMenuTests(unittest.IsolatedAsyncioTestCase):
         self.assertFalse(any("Р" in description or "СЂ" in description for description in descriptions))
 
 
+class BillingCopyTests(unittest.IsolatedAsyncioTestCase):
+    def test_plan_prices_match_current_source_of_truth(self):
+        self.assertEqual(bot.PLAN_PRICES["starter"]["rub"], "450")
+        self.assertEqual(bot.PLAN_PRICES["starter"]["usd"], "5")
+        self.assertEqual(bot.PLAN_PRICES["pro"]["rub"], "1500")
+        self.assertEqual(bot.PLAN_PRICES["pro"]["usd"], "15")
+        self.assertEqual(bot.PLAN_PRICES["annual"]["rub"], "8900")
+        self.assertEqual(bot.PLAN_PRICES["annual"]["usd"], "100")
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
+        self.assertNotIn("1700", text)
+        self.assertNotIn("$20", text)
+        self.assertNotIn("4 часа", text)
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
index 3358d3f..a70fb92 100644
--- a/bot.py
+++ b/bot.py
@@ -48,8 +48,13 @@ LANG_LABELS = {'lang_auto': '🔄 Авто', 'lang_ru': '🇷🇺 Русский
 MAX_VIDEO_SIZE_BYTES = 20 * 1024 * 1024  # 20 MB Telegram Bot API limit
 
 VIDEO_FALLBACK_MESSAGE = (
-    "Сейчас бот принимает только видеофайлы напрямую с устройства. "
-    "Пришлите ролик с телефона (до 20 МБ) — получите транскрипт, ключевые моменты и видео-нарезку."
+    "Я работаю только с файлами с устройства - ссылки не принимаю.\n\n"
+    "Но у нас есть бот, который умеет ссылки:\n"
+    "@transkrib_brief_test_bot\n\n"
+    "Там: YouTube, VK, длинные видео до 3 часов.\n"
+    "Первые 3 видео тоже бесплатно.\n\n"
+    "А если у вас короткий ролик на телефоне (до 20 МБ) -\n"
+    "пришлите файл сюда, обработаю за минуту."
 )
 
 LANG_CONFIRM = {
@@ -185,10 +190,17 @@ async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
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
+        "Первые 3 видео - бесплатно.\n\n"
+        "Совет: можно снять прямо в чате - жми скрепку, Камера.\n\n"
+        "---\n"
+        "Нужны ссылки на YouTube или длинные видео?\n"
+        "@transkrib_brief_test_bot (до 3 часов, любые ссылки)\n\n"
         "🌍 Choose your language:",
         reply_markup=InlineKeyboardMarkup(keyboard),
         parse_mode="HTML"
diff --git a/test_smartcut_texts.py b/test_smartcut_texts.py
new file mode 100644
index 0000000..a6fe13f
--- /dev/null
+++ b/test_smartcut_texts.py
@@ -0,0 +1,53 @@
+import unittest
+import os
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
+        text = bot.VIDEO_FALLBACK_MESSAGE
+
+        self.assertIn("Я работаю только с файлами с устройства", text)
+        self.assertIn("@transkrib_brief_test_bot", text)
+        self.assertIn("YouTube, VK, длинные видео до 3 часов", text)
+        self.assertIn("Первые 3 видео тоже бесплатно", text)
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
+        self.assertIn("Первые 3 видео - бесплатно", text)
+        self.assertIn("@transkrib_brief_test_bot", text)
+        self.assertIn("до 3 часов", text)
+        self.assertNotIn("Пришли видео-файл с телефона", text)
+
+
+if __name__ == "__main__":
+    unittest.main()
```
