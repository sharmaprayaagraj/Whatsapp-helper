<div align="center">

# Family WhatsApp Assistant

**A local, 24/7 family assistant on WhatsApp — expense tracking, bill summaries, tech help and recipes — running entirely on your own machine.**

Python · FastAPI · Ollama · faster-whisper · SQLite · Node.js (Baileys)

[![License: MIT](https://img.shields.io/badge/license-MIT-3b82f6?style=flat-square)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-3776ab?style=flat-square&logo=python&logoColor=white)](https://www.python.org)
[![Tests](https://img.shields.io/badge/tests-101%20offline-16a34a?style=flat-square)](#tests)

</div>

## Overview

Three family numbers message a WhatsApp account; a local bridge forwards what they send to
a FastAPI backend, which transcribes voice notes, reads photos, classifies the intent with
a local Ollama model and answers. Expenses land in SQLite. Nothing — no message, no image,
no audio — ever leaves the machine: the LLM, the speech-to-text model and the database are
all local, and there is no Meta Cloud API, no tunnel and no token to manage.

```
WhatsApp ⇄ bridge/ (Baileys, :8790) ⇄ server.py (FastAPI, :8000) ⇄ Ollama (:11434)
                                              │
                                       family_hub.db (SQLite/WAL)
```

WhatsApp is connected through the **Baileys** bridge — WhatsApp Web with a QR login — so
the whole thing runs on one box behind your router, with both services bound to `127.0.0.1`.

## Features

- **Whitelist** — only three numbers (Dad, Mom, You). Enforced twice: the bridge drops
  strangers locally, and the backend re-checks authoritatively. Nothing from anyone else is
  processed or stored.
- **Six intents** — `budget_log`, `budget_query`, `doc_summary`, `tech_help`, `recipe`,
  `general_chat`, plus a keyword fallback layer for when the model is unavailable.
- **Voice notes** transcribed locally with **faster-whisper** before they ever reach the LLM.
- **Photos** read by the same local vision model (bills, letters, statements).
- **A safety gate on money** — the model's output is validated with Pydantic and
  cross-checked by regex against the sender's actual words. A mismatch or an invented
  amount turns into *"Did you mean $12.50 for Groceries?"* and is only written to the ledger
  after a `yes`.
- **Grounded answers** — spend queries are answered from SQLite totals (individual and
  combined household), not from model arithmetic.
- **Fast acknowledgement** — `/api/incoming` answers `202` in under 100 ms and queues the
  work on a single worker, so Meta's 5-second webhook rule is honoured by construction.
- **Runs offline** — the test suite needs no model downloads and no network.

## Requirements

- Python 3.10+ · Node.js 18+ · `ffmpeg` · [Ollama](https://ollama.com)
- Linux recommended (systemd units included). macOS and Windows work for development.

## Quick start — Linux, one shot

All commands run from the repository root (the folder holding
`family_whatsapp_bot/`, `bridge/` and `deploy/`).

```bash
sudo ./deploy/install.sh            # apt deps, venv, pip, npm, ollama, systemd
nano .env                           # ← set YOUR 3 numbers in WHITELISTED_NUMBERS
journalctl -u family-bridge -f      # scan the QR code with WhatsApp
./deploy/install.sh --check         # verify everything is green
```

`config.py` reads `.env` from the repository root *or* from `family_whatsapp_bot/.env`
(the latter wins) — either location works.

## Quick start — manual / development

```bash
# 1. backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r family_whatsapp_bot/requirements.txt
cp deploy/.env.example .env         # edit WHITELISTED_NUMBERS!

# 2. LLM — one multimodal model for text + images
curl -fsSL https://ollama.com/install.sh | sh
ollama pull llama3.2-vision

# 3. backend
uvicorn family_whatsapp_bot.server:app --host 127.0.0.1 --port 8000

# 4. bridge (new terminal) — scan the QR it prints
cd bridge && npm install && npm start
```

On Windows PowerShell the steps are the same, with `.\\.venv\\Scripts\\activate` and
`python -m family_whatsapp_bot.server` instead of the uvicorn line.

## Try it without WhatsApp

```bash
# health: whitelist, Ollama, whisper, queue depth
curl -s localhost:8000/health | python -m json.tool

# budget_log
curl -s localhost:8000/api/simulate -H 'content-type: application/json' \
  -d '{"from":"+12345678901","type":"text","text":"spent $24.50 on groceries — milk, eggs"}'

# budget_query — household and per-category totals from SQLite
curl -s localhost:8000/api/simulate -H 'content-type: application/json' \
  -d '{"from":"+12345678902","type":"text","text":"how much have we spent in total?"}'

# doc_summary — always three bullets: Summary / Action Required / Due Date
curl -s localhost:8000/api/simulate -H 'content-type: application/json' \
  -d '{"from":"+12345678901","type":"text","text":"summarize this electricity bill due 15 Oct"}'

# tech_help and recipe
curl -s localhost:8000/api/simulate -H 'content-type: application/json' \
  -d '{"from":"+12345678903","type":"text","text":"the wifi is not working"}'
curl -s localhost:8000/api/simulate -H 'content-type: application/json' \
  -d '{"from":"+12345678903","type":"text","text":"give me a recipe with eggs and spinach"}'

# a stranger is dropped instantly and nothing is stored
curl -s localhost:8000/api/incoming -H 'content-type: application/json' \
  -d '{"from":"+15550009999","type":"text","text":"let me in"}'
# → {"status":"ignored","reason":"not_whitelisted"}
```

## HTTP API

| Route | Purpose |
| --- | --- |
| `GET /health` | Whitelist, Ollama and Whisper status, queue depth |
| `POST /api/incoming` | Bridge callback: whitelist, enqueue, reply later via the bridge |
| `POST /api/simulate` | Full pipeline without WhatsApp, reply returned inline |
| `GET /api/expenses?limit=20` | Recent ledger entries from SQLite |

## How a message flows

1. The bridge receives it, drops strangers locally and `POST`s to `/api/incoming`.
2. The backend whitelists again, answers `202` in under 100 ms and enqueues.
3. A single worker turns voice notes into text with Whisper, base64-encodes photos for the
   vision model, then has Ollama classify the intent and draft a reply against a JSON
   contract.
4. `budget_log` passes the safety gate (Pydantic + regex against the original message).
   A match is saved; a mismatch asks for confirmation first.
5. The reply is grounded in SQLite totals, formatted in WhatsApp markdown (`*bold*`,
   `_italics_`, `•`, `1.`) and delivered back through the bridge.

## Tests

```bash
pip install -r family_whatsapp_bot/requirements.txt
python -m pytest family_whatsapp_bot/tests -q      # 101 tests (99 passed, 2 skipped), fully offline
node  --check bridge/index.js                      # bridge syntax
bash  -n deploy/install.sh                         # installer syntax
```

The suite uses temporary databases and a dead Ollama URL — no model downloads, no network,
deterministic on CI.

## Configuration

Everything lives in `.env`; the template is [`deploy/.env.example`](deploy/.env.example).

| Variable | Default | Meaning |
| --- | --- | --- |
| `WHITELISTED_NUMBERS` | placeholder trio | The only senders allowed, in order: dad, mom, user |
| `OLLAMA_MODEL` | `llama3.2-vision` | One multimodal model for text + images |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API |
| `BRIDGE_SHARED_SECRET` | placeholder | Auth between bridge and backend — change it |
| `WHISPER_MODEL_SIZE` | `base` | `tiny` → `medium`; bigger is slower and better |
| `WHISPER_DEVICE` | `auto` | `auto` / `cpu` / `cuda` |
| `CONTEXT_WINDOW_SIZE` | `8` | Sliding window of messages fed to the LLM |
| `CURRENCY_SYMBOL` | `$` | Shown in totals |
| `DB_PATH` | `family_whatsapp_bot/family_hub.db` | SQLite location |
| `PENDING_CONFIRMATION_TTL_SECONDS` | `300` | How long a "Did you mean…?" stays open |

**Model split (optional, needs ≥16 GB VRAM):** the default single `llama3.2-vision` avoids
model-swap stalls. If you want a text specialist, `ollama pull qwen2.5:7b`, set
`OLLAMA_MODEL=qwen2.5:7b`, and expect an extra load hop whenever a photo arrives.

## Project layout

```
family_whatsapp_bot/
  config.py         .env loading and typed settings
  server.py         FastAPI app, queue worker, /api routes
  ollama_engine.py  Model calls and the JSON contract
  prompts.py        Intent classification, prompts, keyword fallbacks
  transcriber.py    faster-whisper wrapper
  db_manager.py     SQLite schema, ledger and conversation turns
  knowledge.md      Architecture cheat sheet
  tests/            101 offline pytest tests
bridge/             Node.js Baileys bridge: QR session, media download, reply send
deploy/             install.sh, systemd units, .env.example
```

A longer, package-level guide lives in
[`family_whatsapp_bot/README.md`](family_whatsapp_bot/README.md).

## Troubleshooting

| Symptom | Fix |
| --- | --- |
| `/health` shows `degraded` | Ollama not running: `sudo systemctl start ollama` |
| Replies say "offline mode" | Model missing: `ollama pull llama3.2-vision` |
| Voice notes get "couldn't transcribe" | `pip install faster-whisper` and `apt install ffmpeg` |
| Bridge prints a QR on every restart | Session deleted — rescan; don't wipe `bridge/auth/` |
| Strangers reach the bot | Check `WHITELISTED_NUMBERS` in `.env`, restart `family-bot` |
| `database is locked` | A stray worker holds the DB; WAL + `busy_timeout` should prevent this |
| Bridge can't reach the backend | Both need the same `BRIDGE_SHARED_SECRET`; check ports 8000/8790 |

## Security

- Whitelist enforced twice: bridge pre-filter plus authoritative backend check.
- The bridge callback requires `X-Bridge-Secret`, and both ports bind `127.0.0.1` by
  default — nothing is exposed to the internet.
- The WhatsApp Web session in `bridge/auth/` is a credential: it is git-ignored, and it
  should stay off shared machines.
- No message content leaves the machine — LLM, speech-to-text and database are all local.

## License

[MIT](LICENSE) © 2026 CoderGamerUnknow
