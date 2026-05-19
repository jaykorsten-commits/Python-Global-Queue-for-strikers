# Handoff — Roblox Global Queue Backend

**Read this first.** This is the FastAPI backend for a Roblox football game's global matchmaking queue (solo/party, regions EU/NA/ASIA, 5v5/7v7, regular/ranked). Your game stays in Roblox Studio (Lua); this service handles queue creation, matching, and teleport coordination over HTTP.

You don't change anything inside Roblox Studio's engine — you call this API from your Lua scripts (`HttpService:RequestAsync`) when a player wants to queue, leave, or get teleported into a match.

---

## What you're building

```
   ROBLOX (your game, Lua)                    THIS BACKEND (FastAPI + Postgres)
  ┌──────────────────────────┐               ┌────────────────────────────────┐
  │  Lobby place             │               │  /queue/join/solo              │
  │  ┌────────────────────┐  │   HTTPS       │  /queue/join/party             │
  │  │  QueueClient.lua   │──┼──────────────►│  /queue/status/{user_id}       │
  │  │  (HttpService)     │  │   JSON        │  /queue/leave/solo             │
  │  └────────────────────┘  │               │  /queue/{code}/reserve         │
  │           │              │               │  /queue/{code}/teleport-info   │
  │           ▼              │               └────────────────────────────────┘
  │  TeleportService         │                              │
  │  → Match place           │◄─────── job_id + place_id ───┘
  └──────────────────────────┘
```

Your AI's job: write Lua that calls these endpoints at the right moments. **You do not modify this Python backend** unless something is genuinely missing — you deploy it as-is and integrate.

---

## The 4 steps to go live

1. **Deploy the backend somewhere cheap.** Heroku Eco ($5/mo) is the easiest. A VPS like Hetzner CX11 (~$4/mo), DigitalOcean ($6/mo), or any Linux box also works. See [docs/integration/DEPLOY.md](docs/integration/DEPLOY.md).
2. **Set up the database.** Postgres is the default (any version 13+). Free options exist (Heroku Postgres mini, Supabase free, Neon free). See [docs/integration/DEPLOY.md](docs/integration/DEPLOY.md#database).
3. **Wire your Roblox game to the API.** Have your AI agent read your existing Lua codebase and add HTTP calls at the right places (lobby join button, match reserve, teleport). See [docs/integration/ROBLOX_BRIDGE.md](docs/integration/ROBLOX_BRIDGE.md).
4. **Test end-to-end.** Use Postman or `docs/test_requests.http` first, then test from Roblox Studio. See [docs/integration/TROUBLESHOOTING.md](docs/integration/TROUBLESHOOTING.md).

---

## If you're using an AI agent (Devin, Claude, Cursor, etc.)

Point it at [AGENTS.md](AGENTS.md) — that file is written for the AI and tells it:
- which files in **this** repo to read (and which to leave alone)
- which Lua files in **your** repo it needs to find and modify
- the exact API contract for each call
- how the queue lifecycle works so it puts calls in the right place

---

## Key files in this repo (so you know what's what)

| What | Where |
|------|-------|
| Entry point + routes | [Admin/main.py](Admin/main.py), [Admin/routers/queue_service.py](Admin/routers/queue_service.py) |
| Queue logic (matching, teams, slots) | [Admin/QueueFunctions.py](Admin/QueueFunctions.py) |
| Request/response shapes | [Admin/schemas.py](Admin/schemas.py) |
| DB models | [Admin/models.py](Admin/models.py) |
| Config (DB URL, API key) | [Admin/config.py](Admin/config.py) |
| **Full API payload reference** | [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) |
| HTTP request examples | [docs/test_requests.http](docs/test_requests.http) |
| DB migrations | [alembic/versions/](alembic/versions/) |

---

## Quick sanity check (after deploy)

```bash
curl https://YOUR_DEPLOYED_URL/health
# → {"status": "ok"}
```

If that returns `ok`, the backend is live. Now wire Roblox to it.

---

## Cost rough estimate

- **Heroku** Eco dyno + mini Postgres: ~$5–10/mo
- **VPS** (Hetzner / Contabo / Vultr small): $4–7/mo + free Postgres on same box
- **Supabase / Neon** free Postgres tier: $0 (with limits)

For a global queue that needs to be up 24/7, do **not** use Heroku free tier (it sleeps). Use Eco or higher, or a VPS that doesn't sleep.

---

## Need to extend the API?

Don't. First check [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) — there are already endpoints for solo, party, manual queue list, region status, reserve, teleport-info, leave, force-start (dev), and cleanup. The only thing your Roblox game needs to do is call them.

If you genuinely need something new (e.g. a different region, a new position, a new team format), see [AGENTS.md § Modifying the backend](AGENTS.md#modifying-the-backend).
