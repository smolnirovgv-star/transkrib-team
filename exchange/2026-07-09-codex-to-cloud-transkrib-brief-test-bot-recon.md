# @transkrib_brief_test_bot reconnaissance — 2026-07-09

Status: read-only reconnaissance. No product code changes, no deploys, no bot rename.

## Executive summary

1. `@transkrib_brief_test_bot` appears to be a two-part system: a Telegram polling bot from `transkrib-bot` plus the `transkrib-api` backend that does downloads, transcription, formatting, cutting, large uploads, and payment endpoints.
2. Exact Railway source-of-truth for the Telegram bot worker is not proven from local code alone. The likely candidate for the feature-rich brief-test behavior is `smolnirovgv-star/transkrib-bot`, branch `feature/video-brief-ai`, but this must be confirmed in Railway dashboard before any rename or merge.
3. The current `transkrib-bot` `main` branch is phone-video-only and rejects links. The feature branch has big-video UX, video brief helpers, episode flow, large upload buttons, and a relaxed billing mode for test bots.
4. The API backend supports YouTube, VK/VK Video, Rutube and broad HTTP/direct URLs through `yt-dlp`/fallbacks, plus Telegram CDN file URLs and large upload via Cloudflare R2. The practical duration limit is inconsistent: bot guard says 4 hours, API rejects over 3 hours.
5. Billing exists, but the test-bot branch can skip video billing when `BOT_NAME` contains `test` or `VIDEO_BILLING_RELAXED` is enabled. This is useful for beta testing but unsafe for a public flagship launch without a clear switch.
6. Cost accounting is incomplete. We can estimate costs from official prices, but current durable metrics do not store enough data to calculate real average cost per minute.
7. Production launch as flagship is feasible, but not ready as-is. The biggest blockers are source-of-truth confirmation, billing policy, cost logging, URL reliability, concurrency limits, text cleanup, and fixing the 3h/4h limit mismatch.

## 1. Architecture

### Local repos inspected

- API/backend: `C:\Dev\Cursor\Transkrib_SmartCut_AI`
  - remote: `https://github.com/smolnirovgv-star/transkrib-api.git`
  - branch: `main`
  - status during reconnaissance: clean
- Bot repo, current production-oriented main: `C:\Dev\Cursor\transkrib-bot`
  - remote: `https://github.com/smolnirovgv-star/transkrib-bot.git`
  - branch: `main`
  - status during reconnaissance: clean
- Bot repo, feature candidate: `C:\Dev\Codex\transkrib-bot`
  - branch: `feature/video-brief-ai`
  - remote branch: `origin/feature/video-brief-ai`
  - HEAD: `bec6143 fix(bot): relax video billing in test bot`
  - status during reconnaissance: dirty from previous local work in `bot.py` and `test_bot_helpers.py`; I did not change or revert it.

### API Railway deployment shape

Confirmed by code:

- `C:\Dev\Cursor\Transkrib_SmartCut_AI\railway.json`
  - Dockerfile builder
  - Dockerfile path: `backend/Dockerfile`
  - start command: `uvicorn main_railway:app --host 0.0.0.0 --port $PORT`
  - healthcheck path: `/health`
- `C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\main_railway.py`
  - includes `bot_tasks`, `bot_payments`, `admin_health`
  - starts temp cleanup and watchdog scheduler
- `C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\app\routers\bot_tasks.py:87`
  - default `TELEGRAM_BOT_USERNAME = transkrib_brief_test_bot`
  - large upload web page links back to this Telegram bot by env-configurable username.

Not confirmed from CLI/dashboard in this pass:

- Exact Railway project/service id for the Telegram polling process behind `@transkrib_brief_test_bot`.
- Whether its Railway Source is `smolnirovgv-star/transkrib-bot`, branch `feature/video-brief-ai`, `main`, or a different service/branch.
- Auto-deploy status for that exact Telegram bot service.

Recommendation before any implementation:

- Геннадий/Cloud should open Railway service for `@transkrib_brief_test_bot` and record:
  - Project name/id
  - Service name/id
  - Source repo
  - Branch
  - Auto-deploy on/off
  - Current env values for `BOT_NAME`, `BOT_MODE`, `TRANSKRIB_API_URL`, `VIDEO_BILLING_RELAXED`, `TELEGRAM_BOT_USERNAME`.

## 2. Capabilities

### Links accepted by the API backend

Confirmed in `C:\Dev\Cursor\Transkrib_SmartCut_AI\backend\app\routers\bot_tasks.py`:

- YouTube / YouTube Shorts:
  - YouTube ID extraction: line 1201
  - first tries `youtube-transcript-api` when available: lines 1809-1813
  - fallback chain includes `pytubefix`, `yt-dlp`, Cobalt, RapidAPI, Supadata and forced yt-dlp paths: lines 1444-1538 and later helpers.
- VK / VK Video:
  - normalizes `vkvideo.ru/video` to `vk.com/video`: lines 1746-1749
  - non-YouTube cutting path uses `yt-dlp`: lines 2218-2238.
- Rutube:
  - handled through non-YouTube `yt-dlp` path. Related validation in download service accepts `rutube.ru`.
- Direct HTTP / Telegram CDN file URLs:
  - Telegram CDN URLs are downloaded directly because yt-dlp cannot treat them as ordinary video sources: lines 1834-1853.
- Large local uploads:
  - `/api/large-upload/session`: line 2338
  - browser upload page: line 2442
  - multipart/presign R2 upload: lines 2849 and 2925
  - legacy direct upload: line 2974.

### What the bot UI seems to accept

There are two different local bot shapes:

- `C:\Dev\Cursor\transkrib-bot\bot.py` on `main` rejects URLs and says the bot accepts only files from phone up to 20 MB.
- `C:\Dev\Codex\transkrib-bot\bot.py` on `feature/video-brief-ai` has:
  - big-video upload button
  - video brief flow
  - large upload check flow
  - URL flow code, but also a separate URL reject handler. Which handler is active depends on the ConversationHandler wiring and must be tested against the actual deployed service.

This is why Railway source-of-truth is a blocker: the repository alone does not prove which behavior is live.

### Maximum duration

Code has a mismatch:

- API hard reject over 3 hours: `bot_tasks.py:1785` checks `duration > 3 * 3600`.
- Bot feature branch hard guard over 4 hours: `C:\Dev\Codex\transkrib-bot\bot.py:1838-1848`.

Practical expected limit today: 3 hours, because the backend can reject even if the bot allows up to 4 hours.

I did not find evidence of a real tested maximum duration in logs/docs during this read-only pass. Treat “3 hours” as code-derived, not production-proven.

### Maximum file size

- Telegram direct file path: 20 MB Bot API limit.
  - `C:\Dev\Cursor\transkrib-bot\bot.py:48`
  - `C:\Dev\Codex\transkrib-bot\bot.py:706`
- Large upload / R2 defaults:
  - `R2_MAX_FILE_GB=2` -> 2 GB per file by default: `bot_tasks.py:80`
  - `R2_PROJECT_SOFT_LIMIT_GB=8` -> 8 GB temporary project soft limit by default: `bot_tasks.py:81`
  - UI text says test free mode allows one file up to this R2 limit: `bot_tasks.py:2536`.

How it bypasses Telegram 20 MB:

- Files over 20 MB are not sent through Telegram Bot API.
- Bot creates a browser upload session on the API.
- User uploads to a web page; the browser uploads to Cloudflare R2 through presigned multipart upload.
- API then processes the uploaded object and sends results back to Telegram.

### Output formats

Confirmed API mapping in `bot_tasks.py:1754-1763`:

- `fmt_text` -> formatted text transcript
- `fmt_md` -> text path in API, Markdown handling belongs to bot side if used
- `fmt_cut` -> transcript + smart cut video
- `fmt_srt` -> SRT subtitles
- `fmt_cut_srt` -> cut video + SRT subtitles

Feature branch adds:

- video brief helper
- automatic cut / episode-first / transcript-only modes
- episode map and selected episode video assembly helpers.

### Timecodes

- SRT has real timestamps from Groq verbose JSON segments: `bot_tasks.py:890-915`.
- Text transcript is formatted by Claude and is not guaranteed to contain timecodes in every paragraph.
- Episode map likely contains time spans, but this should be verified by a real output sample before selling it as a guaranteed feature.

## 3. Billing

### Existing tariff/limit logic

`C:\Dev\Cursor\transkrib-bot\billing.py`:

- `bot_users` is read/created in `get_user()`.
- `bot_usage_log` is written by `log_usage()`.
- `can_process_video()` checks:
  - plan expiration
  - max minutes per video
  - sliding 24-hour video count.
- Current code limits:
  - free: 30 min/video, 3 videos/day
  - starter: 60 min/video, 3 videos/day
  - pro: 120 min/video, 10 videos/day
  - annual: no duration cap, 20 videos/day
  - tester: no duration cap, 9999 videos/day

Older/simple video counter also exists:

- `can_process()` checks `videos_used` / `videos_limit`.
- `increment_usage()` increments `videos_used`.

This means the billing code currently has two limit concepts: old total video count and newer duration/day limits. Before public launch we should decide which one is authoritative.

### Test-bot relaxed billing

`C:\Dev\Codex\transkrib-bot\bot.py:54-60`:

- Billing is relaxed if `VIDEO_BILLING_RELAXED` is true.
- Billing is also relaxed if `BOT_NAME` contains `test`.

During processing, if billing is relaxed, bot logs and skips `can_process()`: `C:\Dev\Codex\transkrib-bot\bot.py:1763-1775`.

This is probably why `@transkrib_brief_test_bot` can be used for broad experiments, but it is dangerous for a flagship public bot unless deliberately changed.

### Payment endpoints

Payment endpoints are in API `bot_payments.py`:

- YooKassa create/webhook
- LemonSqueezy create/webhook

Bug #27 proved LemonSqueezy signed webhook end-to-end for `transkrib-api`, but that does not automatically mean brief-test bot has a production-safe payment UX. The bot branch/source must be confirmed.

### Free access / whitelist

No dedicated whitelist for `@transkrib_brief_test_bot` was confirmed in local code. The feature branch has billing relaxed by bot name/env, not a user whitelist.

Recommendation:

- For public flagship, do not rely on `BOT_NAME contains test` behavior.
- Add explicit launch-mode config such as `BRIEF_BOT_ACCESS_MODE=closed|beta|public` or remove test-name billing relaxation before rename.

## 4. User-visible texts

### Current phone-only main branch `/start`

From `C:\Dev\Cursor\transkrib-bot\bot.py`:

```text
👋 Привет! Я {BOT_NAME}!

📹 Пришли видео-файл с телефона (до 20 МБ) — я сделаю транскрипцию и умную нарезку!

💡 Совет: можно снять прямо в чате — жми 📎 → Камера

🌐 transkrib.su · ✉️ info@transkrib.su

🌍 Choose your language:
```

### Feature branch `/start` candidate

From `C:\Dev\Codex\transkrib-bot\bot.py:1049-1080`:

```text
👋 Привет! Я {BOT_NAME}!

🧪 Тестовый бот
Сейчас идёт закрытая проверка для 2-3 пользователей. Возможны задержки и ошибки.
Не отправляйте конфиденциальные видео.
Пожалуйста, не передавайте ссылку на бота третьим лицам.
Если бот завис или выдал ошибку, пришлите время, действие и скрин/текст сообщения.

📹 Большие файлы загружайте через кнопку «Загрузить большое видео», но вы можете прислать видео-файл с телефона прямо в чат (до 20 МБ).

💡 Совет: можно снять прямо в чате — жми 📎 → Камера

🌐 transkrib.su · ✉️ info@transkrib.su

🌍 Выберите язык / Choose your language:
```

Buttons include languages, `Мой тариф`, `Обсудить задачу с AI`, `Загрузить большое видео`.

### Feature branch `/help` candidate

From `C:\Dev\Codex\transkrib-bot\bot.py:2360-2384`:

```text
🤖 {BOT_NAME} — что умеет бот:

🎬 Загрузка видео с телефона (до 20 МБ):
Отправь видео-файл прямо в чат или сними через 📎 → Камера.

📤 Большие видео:
Нажми «Загрузить большое видео», выбери файл в браузере и вернись в Telegram после загрузки.

⚙️ Режимы обработки:
- автоматическая нарезка;
- режим по эпизодам;
- только транскрипция;
- SRT субтитры и монтажная карта.

💳 Тарифы:
- пробный — бесплатно;
- Базовый — 450₽ / $5, 10 дней;
- Стандарт — 1700₽ / $20, 30 дней;
- Про — 8900₽ / $100, 365 дней.

📌 Команды:
/start — главная
/plan — тариф и подписка
/help — справка
/cancel — отменить обработку

📧 Контакты:
info@transkrib.su
support@transkrib.su
transkrib.su
```

Important inconsistency: this help text has `Стандарт — 1700₽ / $20`, while Bug #27 payment plan accepted `1500₽ / $15`. Must be cleaned before launch.

### Error/refusal texts found

Phone-only URL refusal:

```text
Сейчас бот принимает только видеофайлы напрямую с устройства. Пришлите ролик с телефона (до 20 МБ) — получите транскрипт, ключевые моменты и видео-нарезку.
```

Feature URL refusal:

```text
📵 Обработка ссылок пока не поддерживается. Пришлите видео-файл с телефона (до 20 МБ) или используйте кнопку «Загрузить большое видео».
```

Non-video file:

```text
⚠️ Это не похоже на видео-файл. Пришлите видео-файл с телефона (до 20 МБ).
```

Unable to get Telegram file:

```text
❌ Не удалось получить файл от Telegram. Попробуйте еще раз.
```

Unable to create large upload link:

```text
Не удалось создать ссылку для загрузки большого видео. Попробуйте еще раз через минуту.
```

Large upload not ready:

```text
Файл еще не загружен.

Если загрузка открыта в браузере, дождитесь сообщения «Видео загружено», затем вернитесь сюда и нажмите проверку еще раз.
```

Billing check failed:

```text
⚠️ Не удалось проверить тариф перед обработкой видео.

Попробуйте ещё раз через пару минут или откройте /plan.
```

No access by tariff:

```text
⚠️ Сейчас обработку запустить нельзя: {лимит бесплатных видео исчерпан|подписка закончилась}.

Откройте /plan, чтобы проверить тариф или продлить доступ.
```

Unable to determine duration:

```text
Не удалось определить длительность видео

Возможные причины:
- сервис временно перегружен; попробуйте через 5-10 минут;
- видео удалено, закрыто или имеет ограничения;
- есть региональные ограничения.

Попробуйте отправить видео еще раз или загрузить видеофайл с устройства.
```

Over 4 hours bot guard:

```text
🎬 Видео: {hours} ч {mins} мин

⛔ Превышен лимит 4 часа

Сейчас бот обрабатывает видео до 4 часов за один раз. Для более длинных видео разбейте материал на части и пришлите их по очереди.
```

Timeout:

```text
⏱ Превышено время ожидания ({minutes} мин). Попробуйте еще раз или свяжитесь с поддержкой.
```

## 5. Cost model / себестоимость

### APIs used in video pipeline

Confirmed by code:

- Groq Whisper:
  - SRT/verbose JSON: `whisper-large-v3` (`bot_tasks.py:890`)
  - text transcription: `whisper-large-v3-turbo` (`bot_tasks.py:895`)
- Claude:
  - formatting through `format_with_claude()` and `settings.claude_model`
  - code cost formula uses `$3 / 1M input tokens` and `$15 / 1M output tokens` in `bot_tasks.py:2066-2072`, matching Sonnet pricing.
- ffmpeg:
  - audio extraction/chunking/cutting locally on Railway.
- Download fallbacks:
  - yt-dlp, pytubefix, Cobalt, RapidAPI, Supadata, plus YouTube transcript API where possible.

### Official prices checked on 2026-07-09

Sources:

- Groq pricing: https://groq.com/pricing
  - Whisper V3 Large: `$0.111/hour transcribed`
  - Whisper Large v3 Turbo: `$0.04/hour transcribed`
  - audio minimum billing: 10 seconds/request
- Anthropic pricing: https://platform.claude.com/docs/en/about-claude/pricing
  - Claude Sonnet family used here: `$3 / 1M input tokens`, `$15 / 1M output tokens`
  - Claude Haiku 4.5, used by chat helper in bot repo: `$1 / 1M input`, `$5 / 1M output`
- Railway pricing: https://railway.com/pricing
  - pay by the second
  - CPU: `$0.00000772 / vCPU-second`
  - memory: `$0.00000386 / GB-second`
  - egress: `$0.05 / GB`

### Durable cost logging status

There is no complete durable per-video cost ledger today.

Found:

- `bot_api_usage` records chat/assistant token usage and `cost_usd` in `claude_assistant.py`.
- API task processing stores `claude_usage` in in-memory `tasks_store`.
- Bot can append this into `_daily_usage_store`, but this is process-local and not a durable per-video ledger.
- `task_metrics` records only:
  - `task_id`
  - `final_status`
  - `cut_status`
  - `download_method`
  - `formatter_status`
  - `processing_time_sec`

Missing for true tariff math:

- video duration seconds/minutes
- source type: YouTube transcript vs Groq vs large upload
- Groq model and cost
- Claude input/output tokens and cost per video
- ffmpeg processing seconds / output size / egress estimate
- final total cost per task

Conclusion: real average cost per minute cannot be computed reliably from current durable data.

### Theoretical estimate, 30 / 60 / 120 min

Assumptions:

- Text transcript uses `whisper-large-v3-turbo` when YouTube transcript is unavailable.
- SRT uses `whisper-large-v3`.
- Formatting uses Claude Sonnet pricing and a conservative rough transcript model: about 250 input tokens/min and 150 output tokens/min after formatting. Real output can be higher; use this as a floor-to-mid estimate, not a guarantee.
- Cutting/episode selection adds one or more Claude analysis calls plus ffmpeg CPU/egress.

| Video length | Groq text turbo | Groq SRT large | Claude formatting rough | Text-only total if Groq needed | Text+cut rough total |
|---:|---:|---:|---:|---:|---:|
| 30 min | $0.02 | $0.056 | ~$0.09 | ~$0.11 | ~$0.13-$0.25 |
| 60 min | $0.04 | $0.111 | ~$0.18 | ~$0.22 | ~$0.25-$0.50 |
| 120 min | $0.08 | $0.222 | ~$0.36 | ~$0.44 | ~$0.55-$1.10 |

If YouTube transcript API returns a transcript and Groq is skipped, transcription cost can be near zero, but formatting still costs money.

If Claude output is verbose, if multiple chunks are needed, or if episode maps are long, Claude cost can be 2-4x the rough formatting estimate.

### Hidden costs

- Railway CPU/RAM during download, ffmpeg extraction, chunking and cutting.
  - Example: 1 vCPU + 1 GB RAM for 10 minutes is about `$0.00695` before egress.
- Egress for returning cut videos/downloads.
  - 200 MB video delivery is about `$0.01` at `$0.05/GB`.
  - 1 GB delivery is about `$0.05`.
- Temporary storage / R2 upload costs. Large uploads default to 2 GB per file and 8 GB project soft limit; this can become material if files are not cleaned up or many users upload at once.
- Failed jobs still cost download/API/CPU time.

Recommendation before pricing launch:

- Add a `video_cost_events` or extend `task_metrics` with: `telegram_id`, `source_bot`, `duration_seconds`, `source_type`, `groq_model`, `groq_cost_usd`, `claude_input_tokens`, `claude_output_tokens`, `claude_cost_usd`, `processing_time_sec`, `output_bytes`, `estimated_railway_cost_usd`, `total_cost_usd`.
- Log failures too, not only successful jobs.

## 6. Stability

### How long in test mode

Based on repository history/docs, the long-video / video brief work has existed since at least mid-June 2026:

- `feature/video-brief-ai` commits include video brief and episode editor helpers.
- API large upload/R2 work is already in `transkrib-api` main.
- Older project notes mention YouTube ASN/download instability in May 2026 and later pivoting part of UX to phone-upload.

Exact number of processed brief-test videos was not computed, because live DB queries were not performed in this read-only pass and durable metrics do not include enough detail for per-bot attribution.

### Known risks / bugs

1. Source-of-truth ambiguity.
   - Current `main` and `feature/video-brief-ai` differ significantly.
   - Must confirm Railway Source before moving or renaming anything.

2. YouTube/download reliability.
   - Code has many fallbacks because YouTube/datacenter downloads are fragile.
   - Health checks and fallback state exist, but public users will still see intermittent failures for blocked/private/regional videos.

3. Duration limit mismatch.
   - Bot says 4 hours, API rejects over 3 hours.

4. Billing relaxation for test bot.
   - `BOT_NAME` containing `test` can skip billing.
   - Rename to `transkrib_brief_bot` may silently change billing behavior unless env/code is explicit.

5. Cost logging incomplete.
   - Cannot safely price one-off 30/60/120 minute products on real margins yet.

6. In-memory state.
   - `tasks_store` and `large_upload_sessions` are in-memory. Restart can lose active task/session state.
   - Telegram bot also relies on conversation/user_data state.

7. Concurrency.
   - FastAPI background tasks, ffmpeg, downloads, Groq and Claude calls run through the same service resources.
   - I did not find an explicit queue/backpressure system.
   - 10 simultaneous long videos may compete for CPU/RAM/temp disk/API limits and can cause timeouts, restarts, or cost spikes.

8. Text/product inconsistencies.
   - Tariff names/prices differ between branches and recent Lemon work.
   - Some feature branch comments/text show mojibake in comments, though user-facing Russian strings are mostly readable.

### What happens with 10 simultaneous users

Expected behavior from architecture:

- Bot accepts requests and creates tasks.
- API starts background work for each task.
- CPU-heavy ffmpeg work and network-heavy downloads happen concurrently on Railway.
- No durable queue means jobs are not naturally serialized or retried after restart.
- Long jobs can time out from polling/user side even if backend is still working.

Recommendation:

- Before flagship public launch, add either a small queue or a concurrency cap, for example max 1-2 active long-video tasks per worker with “you are in queue” UX.

## 7. Production readiness assessment

### Ready / promising

- Core backend can process links and large uploads.
- Good technical feature set: formatted transcript, SRT, cut video, episode map, large uploads.
- The strongest user value is exactly what respondent noticed: readable formatted text with punctuation/capitalization. This should be first-screen messaging.
- Payment backend for LemonSqueezy has already passed signed webhook smoke in Bug #27.

### Not ready for flagship public launch until fixed

1. Confirm Railway source-of-truth for `@transkrib_brief_test_bot`.
2. Decide and implement explicit billing mode for renamed public bot.
3. Remove `BOT_NAME contains test` as hidden billing control or replace with explicit env.
4. Align 3h/4h duration limits and text.
5. Clean `/start`, `/help`, `/plan`, pricing, and error texts.
6. Add one-off 30/60/120 minute payment products only after deciding whether they are time-limited credits or one-job checkout.
7. Add durable cost logging before final tariff math.
8. Add queue/concurrency control for long jobs.
9. Verify real live path with one URL, one large upload, one SRT, one cut, one failed/private URL.
10. Decide what to do with `@transkrib_smartcut_bot`: keep phone-only and make its refusal message send users to `@transkrib_brief_bot` instead of letting them leave.

### Rough work estimate

Assuming source-of-truth is confirmed and no surprise deployment mismatch:

- Source confirmation + branch/diff plan: 0.5 day
- Text rewrite for two bots: 0.5-1 day
- Explicit billing mode + test whitelist / beta mode: 0.5-1 day
- One-off payment model and DB schema: 1-2 days
- API + bot checkout integration for one-off minutes: 1-2 days
- Cost logging: 1 day
- Queue/concurrency cap: 1-2 days
- QA/smoke/Cloud review: 1 day

Conservative total: 5-8 engineering days before a careful public launch.

## Suggested next STOP gates

1. STOP 0 — Confirm Railway source-of-truth for `@transkrib_brief_test_bot`.
2. STOP 1 — Cloud approves product/billing model: subscriptions + one-off minutes.
3. STOP 2 — Codex prepares isolated diff, no deploy.
4. STOP 3 — Cloud review + Геннадий explicit OK.
5. STOP 4 — Deploy to beta/whitelist first.
6. STOP 5 — Public rename/link rollout only after smoke.

## Questions for Cloud / Геннадий

1. Which exact Railway service currently runs `@transkrib_brief_test_bot`?
2. What are its Source repo/branch and auto-deploy settings?
3. What is current `BOT_NAME` and is `VIDEO_BILLING_RELAXED` set?
4. Should renamed `@transkrib_brief_bot` be public immediately or beta/whitelist first?
5. For one-off payments: should 30/60/120 minutes buy one processing job, a credit balance, or access for a short time window?
6. Should YouTube transcript-only jobs be cheaper than jobs requiring Groq + Claude + cut video?
