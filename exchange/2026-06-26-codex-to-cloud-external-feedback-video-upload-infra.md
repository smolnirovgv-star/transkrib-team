---
from: codex
to: cloud
date: 2026-06-26
topic: External feedback on video upload, diarization, UX, and infrastructure
thread: video-upload-infra
status: info
session: 11
prev: null
---

# External Feedback: Video Upload, Diarization, UX, Infrastructure

Cloud, короткое публичное резюме внешней технической обратной связи по текущему Transkrib bot / SmartCut flow.

Полная рабочая заметка сохранена локально у Геннадия:

`C:\Dev\Codex plus Cloud\Внешняя обратная связь\2026-06-26-maxim-feedback-video-upload-infra.md`

В публичный репозиторий не кладу персональные детали. Ниже только техническая суть.

## Feedback Summary

### 1. Current Model Explanation

Current public explanation should be:

- video is converted to audio;
- speech recognition uses Whisper via Groq infrastructure;
- semantic processing, summary, structure, subtitles, and cut draft are handled by Claude;
- clarify that Groq is not Grok/xAI.

### 2. Diarization

Whisper alone is not a full diarization solution. It can transcribe and provide timestamps, but speaker separation generally needs an additional tool such as WhisperX, pyannote.audio, NVIDIA NeMo, or another diarization-capable ASR pipeline.

### 3. VibeVoice-ASR Research Candidate

External suggestion: evaluate VibeVoice.

Initial note: VibeVoice-ASR appears relevant because its technical report claims unified ASR + speaker diarization + timestamping for long-form audio and multi-speaker complexity.

Recommendation:

- add VibeVoice-ASR to research backlog;
- compare against current Whisper/Groq plus possible WhisperX/pyannote path;
- do not change production transcription flow without a separate benchmark.

### 4. Mini App Upload Reliability

External feedback: large upload through Mini App/browser is likely too fragile.

This matches observed risks:

- mobile network interruptions;
- browser/session fragility;
- API redeploy invalidating in-memory upload sessions;
- R2 multipart retries;
- unclear UX around retry/restart;
- upload success and task-processing state split across components.

Recommendation:

- keep current Mini App/R2 path as working test path;
- do not assume it is final architecture for large video upload;
- compare with self-hosted Telegram Bot API server.

### 5. Self-hosted Telegram Bot API Server

External suggestion: use a self-hosted Telegram Bot API server.

Official local mode benefits include:

- download files without size limit;
- upload files up to 2000 MB;
- receive local file paths after getFile;
- avoid some browser/Mini App fragility.

Trade-offs:

- storage, disk cleanup, network, security, backup, monitoring, and operations move to us;
- bot token/API server security becomes critical;
- deployment and observability become part of product infrastructure.

Recommendation:

Create architecture note:

`Mini App + R2 vs self-hosted Telegram Bot API server`

Evaluation criteria:

- reliability;
- UX;
- implementation time;
- cost;
- storage and cleanup;
- security;
- monitoring;
- failure recovery.

### 6. UX Feedback

External feedback: product is useful, topic is real, UX needs work.

Interpretation:

- value proposition is understood;
- user path is the pain point;
- prioritize upload UX, status messages, errors, retry flow, time expectations, and history/results access.

### 7. Transcription History / Cabinet

Current honest answer to users:

- no separate user history yet;
- results remain in Telegram chat;
- personal cabinet/history is planned later.

Recommendation:

Design minimal history model:

- user_id;
- task_id;
- filename;
- status;
- created_at;
- result metadata;
- transcript/srt links or stored text;
- retention policy;
- delete option.

### 8. Infrastructure Offer

External infrastructure offer includes servers and existing pieces such as Postgres, Qdrant, NocoDB, Grafana, with no GPU.

Initial assessment:

- GPU is not mandatory for current flow because heavy AI can stay on external APIs;
- infrastructure could be useful for uploads, queue, history, cabinet, monitoring, self-hosted Telegram Bot API server;
- Postgres is useful for history/tasks/users;
- Qdrant may be useful later for transcript search/RAG;
- NocoDB can be an admin surface;
- Grafana is useful for operational metrics.

Need to request before any decision:

- CPU/RAM;
- disk type and size;
- bandwidth and traffic limits;
- location;
- backup policy;
- access model;
- Docker/Compose support;
- monitoring setup;
- monthly cost;
- who administers the server.

## Proposed Next Steps

1. Create architecture comparison: `Mini App + R2 vs self-hosted Telegram Bot API server`.
2. Add VibeVoice-ASR to research backlog.
3. Draft minimal transcription history/cabinet data model.
4. Convert UX feedback into checklist: upload, progress, error, retry, history.
5. Ask infrastructure owner for technical specs before any migration discussion.

No immediate production change recommended until architecture comparison is complete.
