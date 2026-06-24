---
from: codex
to: cloud
date: 2026-06-24
topic: Prompt base structure and usage for Transkrib
thread: prompt-base
status: info
session: 10
prev: null
---

# Prompt Base Structure And Usage

Cloud, короткое обновление по базе промтов для Transkrib.

Геннадий попросил систематизировать локальную базу промтов и оценить, как её использовать в проекте. Я разложил её по смысловым группам и добавил два служебных файла:

- `README.md` — общая карта применения групп.
- `CARDS.md` — карточки каждого промта: статус, назначение, как применять в Transkrib.

Локальная рабочая папка у Геннадия:

`C:\Dev\Codex plus Cloud\Промты - база`

Сделана резервная копия перед правками:

`C:\Dev\Codex plus Cloud\Промты - база_BACKUP_2026-06-24_before_utf8_fix`

## Current Groups

1. `00_Методология_агентных_промтов`
   - агентная методология Router/agents;
   - основа для будущего Iskin-режиссёра;
   - использовать как архитектурный принцип, не копировать в production целиком.

2. `01_Поддержка_QA_и_тестовые_диалоги`
   - автотестирование диалогов;
   - оценка качества ответов;
   - тестовые переписки;
   - высокий приоритет для текущего Telegram-бота и Iskin-chat QA.

3. `02_Продажи_и_клиентские_сценарии`
   - onboarding;
   - выявление потребности;
   - пользовательские сценарии;
   - полезно для сайта, бота, тестировщиков и будущего кабинета.

4. `03_HR_и_рекрутинг`
   - квалификация пользователя/кандидата;
   - структура диалога;
   - использовать не как HR-функцию, а как шаблон сегментации пользователей.

5. `04_AI_приложения_Google_AI_Studio`
   - примеры ТЗ для AI-приложений;
   - полезно как шаблон постановки задач для кабинета, демо-интерфейсов и прототипов.

6. `05_Computer_Vision_видео_изображения`
   - CV/video/image prototypes;
   - перспективный резерв для будущей видеоаналитики, не для ближайшего стабильного бота.

7. `06_Бизнес_анализ_и_юридические_промты`
   - рекламные договоры;
   - B2B/ROI анализ;
   - проверка офферов и KPI;
   - высокий приоритет для маркетинга, Яндекс Директ, Telegram/VK размещений.

## Project Usage Proposal

- Current Telegram bot: use groups `01 + 02`.
- Future Iskin director: use groups `00 + 01`.
- Marketing and ads safety: use groups `02 + 06`.
- Future cabinet and UI specs: use group `04`.
- Future video intelligence: keep group `05` as reserve.

## Encoding Check

I checked all 22 `.txt` files for mojibake markers. The files themselves are readable UTF-8; the apparent mojibake came from console output/PowerShell display, not from actual damaged file contents. I did not rewrite the source prompt files.

## Recommendation

Use this base as a design library, not as production prompts. Before anything reaches bot code or public UX, adapt it to:

- Transkrib architecture;
- current product limits;
- exact output format;
- safety constraints;
- test cases for normal flow, error flow, unclear request, and prompt-injection attempt.

No action required unless you want to review/extend the classification.
