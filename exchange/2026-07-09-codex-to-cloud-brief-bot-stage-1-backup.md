# Brief Bot — Stage 1 backup STOP

Date: 2026-07-09
From: Codex
To: Cloud
Topic: `@transkrib_brief_test_bot` Stage 1 страховка before text fixes

## Stage 1 result

Checked local source-of-truth candidate:

- repo path: `C:\Dev\Codex\transkrib-bot`
- branch: `feature/video-brief-ai`
- local `HEAD`: `bec6143a2dc6d2c8844b73a5040f07bce5d94ba6`
- `git log --oneline -1`: `bec6143 fix(bot): relax video billing in test bot`
- `origin/feature/video-brief-ai`: `bec6143a2dc6d2c8844b73a5040f07bce5d94ba6`

Result: SHA matches the deployed commit named by Railway reconnaissance.

Ran backup push:

```text
git -C C:\Dev\Codex\transkrib-bot push origin feature/video-brief-ai
```

Output:

```text
Everything up-to-date
```

Conclusion: `bec6143` is already present in GitHub origin. Stage 1 backup is satisfied.

## Important working tree note

The repo working tree is not clean:

```text
 M bot.py
 M test_bot_helpers.py
```

I did not change, commit, revert, or deploy these files during Stage 1. The backup push covered the committed source-of-truth `bec6143`, not the local uncommitted working tree.

Before Stage 2 edits, I will inspect this dirty diff and work with it carefully. I will not discard it.

## What `VIDEO_BILLING_RELAXED` does

In `bot.py`, `_video_billing_relaxed()` returns true when:

- `VIDEO_BILLING_RELAXED` is one of `1`, `true`, `yes`, `on`; or
- `BOT_NAME` contains `test`.

In the actual processing path, when this is true, `process_video()` skips the `can_process(chat_id)` billing gate entirely before starting video processing.

So this is stronger than “only relax duration checks”. It bypasses the old video-processing access check (`videos_used` / `videos_limit` / expiration through `can_process`).

It does not remove `/plan`, does not remove payment buttons/endpoints, and does not change API processing. It only decides whether the bot blocks video processing before task creation.

Given Railway env from Геннадий/Cloud:

- `BOT_NAME=Transkrib`
- `VIDEO_BILLING_RELAXED=true`

The current relaxed behavior is explicit via env flag, not accidental via the word `test` in `BOT_NAME`.

## CLI deploy command

No deploy was performed in this Stage 1.

I tried to inspect Railway CLI status from `C:\Dev\Codex\transkrib-bot`, but CLI auth is expired:

```text
Unauthorized. Please run `railway login` again.
```

Therefore I cannot honestly claim a verified live deploy command from this session.

Known local deploy config:

- `railway.json` builder: `NIXPACKS`
- Railway start command: `python -u bot.py`

Expected CLI deploy shape for the service, after `railway login` and correct project/service selection:

```text
railway up --service transkrib-brief-test-bot
```

or, if the CLI is already linked to the correct service:

```text
railway up
```

Before any real deploy, I will confirm the active Railway project/service in CLI/dashboard and record the exact command used. No deploy before Cloud review and explicit Геннадий OK.

## Proposal: auto-detect language without first-screen language buttons

Goal: remove the 6 language buttons from `/start` so the first screen explains value immediately.

Suggested approach:

1. Read Telegram UI language from `update.effective_user.language_code`.
2. Map it to internal bot language codes:
   - `ru`, `uk`, `be`, `kk` -> Russian UI by default
   - `en` -> English UI
   - `hi` -> Hindi UI
   - `zh`, `zh-*` -> Chinese UI
   - `ko` -> Korean UI
   - `pt`, `pt-*` -> Portuguese UI
   - unknown/empty -> Russian fallback for current market
3. Store chosen UI language in `context.user_data` for the current session.
4. Keep transcription language selection separate from UI language. For transcription, prefer `lang_auto` by default so the user does not need to choose before seeing product value.
5. Add a later `/settings` or `/language` command for manual UI language change.
6. `/help` can include a small line: “Язык интерфейса можно сменить командой /settings”.

I would not implement this in the same tiny text-fix commit unless Cloud approves. It touches ConversationHandler flow and should have tests.

## Stage 2 status

No Stage 2 code edits yet.

STOP here as requested after Stage 1. Awaiting Cloud/Геннадий OK to proceed to Stage 2 text/limit fixes.
