# Chivers Home Setup — Plain-English Documentation

_Updated: 2026-09-29. Written by Hermes (the home AI agent), reviewed by Gemini and Claude._

This document explains what runs in this home, what each part does, and how they fit together. It is written for a network/security engineer: technical terms are fine, but I keep it plain and skip marketing fluff.

## The big picture

There are three servers on the home LAN (`192.168.81.0/24`, `home.arpa` DNS):

| Host | IP | Role |
|---|---|---|
| **nuc1** | 192.168.81.251 | AI brain: model gateway, local models, the home AI agent ("Hermes/Kal"), flight tracking, voice stack, SearXNG |
| **nuc2** | 192.168.81.252 | Media: IPTV ingestion (Dispatcharr), Emby media server, OpenBao secrets, reverse proxy |
| **nuc3** | 192.168.81.253 | Proxmox host running VMs (including an OPNsense firewall VM and a UniFi controller) |

All three are Intel NUC-class mini PCs on the same LAN. nuc1 has 64 GB RAM and an Intel Iris Xe iGPU (no discrete GPU — local models run on CPU).

---

## 1. LiteLLM — the model gateway (nuc1, port 4000)

**What it is:** a self-hosted LLM gateway exposing an OpenAI-compatible API at `http://nuc1:4000/v1`. Every AI request made in this house — by Hermes, Home Assistant, scripts, OpenWebUI — goes through it. It is the single door to AI models.

**What it does:**

- **Routing:** ~153 named models are configured. Names tell you the route: `or-...` = OpenRouter (hundreds of third-party models), `ha-*` = Home Assistant helpers, local names (`gpt-oss-local`, `qwen3-general`, `embed-local`) = Ollama on the same box. The bulk of traffic goes through the **Cloudflare AI Gateway** (Workers AI / `@cf/...` models), billed through the owner's Cloudflare account.
- **Access control:** a master key for admin; per-client keys can be issued with budgets and rate limits. All clients authenticate — nothing talks to a model provider directly.
- **State in PostgreSQL:** models, fallbacks, and virtual keys live in the database (Postgres 16 container), not in the YAML — the YAML is intentionally near-empty to avoid split-brain config.
- **Caching and router state in Valkey** (Redis-compatible, Alpine) — 64 MB, persistence on.
- **Fallbacks:** if a model fails, LiteLLM retries or falls back per configured chains (e.g. the home assistant models fall back between Cloudflare and OpenRouter variants).

**Why it matters:** one place to control cost, access, logging, and failover for every AI consumer in the house. No app holds its own API keys to external providers.

## 2. Ollama — local models (nuc1, port 11434)

**What it is:** a container running local open-weight models on CPU (Iris Xe iGPU is not used for inference).

**Models kept loaded:**

| Model | Size | Use |
|---|---|---|
| gpt-oss:20b | 14 GB | general local reasoning |
| qwen3.6:35b-a3b | 24 GB | newer local generalist |
| qwen3:30b-a3b | 19 GB | general (MoE, fast) |
| qwen3-coder:30b | 19 GB | code tasks |
| embeddinggemma | 0.6 GB | local embeddings |

Tuned for this hardware: flash attention on, KV cache quantized (q8_0), up to 3 models resident, 30-minute keep-alive. Exposed to the LAN for Home Assistant and OpenCode fallback, and wrapped in LiteLLM as `*-local` models.

**Why it matters:** fully-local inference works even when the internet or Cloudflare is down, and nothing local leaves the house.

## 3. Hermes — the home AI agent (nuc1)

**What it is:** the assistant you talk to (Telegram as "Kal", CLI, API). Version tracked from the upstream Hermes Agent project (Nous Research), installed under `~/.hermes/` with the gateway running as a service.

**What it does:**

- **Conversations** on Telegram (and CLI/API endpoints). One persistent agent identity per chat with memory across sessions.
- **Model access** only through LiteLLM (`http://nuc1:4000/web/v1`). Default model `glm-5.3-flash-Routed`; fallback chains handled by the gateway. No provider keys in Hermes itself.
- **Tools:** terminal, file editing, web search (SearXNG), browser automation (Cloudflare "Kitesurf" remote browser via Chrome DevTools protocol), code execution, image understanding, TTS/STT, Home Assistant control, and MCP servers (memory, media search, blog reader).
- **Memory:** two layers — a compact local memory file (injected into every session) and a Cloudflare-hosted vector memory (agent-memory-proxy worker) for semantic recall.
- **Scheduled jobs (cron):** ~14 jobs run autonomously — IPTV maintenance (see §4), a weekly stock research report, memory bookkeeping, media heartbeats, an IPsec tunnel watchdog. Each sends its report to Telegram in plain English.
- **API server:** an OpenAI-compatible endpoint on `nuc1:8642` so other home software can use the agent programmatically (authenticated).
- **Secrets:** no plaintext credentials in config — secrets resolve at runtime from OpenBao on nuc2 (and injected environment).

**In this house Hermes is also the operator:** it maintains the other systems — the IPTV pipeline, backups, watchdogs — under rules you set (no unapproved changes, all changes backed up, plain-English reporting).

## 4. IPTV pipeline (nuc2)

**What it is:** the system that takes a commercial IPTV subscription and turns it into a proper media library in Emby, with a curated channel list.

**Components:**

- **Dispatcharr** (nuc2:9191) — channel management: ingests the provider's channels and EPG, manages "profiles" (channel lineups). The curated profile is kept to a hand-picked set of channels.
- **IPTV provider** — an Xtream-Codes-API provider (base URL `http://uwptgchb.qvpng.com`). Provides live TV, VOD movies and series with metadata.
- **iptv-maint (Rust binary, on nuc2 at `~/bin/iptv-maint`)** — the maintenance engine, a rewrite of the original Python scripts. Eleven subcommands cover: syncing movies and series into Emby-compatible `.strm`/`.nfo` files, reconciling and cleaning the library, building and refreshing the EPG (TV guide), restoring channel groups after provider shuffles, checking provider health, and tagging Emby items. It runs from Hermes cron (nightly movie sync, twice-daily EPG refresh, weekly series/cleanup jobs, midnight provider check). All writes are preceded by state backups and every job reports to Telegram.
- **Emby** (nuc2:2095) — the media server the family actually uses. Library currently ~9,400 movies, 49 series, 1,000+ episodes, all populated by the pipeline above.
- **EPG sources** — iptv-org EPG plus the provider's own guide data, matched to channels and cached.

**Data flow:** provider → Dispatcharr (channels/EPG) and iptv-maint (VOD/series sync) → Emby library on disk → family watches on any Emby client.

## 5. Supporting cast

- **OpenBao** (nuc2:8200) — open-source HashiCorp Vault fork. Central secret store: Telegram bot token, Cloudflare token, LiteLLM key, etc. Agents and scripts fetch secrets at runtime; nothing sensitive sits in plaintext config. (Note: OpenBao has had an outage; workarounds documented in ops notes.)
- **Traefik ("tfk02", nuc2:80/443)** — reverse proxy for internal services with TLS.
- **Cloudflare Tunnel (`nagra-family-cloudflared`)** — selected internal services reachable remotely without port-forwarding; Cloudflare-side access policies apply.
- **SearXNG** (nuc1:3020) — self-hosted meta search engine, used as Hermes' web search backend (no tracking, aggregated results).
- **OpenWebUI** (nuc1:3000) — self-hosted ChatGPT-style front-end pointed at LiteLLM for manual model experiments.
- **Voice stack (nuc1)** — Wyoming Whisper (speech-to-text, port 10300) and Wyoming Piper (text-to-speech, port 10200) for Home Assistant voice, with `ha-voice`/`ha-brain`/`ha-vision` LiteLLM models routing assistant LLM calls.
- **Flight tracking (nuc1)** — FR24 feed, OpenSky, and PiAware containers feeding ADS-B data from an attached receiver.
- **The "Kal" download stack (nuc1)** — gluetun (VPN), qBittorrent, and a monitor container that fetches a specific show (The Daily Show) daily and drops it into Emby.
- **IPsec watchdog (nuc1 cron)** — pings the Cloudflare Magic-WAN IPsec tunnel peer every 5 minutes; alerts on failure via Telegram; safe restart sequence documented for Phase 2 (currently monitor-only).
- **Home Assistant** — runs on this LAN (integrates with voice stack and LiteLLM `ha-*` models).
- **nuc3 / Proxmox** — hypervisor (:8006) hosting VMs, including OPNsense (firewall with the Cloudflare IPsec tunnel) and UniFi Network controller.

## 6. Security posture (summary)

- Single model gateway with authenticated access; no provider keys scattered in apps.
- Secrets centralized in OpenBao; plaintext secrets banned from config files and repos.
- No unauthenticated public endpoints: everything either LAN-only, behind the authenticated API, or behind Cloudflare Tunnel with access policies.
- The only third-party paid surfaces are the IPTV subscription, OpenRouter (via LiteLLM), and Cloudflare (Workers AI, tunnels, memory).
- Backups before changes; watchdogs alert to Telegram; the agent follows a "report in plain English" convention for all automated messaging.

## 7. Where things can break

- **OpenBao down** → agents can't fetch secrets → scheduled jobs fail at the secret step (mitigations documented; fix OpenBao first).
- **Provider reshuffles** → channel groups and EPG drift → the `override-restore` and `epg-refresh` jobs exist exactly for this.
- **LiteLLM/Postgres down** → all AI features stop (Hermes, OpenWebUI, HA voice). Local Ollama still works directly on :11434.
- **Cloudflare outages** → remote browser, AI Gateway models, and remote memory degrade; local models and Emby are unaffected.
