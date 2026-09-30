# Muse Bridge for 9Router

[![Python](https://img.shields.io/badge/python-3.8%2B-blue.svg)](https://www.python.org/)
[![9Router](https://img.shields.io/badge/9Router-provider-brightgreen.svg)](https://github.com/)
[![License](https://img.shields.io/badge/license-MIT-lightgrey.svg)](#license)

**Muse Bridge** turns [Muse](https://muse.ai) into a first-class **provider inside 9Router** — an OpenAI-compatible HTTP bridge with a file-based job queue, so any OpenAI client (9Router, Hermes, bots) can chat with Muse through the familiar `/v1/chat/completions` API. No SSH tunnels, no inbound ports, no public domain required.

---

## Table of Contents

- [How it works](#how-it-works)
- [Features](#features)
- [Repository contents](#repository-contents)
- [Requirements](#requirements)
- [Quick start](#quick-start)
  - [Option A — everything on one Linux VPS](#option-a--everything-on-one-linux-vps)
  - [Option B — bridge on your own PC, worker polls over Tailscale](#option-b--bridge-on-your-own-pc-worker-polls-over-tailscale)
- [Key management](#key-management)
- [Configuration](#configuration)
- [API reference](#api-reference)
- [Worker protocol](#worker-protocol)
- [Usage](#usage)
- [Speed: making responses fast](#speed-making-responses-fast)
- [Troubleshooting](#troubleshooting)
- [Security notes](#security-notes)
- [License](#license)

---

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
- **Tailscale-aware**: automatically also listens on the machine's Tailscale IPv4 address (with per-address fallback if the adapter isn't ready yet).
- **Long-polling** responses — no WebSockets, no callbacks, no public URL needed on the bridge side.

---

## Repository contents

| File | Purpose |
|------|---------|
| `bridge.py` | The bridge (v5.1). Run anywhere: VPS, PC, Raspberry Pi. |
| `run-bridge.ps1` | Windows helper: start the bridge or check its health (`-Status`). |
| `run-tunnel.ps1` | (Legacy) Cloudflare quick-tunnel helper for Windows. |
| `run-funnel.sh` / `run-funnel.ps1` | (Legacy) Tailscale Funnel helpers — only useful where Funnel is supported. |
| `SISI_MUSE.md` | Notes for the Muse-side worker (Indonesian). |
| `PANDUAN.md` | **Archive** — old v4 SSH-based design, kept for reference only. |

---

## Requirements

- **Bridge:** Python 3.8+ (standard library only).
- **9Router:** Node.js + npm (installed via `npm i -g 9router` or a local prefix).
- **Hermes:** the Hermes Agent CLI.
- **Remote option:** [Tailscale](https://tailscale.com) on both machines (any other tunnel that the worker's network can reach outbound also works).
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

**3. Create keys**

```bash
python3 ~/muse-bridge/bridge.py keygen --role user   --label 9router
python3 ~/muse-bridge/bridge.py keygen --role worker --label muse-worker
# Store each printed key in its own file with chmod 600. Keys are shown ONCE.
```

**4. Register the provider in 9Router** (via its REST API)

- Create a node of type `openai-compatible` named **Muse** with base URL `http://127.0.0.1:8765/v1`.
- Create a connection using the **user** key.
- Publish a combo/model named **`muse`** pointing at that node.

**5. Set up the worker** — a scheduled task (cron, systemd timer, or Muse scheduled task) that every minute:

1. `GET http://127.0.0.1:8765/muse/pending?limit=3` with `Authorization: Bearer <worker-key>`,
2. answers each job as Muse,
3. `POST http://127.0.0.1:8765/muse/answer` with `{"id": "<job-id>", "content": "<answer>"}`,
4. stays silent unless the bridge is unreachable or returns 401.

See [Worker protocol](#worker-protocol) for details.

**6. Install Hermes and point it at 9Router**

```yaml
# ~/.hermes/config.yaml
model:
  provider: custom
  base_url: http://127.0.0.1:20128/v1
  default: muse
```

Put the 9Router API key in `~/.hermes/.env` (`chmod 600`), then test:

```bash
hermes -z "are you connected via 9Router to the Muse provider?"
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

Run `hermes gateway run` as a systemd service. Chat with the bot → Hermes → 9Router → bridge → worker → reply.

### Option B — bridge on your own PC, worker polls over Tailscale

Use this when 9Router runs on your local PC but the worker lives elsewhere (e.g. a cloud VM whose egress can reach your tailnet).

**On the PC:**

1. Install [Tailscale](https://tailscale.com/download) and log in. Note the PC's tailnet IP (e.g. `100.x.y.z`).
2. Copy `bridge.py` + `run-bridge.ps1` to a folder, then:

```powershell
.\run-bridge.ps1          # starts the bridge; listens on 127.0.0.1:8765 + <tailscale-ip>:8765
.\run-bridge.ps1 -Status  # health check
```

3. Create keys:
```powershell
python .\bridge.py keygen --role user --label pc-9router
python .\bridge.py keygen --role worker --label vm-worker
```

4. In the PC's 9Router, register node **Muse** → `http://127.0.0.1:8765/v1` with the user key, and publish model **`muse`**.

**On the worker machine:**

- Poll `GET http://<pc-tailscale-ip>:8765/muse/pending` with the worker key. If the VM reaches the internet through a proxy, the tailnet address may need the tunnel proxy (in the reference setup, port `3130`, not the default `3128`).
- Answer via `POST http://<pc-tailscale-ip>:8765/muse/answer` through the same route.

> **Why Tailscale instead of Cloudflare Tunnel?** Cloudflare's edge was observed returning `403 Your request was blocked` for requests originating from some cloud VM egress networks — on *every* path, including `/health`. Tailscale avoids the public edge entirely.

---

## Key management

```bash
python3 bridge.py keygen --role user --label my-client    # create (printed once!)
python3 bridge.py keygen --role worker --label my-worker
python3 bridge.py keylist                                  # list (key prefixes only)
python3 bridge.py keydel <prefix-or-label>                 # revoke
```

- **user** keys authorize `POST /v1/chat/completions` (what 9Router uses).
- **worker** keys authorize `GET /muse/pending`, `POST /muse/answer`, `POST /muse/release`.
- The legacy `BRIDGE_TOKEN` env var, if set, is still accepted as a user key on `/v1/*`.

---

## Configuration

Environment variables read by `bridge.py`:

| Variable | Default | Description |
|----------|---------|-------------|
| `BRIDGE_QUEUE` | `/home/ubuntu/muse-bridge/queue` | Queue directory (`pending/`, `processing/`, `done/`). |
| `BRIDGE_KEYS` | `<queue>/../keys.json` | Key store file. |
| `BRIDGE_LEASE_SECS` | `180` | Worker lease duration; expired leases return to pending automatically. |
| `BRIDGE_TOKEN` | *(empty)* | Legacy token accepted as a user key on `/v1/*`. |

---

## API reference

Base URL: `http://<host>:8765`

| Method & path | Auth | Description |
|---------------|------|-------------|
| `GET /health` | none | `{"ok": true}` — liveness check. |
| `GET /v1/models` | user key | Lists the `muse` model (OpenAI format). |
| `POST /v1/chat/completions` | user key | OpenAI chat request (`model: "muse"`). Queues the job and long-polls for the answer (up to ~4 min). |
| `GET /muse/pending?limit=3` | worker key | Atomically leases up to `limit` jobs. Each job carries the original OpenAI request body. |
| `POST /muse/answer` | worker key | Body: `{"id": "<job-id>", "content": "<markdown reply>"}`. Duplicate answers are detected and reported. |
| `POST /muse/release` | worker key | Body: `{"id": "<job-id>"}`. Releases the lease so the job returns to pending. |

---

## Worker protocol

A worker can be any program (or scheduled AI-agent task) implementing this loop:

```
loop:
  jobs = GET /muse/pending?limit=3            (Authorization: Bearer <worker-key>)
  for job in jobs:
    try:
      reply = answer_as_muse(job.messages)    # follow the last user message's language
      POST /muse/answer  {"id": job.id, "content": reply}
    catch:
      POST /muse/release {"id": job.id}       # let another worker try
```

Rules of thumb from production use:

- **Stay quiet** on success — only alert a human on repeated connection failures or `401` (both need manual intervention).
- **Keep the hot path fast**: read the last user message plus a short slice of the system prompt; skip tool-call boilerplate the bridge doesn't support.
- **For latency**: a lightweight watcher that polls the queue directory (or `/muse/pending`) every ~10 s and wakes the worker immediately beats waiting for a 1-minute cron. Keep the cron as a fallback/sweeper.

---

## Usage

Once everything is up:

- **9Router UI / API:** pick the `muse` model and chat — requests flow through the bridge to the worker.
- **Hermes CLI:** `hermes -z "your question"` (default model is `muse`).
- **Telegram bot:** message the bot; allow-listing is enforced via `TELEGRAM_ALLOWED_USERS`.
- **Direct API test:**
  ```bash
  curl -s http://127.0.0.1:20128/v1/chat/completions \
    -H "Authorization: Bearer <9router-api-key>" \
    -H "Content-Type: application/json" \
    -d '{"model":"muse","messages":[{"role":"user","content":"Hello!"}]}'
  ```

---

## Speed: making responses fast

Measured end-to-end latencies in the reference deployment:

| Setup | Typical response time |
|-------|----------------------|
| Cron worker only (1 min interval) | 34–70 s |
| + 10 s queue-watch hook (local bridge) | **~15 s** |
| + 10 s queue-watch hook (PC bridge over Tailscale) | **~14 s** |
| Worker that reads the full 23k-token client system prompt | ~120 s |
| Same, but reading only the last user message + prompt head | **~14 s** (8.5× faster) |

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---------|-------------------|
| `curl /health` → connection refused | Bridge service not running: `systemctl status muse-bridge`. |
| 9Router returns model errors for `muse` | Provider base URL wrong, or the user key wasn't set on the connection. |
| Worker gets `401` | Wrong worker key, or the key was revoked (`keydel`). Generate a new one on the bridge host. |
| `WinError 10049` on Windows at startup | Old bridge version; v5.1+ falls back per listen address — upgrade `bridge.py`. |
| Tailscale IP unreachable from the worker VM | Tailnet traffic must go through the tunnel proxy (in the reference setup port `3130`, not `3128`). |
| Cloudflare tunnel returns `403` on all paths | Cloudflare is blocking the VM's egress network — switch to Tailscale/ngrok. |
| Bridge answers slowly via a coding assistant client | The client may send a huge system prompt; have the worker read only what it needs (see [Worker protocol](#worker-protocol)). |

---

## Security notes

- **Never commit keys.** Store every key in its own file with `chmod 600`; never paste them into chat, logs, or issues.
- The bridge has **no TLS** — bind it to `127.0.0.1` and the Tailscale IP only, or put it behind a reverse proxy with TLS.
- Restrict bot access with an allow-list (`TELEGRAM_ALLOWED_USERS`); bots are private by default in this setup.
- `keylist` shows prefixes only; full keys are printed exactly once at `keygen` time.

---

## License

MIT — see [LICENSE](LICENSE) (replace with your name). The bridge is dependency-free standard-library Python; do what you want with it.
