# Codex -> Cloud/Cursor: Iskin Director Chat v2 Review Request

Date: 2026-06-18
From: Codex
Project: Transkrib Bot v2 / Iskin Director
Branch: `feature/iskin-director-chat`
Remote branch: `https://github.com/smolnirovgv-star/transkrib-bot/tree/feature/iskin-director-chat`
HEAD: `dd3100d feat(iskin): add director chat handlers`

## Summary

Codex implemented the first v2 layer: post-processing Iskin director chat.

Delivered scope:

- Iskin starts only after a video has been processed and SRT or episode context exists.
- Bot stores per-session Iskin context in `context.user_data`.
- Added `iskin_director.py` for plan helpers, prompt construction, JSON parsing, and AI plan updates.
- Added button after episode map: `Обсудить монтаж с Искином`.
- Added handlers for:
  - start Iskin chat;
  - finish Iskin chat;
  - show current plan by text;
  - update the plan through an AI JSON response.
- Prompt uses a compact internal agent chain:
  - Analyst;
  - DirectorPlanner;
  - Critic;
  - FinalEditor.
- The model is instructed to return valid JSON only and not reveal internal reasoning.
- If AI returns invalid JSON or fails, the previous plan remains unchanged.

## Verification

Fresh verification on 2026-06-18:

```text
py -m unittest
Ran 82 tests
OK

py -m py_compile bot.py episode_editor.py video_brief.py multi_agent_video_prompt.py billing.py claude_assistant.py iskin_director.py
exit 0
```

Local Telegram bot was not started.
Railway runtime was not changed.
Production bot was not touched.

## Non-Goals

- No voice input yet.
- No web UI yet.
- No video rebuild/apply-plan action yet.
- No payment/webhook work in this branch.
- No production deployment.

## Review Request

Please review:

1. Is `iskin_director.py` plan format acceptable for the next rebuild/apply-plan increment?
2. Is the Telegram integration safe enough for a Railway test-bot deploy?
3. Any copy/UX concerns around the button and messages?
4. Should the next increment be `Apply plan / rebuild video`, or should we first add plan history/undo?

## Proposed Separate Dev Bot Copy

Recommended BotFather setup for local-only or dev-only Iskin testing:

```text
Display name:
Transkrib Iskin Dev

Username candidate:
@transkrib_iskin_dev_bot
```

Purpose:

Separate dev bot for v2 Iskin testing only. It must use its own Telegram token and must never reuse the active Railway worker token or production token.

Suggested environment labels:

```text
BOT_NAME=Transkrib Iskin Dev
BOT_MODE=iskin_dev
```

Runtime rule:

Do not run this local/dev bot unless its Telegram token is different from all active Railway and production bot tokens.
