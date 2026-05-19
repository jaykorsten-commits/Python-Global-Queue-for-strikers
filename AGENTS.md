# AGENTS.md — Instructions for AI Agents

You (the AI) are helping a developer integrate **their existing Roblox game** with **this FastAPI backend** to add a global matchmaking queue. The developer is relatively new to backend work — be explicit, ask before doing risky things (deploys, deletes, force pushes), and prefer the simplest working path.

This file is your contract. Read it before touching anything.

---

## Project shape

**Two codebases.**

1. **This repo** (`Admin/...` FastAPI + Postgres) — the queue backend. You are reading it now. It is **complete and working**. Treat it as a black box you call over HTTPS. Do not refactor it, do not "improve" it, do not add new endpoints unless the developer explicitly asks.
2. **The developer's Roblox project** (somewhere else — in Roblox Studio, `.rbxl` / `.rbxlx` files, or a Rojo project with `.lua`/`.luau` files). This is what you are modifying. You write Lua that calls this backend's HTTP API at the right moments.

The developer's AI flow looks like: *your* AI reads the developer's Roblox Lua code, figures out where the "join queue" / "leave queue" / "teleport to match" hooks should go, and adds `HttpService:RequestAsync` calls to this backend.

---

## What to do, in order

1. **Read these files in this repo (in this order):**
   - [HANDOFF.md](HANDOFF.md) — high-level overview
   - [docs/integration/ROBLOX_BRIDGE.md](docs/integration/ROBLOX_BRIDGE.md) — the bridge architecture + example Lua module
   - [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) — the full API contract (request/response for every endpoint)
   - [Admin/schemas.py](Admin/schemas.py) — authoritative request/response types, enums (Region, Position, QueueType, RankedTier, TeamFormat)

2. **Confirm with the developer:**
   - The base URL where this backend will be deployed (Heroku app URL, VPS IP/domain).
   - The `QUEUE_API_KEY` value (or generate one and tell them to set it in their backend env).
   - The Roblox **MATCH place ID** (the Roblox place ID for the actual match — separate from the lobby place).

3. **Read the developer's Roblox codebase.** Find:
   - The lobby place — where players click "play" / "queue" / "find match".
   - Any existing UI for picking a region / position / team format.
   - The match place — where `TeleportService` brings players in. The lobby reserves a server (using `TeleportService:ReserveServer`) and teleports players to this place.
   - Any existing party system (friends grouping in the lobby).

4. **Add the bridge.** Implement, in the developer's Lua codebase:
   - A `QueueClient` module (or whatever fits their structure) that wraps `HttpService:RequestAsync` to call each endpoint. The example in [docs/integration/ROBLOX_BRIDGE.md](docs/integration/ROBLOX_BRIDGE.md) is a starting point — **adapt it to their codebase conventions**, don't drop it in raw.
   - Hook the queue join/leave to the existing lobby UI events.
   - Implement the **reserve → teleport-info → teleport** flow on the lobby server when the API returns a queue with status `teleporting`.

5. **Test end-to-end** following [docs/integration/TROUBLESHOOTING.md](docs/integration/TROUBLESHOOTING.md).

---

## Hard rules — do not violate these

- **Never put the API key in the Roblox client (LocalScript).** Only server-side `Script`s call this backend. Roblox clients cannot be trusted with secrets. The API key lives in the lobby's server scripts only.
- **Never call this backend directly from a LocalScript.** Use `RemoteEvent`/`RemoteFunction` if the client needs to trigger a queue action; the server makes the HTTP call.
- **`HttpService.HttpEnabled` must be true** in Roblox game settings. Tell the developer to enable it: Game Settings → Security → "Allow HTTP Requests". Without this, every call fails.
- **Do not modify [Admin/](Admin/) Python code** unless the developer explicitly says "change the backend". If you think the backend is missing something, *ask first* and check [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) — the answer is usually that it already exists.
- **Do not invent endpoints or fields.** The schema in [Admin/schemas.py](Admin/schemas.py) is `extra="forbid"` — unknown fields cause a 422. Stick to what's documented.
- **Use HTTPS for the API URL.** Heroku and most VPSs with a reverse proxy give you HTTPS for free. Roblox `HttpService` works with HTTP but you should use HTTPS in production.

---

## API quick reference

All routes require header `X-API-Key: <QUEUE_API_KEY>` when the key is set in the backend's env. Content type is `application/json`.

| Purpose | Method + Path |
|---------|---------------|
| Health check | `GET /health` |
| Solo join (auto-match) | `POST /queue/join/solo` |
| Party join (2–4 players) | `POST /queue/join/party` |
| Leave the queue | `POST /queue/leave/solo` |
| Status of a user | `GET /queue/status/{user_id}` |
| Open queues in a region (counts) | `POST /queue/region/status` |
| Paginated server list (manual mode) | `POST /queue/list` |
| Join a specific queue by code | `POST /queue/join/manual` |
| Reserve a Roblox server for a full queue | `POST /queue/{queue_code}/reserve` |
| Teleport info after reserve | `GET /queue/{queue_code}/teleport-info` |
| Force-start a queue (DEV ONLY) | `POST /queue/{queue_code}/test-force-start` |
| Run cleanup pass | `POST /queue/cleanup/run` |

**Full payload examples**: [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md). Do not guess fields — look them up.

---

## Enums (these are the only valid values)

```
Region        = ["EU", "NA", "ASIA"]
Position      = ["GK", "CB", "CM", "ST", "RDF", "LDF", "FW", "LM", "RM"]
QueueType     = ["regular", "ranked"]
RankedTier    = ["pro", "elite"]   # required when queue_type == "ranked"
TeamFormat    = ["5v5", "7v7"]
QueueStatus   = ["open", "countdown", "starting", "teleporting", "live", "cancelled"]
```

If the developer's game uses different positions or regions, **ask them first** — adding to these enums means changing [Admin/schemas.py](Admin/schemas.py) and a DB migration; it is not a casual edit.

---

## Queue lifecycle (so you know where each call goes)

1. Player clicks "queue" in the lobby UI → LocalScript fires `RemoteEvent` → server calls `POST /queue/join/solo` (or `/queue/join/party`).
2. Backend returns the queue state. If `players_in_queue < max_players` → poll status (or push via a server loop) until it reaches `countdown` or `teleporting`.
3. When status becomes `teleporting`, **the lobby server** does:
   1. Call `TeleportService:ReserveServer(MATCH_PLACE_ID)` → gets `accessCode, privateServerId`.
   2. `POST /queue/{queue_code}/reserve` with `job_id = privateServerId` (claim it server-side).
   3. `GET /queue/{queue_code}/teleport-info` → returns the `user_ids` list and `place_id`.
   4. `TeleportService:TeleportToPrivateServer(place_id, accessCode, players)` for those user IDs.
4. If the player cancels: `POST /queue/leave/solo` with their `user_id`.

Polling cadence: every 2–3 seconds is fine. Backend has lazy cleanup, but a player leaving the lobby place should still call `/queue/leave/solo` (e.g. in `Players.PlayerRemoving`).

---

## Modifying the backend

If the developer **explicitly** asks to change the Python side:

1. Add a request/response schema in [Admin/schemas.py](Admin/schemas.py) (`extra="forbid"` stays).
2. Add the business logic to [Admin/QueueFunctions.py](Admin/QueueFunctions.py).
3. Wire the route in [Admin/routers/queue_service.py](Admin/routers/queue_service.py), depending on `verify_api_key`.
4. If you add/change columns: write an Alembic migration (`alembic revision -m "..."` then edit, then `alembic upgrade head`).
5. Update [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) and [docs/test_requests.http](docs/test_requests.http).

Do not touch [Admin/main.py](Admin/main.py) or [Admin/Database.py](Admin/Database.py) unless the developer asks.

---

## When in doubt

- **Architecture question** → re-read [HANDOFF.md](HANDOFF.md) and [docs/integration/ROBLOX_BRIDGE.md](docs/integration/ROBLOX_BRIDGE.md).
- **Payload question** → look it up in [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md). Do not guess.
- **Error from a call** → see [docs/integration/TROUBLESHOOTING.md](docs/integration/TROUBLESHOOTING.md).
- **Deploy question** → see [docs/integration/DEPLOY.md](docs/integration/DEPLOY.md).
- **Anything destructive** (DB resets, dropping tables, force-pushes, deleting Roblox places) → **ask the developer first**. They have asked us to be cautious because they are new to this.
