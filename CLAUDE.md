# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenClaw (aka Clawdbot/MoltBot) is an open, self-hosted AI agent platform deployed via Coolify. It runs as a Docker Compose stack and connects to chat platforms (WhatsApp, Telegram, Discord, Slack). The agent orchestrates sandbox containers for coding tasks, manages its own state, and provides web access via Cloudflare tunnels.

## Architecture

**Docker Compose services:**
- `openclaw` — Main agent container (Node 20 on Debian Bookworm). Runs the `openclaw gateway run` process on port 18789. Built from root `Dockerfile`.
- `docker-proxy` — Tecnativa socket proxy restricting Docker API access (no Swarm, Secrets, System, Volumes). OpenClaw reaches Docker via `tcp://docker-proxy:2375`.
- `searxng` — Private search engine (port 8080 internal). Built from `searxng/Dockerfile`.
- `registry` — Local Docker registry for image caching.

**Persistent volumes:** `openclaw-data` (`/data`), `openclaw-config` (`/root/.openclaw`), `openclaw-workspace` (`/root/openclaw-workspace`), `searxng-data`, `registry-data`.

**Key paths inside the container:**
- `/app` — Application code (this repo, copied via `COPY . .`)
- `/data/.openclaw` — State directory (`OPENCLAW_STATE_DIR`), contains `openclaw.json` config
- `/data/openclaw-workspace` — Agent workspace (`OPENCLAW_WORKSPACE`)
- `/data/.openclaw/state/sandboxes.json` — Sandbox state managed via lowdb

**Boot sequence:** `scripts/bootstrap.sh` → seeds agent workspaces (SOUL.md, BOOTSTRAP.md) → generates `openclaw.json` config with auth token → runs sandbox setup scripts → runs recovery/monitoring → starts `openclaw gateway run`.

## Key Files

- `SOUL.md` — Agent personality and operational rules (Prime Directive, container safety, image selection, state management, web operations protocol, recovery procedures)
- `BOOTSTRAP.md` — First-run context for the orchestrator agent
- `coolify.json` — Coolify marketplace manifest
- `docker-compose.yaml` — Service definitions and environment variable mappings
- `Dockerfile` — Multi-stage build: base system → runtimes (Bun, Python tools, Playwright) → dependencies (global npm/bun packages, OpenClaw CLI) → final image
- `.env.example` — All configurable environment variables (API keys, deployment tokens)

## Build & Deploy

```bash
# Build locally (requires Docker)
docker compose build

# Run the full stack
docker compose up -d

# View logs
docker compose logs -f openclaw

# Rebuild after changes
docker compose up -d --build
```

The primary deployment path is Coolify: point it at this repo as a public repository and it handles build/deploy automatically.

## Scripts

All in `scripts/`, made executable during Docker build:
- `bootstrap.sh` — Entrypoint. Generates config, seeds workspaces, starts gateway.
- `sandbox-setup.sh` — Configures sandbox container environment.
- `sandbox-browser-setup.sh` — Sets up browser capabilities in sandboxes.
- `recover_sandbox.sh` — Recovers sandbox containers and tunnels after restart.
- `monitor_sandbox.sh` — Background health checker (runs every 5 min).
- `openclaw-approve.sh` — Break-glass utility to accept pending pairing requests.
- `migrate-to-data.sh` — One-time migration of state to `/data` volume.

## Skills System

Skills live in `skills/` as folders containing `SKILL.md` (instructions) and `scripts/`. Current built-in skills:
- `web-utils` — Web search (`scripts/search.sh`) and scraping (`scripts/scrape.sh`, `scripts/scrape_botasaurus.py` for Cloudflare bypass)
- `sandbox-manager` — Sandbox container lifecycle management

External skills are installed via `clawhub install <slug>` (ClawHub registry).

## Critical Constraints

- **No `docker build` or `docker push`** — The agent discovers and runs existing images only. This is enforced by the docker-socket-proxy.
- **Container safety** — Only manage containers with label `SANDBOX_CONTAINER=true`, `openclaw.managed=true`, or name prefix `openclaw-sandbox-`.
- **No port exposure** — Sandbox containers must NOT use `-p`/`--port`. Public access goes through `cloudflared` tunnels.
- **State via lowdb** — All sandbox state tracked in `sandboxes.json`, not Docker metadata.
- **Data volumes are sacred** — Never delete `openclaw-data`, `openclaw-config`, or `openclaw-workspace` volumes without explicit user permission.

## Environment Variables

Key variables (set in `.env` or Coolify UI):
- `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`, `MINIMAX_API_KEY`, `KIMI_API_KEY` — LLM provider keys
- `TELEGRAM_BOT_TOKEN` — For Telegram channel
- `CF_TUNNEL_TOKEN` — Cloudflare tunnel access
- `GITHUB_TOKEN`, `GITHUB_USERNAME`, `GITHUB_EMAIL` — Git operations
- `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID` — Vercel deployments
- `OPENCLAW_BETA=true` — Install beta version of OpenClaw CLI
