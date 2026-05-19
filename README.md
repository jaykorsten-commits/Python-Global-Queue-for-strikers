# Roblox Global Queue Backend

> **New here?** Start with **[HANDOFF.md](HANDOFF.md)** — it's the entry point for handing this repo to another developer.
> Using an AI agent (Devin, Claude, Cursor, etc.)? Point it at **[AGENTS.md](AGENTS.md)**.

FastAPI backend for a Roblox game’s global matchmaking queue (solo/party, regions, queue types, 5v5/7v7).

## Stack

- **FastAPI** — API and routes
- **PostgreSQL** — queues, slots, players (e.g. Heroku Postgres)
- **SQLAlchemy** — models and DB access

## Local run

```bash
# Optional: .env with DATABASE_URL or DB_* vars, and QUEUE_API_KEY for API auth
uvicorn Admin.main:app --reload
```

Health: `GET /health`

## Heroku

- Procfile runs: `gunicorn -w 1 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:$PORT Admin.main:app`
- Set `DATABASE_URL` (Postgres add-on) and `QUEUE_API_KEY` (or `API_KEY`) in config vars.
- Health checks can use `GET /health`.

## API and design

Full API contracts, payloads, cleanup rules, and security are in:

**[.cursor/plans/roblox-global-queue-backend_6b013cc3.plan.md](.cursor/plans/roblox-global-queue-backend_6b013cc3.plan.md)**

See **[Project_context.md](Project_context.md)** for file map and Cursor/AI context.

**Payload reference**: [docs/API_PAYLOADS.md](docs/API_PAYLOADS.md) — dict-style request/response examples.  
**Example generator**: `python scripts/generate_payload_examples.py` → `docs/PAYLOAD_EXAMPLES.txt`  
**Route tests**: `python scripts/test_routes.py` (server must be running).  
**HTTP requests**: [docs/test_requests.http](docs/test_requests.http) — for VS Code REST Client or curl.

## Core routes (current)

- `POST /queue/join/solo` — join queue (solo)
- `POST /queue/join/party` — join queue (party)
- `POST /queue/leave/solo` — leave queue
- `GET /queue/status/{user_id}` — current queue for user
- `POST /queue/region/status` — open queues / players in region
- `POST /queue/{queue_code}/reserve` — reserve (teleport flow)
- `POST /queue/{queue_code}/test-force-start` — dev/test only
- `GET /health` — liveness

All queue endpoints require `X-API-Key` when `QUEUE_API_KEY` (or `API_KEY`) is set.
