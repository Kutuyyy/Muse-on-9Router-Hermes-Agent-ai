# Muse Bridge for 9Router

[![Python](https://img.shields.io/badge/python-3.8%2B-blue.svg)](https://www.python.org/)
[![9Router](https://img.shields.io/badge/9Router-provider-brightgreen.svg)](https://github.com/)

**Muse Bridge** runs **9Router + Hermes inside the Muse VM** and provides a bridge for integrating Muse with **Telegram, Discord, and WhatsApp bots**. Your bots can send messages to the Muse environment and receive AI-generated responses through the bridge, without exposing inbound ports, setting up SSH tunnels, or requiring a public domain.

## How it works

```
 ┌──────────┐   POST /v1/chat/completions    ┌──────────┐  queue as file   ┌────────┐
 │ 9Router  │  ──────────────────────────▶   │ bridge.py│ ───────────────▶ │ worker │
 │ :20128   │   (key role=user)               │  :8765   │  GET /muse/      │ (Muse) │
 └────▲─────┘                                │          │ ◀─────────────── │        │
      │        OpenAI-compatible              └────▲─────┘  pending/answer  └────────┘
      │        /v1/* API                           │       (key role=worker)
 ┌────┴─────┐      ┌──────────────┐                │  answer flows back,
 │  Hermes  │      │ Telegram /   │                │  client long-polls
 │  agent   │◀────▶│ Discord / WA │◀───────────────┘  up to 4 minutes
 └──────────┘      │ bot (gateway)│
                   └──────────────┘
```

1. A client (9Router, Hermes CLI, or a chat bot) sends a standard **OpenAI chat-completions request** to the bridge with the model name `muse`.
2. The bridge **queues the request as a JSON file** and holds the HTTP connection (long-poll, up to ~4 minutes).
3. A **worker** (any agent able to answer as Muse — in the reference setup, a scheduled Muse task) polls `GET /muse/pending`, picks up the job with an atomic lease, composes a reply, and posts it to `POST /muse/answer`.
4. The bridge delivers the reply to the waiting client as a normal OpenAI-style response.

Because the worker **pulls** jobs outbound, the bridge never needs inbound connectivity from the worker's network — it works behind NAT, and over Tailscale or any tunnel for remote setups.

---

## Features

- **OpenAI-compatible** `/v1/chat/completions` and `/v1/models` — drop-in for 9Router providers, Hermes custom models, or any OpenAI client.
- **File-based queue** with atomic worker leases (default 3 min), automatic lease expiry, crash recovery, and duplicate-answer detection.
- **Role-based API keys** (`user` / `worker`), plus legacy `BRIDGE_TOKEN` support.
- **Zero dependencies** — pure Python standard library. No `pip install` needed.

---

## Repository contents

| File | Purpose |
|------|---------|
| `bridge.py` | The bridge (v5.1). Upload it when u paste prompt. |
| `PROMPT.md` | Prompting to Muse.ai |

---

## Requirements

- **Bridge:** Python 3.8+ (standard library only).
- **9Router:** Node.js + npm (installed via `npm i -g 9router` or a local prefix).
- **Hermes:** the Hermes Agent CLI.
- **Bots:** a bot token (Telegram via BotFather, Discord via the Developer Portal, …) for the Hermes gateway.

---

## Quick start

### Option A — everything on one Linux VPS

This is the reference deployment: 9Router, bridge, worker, Hermes, and a Telegram bot all on the same machine.

**1. Install 9Router and run it as a service**

```bash
npm install -g 9router          # or use a persistent prefix like ~/.npm-global
sudo tee /etc/systemd/system/9router.service > /dev/null <<'EOF'
[Unit]
Description=9Router
After=network.target
[Service]
ExecStart=/home/<user>/.npm-global/bin/9router serve --port 20128 --host 127.0.0.1
Restart=always
User=<user>
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now 9router
```

Grab the 9Router API key from its first-run output (or config) and back it up with `chmod 600`.

**2. Run the bridge as a service**

```bash
mkdir -p ~/muse-bridge && cp bridge.py ~/muse-bridge/
sudo tee /etc/systemd/system/muse-bridge.service > /dev/null <<'EOF'
[Unit]
Description=Muse bridge for 9Router
After=network.target
[Service]
ExecStart=/usr/bin/python3 /home/<user>/muse-bridge/bridge.py serve
Restart=always
User=<user>
Environment=BRIDGE_QUEUE=/home/<user>/muse-bridge/queue
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now muse-bridge
curl -s http://127.0.0.1:8765/health   # -> {"ok": true}
```

**3. Install Hermes and point it at 9Router**

```yaml
# ~/.hermes/config.yaml
model:
  provider: custom
  base_url: http://127.0.0.1:20128/v1
  default: muse
```


**7. (Optional) Add a chat bot** — Telegram, Discord, or WhatsApp via the Hermes gateway.
Which one is up to you; the gateway handles the platform, the model chain stays
the same (bot → Hermes → 9Router → bridge → worker).

Example for Telegram:

```bash
# ~/.hermes/.env  (chmod 600)
TELEGRAM_BOT_TOKEN=<token from @BotFather>
TELEGRAM_ALLOWED_USERS=<your numeric Telegram user id>
```
