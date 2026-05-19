# Troubleshooting

If something's broken, find the symptom below.

---

## Backend side

### `curl https://your-url/health` does not return `{"status":"ok"}`

- **Connection refused / timeout** → the service isn't running. On Heroku: `heroku logs --tail`. On VPS: `systemctl status queue-backend` and `journalctl -u queue-backend -n 50`.
- **502 / 503** → app crashed. Look at logs. Most common cause: bad `DATABASE_URL`.
- **404** → wrong URL path. Health route is exactly `/health`.

### Logs show `DATABASE_URL is empty on Heroku`

The Postgres addon didn't attach or you removed it. Run:
```bash
heroku addons -a your-app-name
```
You should see a `heroku-postgresql` line. If not, recreate with `heroku addons:create heroku-postgresql:mini`.

### Logs show `relation "queues" does not exist` (or similar)

You skipped the migration step. Run:
```bash
# Heroku
heroku run "alembic upgrade head" -a your-app-name
# VPS
cd /opt/queue-backend && source venv/bin/activate && alembic upgrade head
```

### `sslmode=require` errors against a local Postgres

Local Postgres usually doesn't have SSL. The code only forces `sslmode=require` for non-`localhost` URLs (see [Admin/Database.py:28](../../Admin/Database.py:28)). Make sure your local `DATABASE_URL` uses `localhost` as the host.

---

## Auth / 401

### `{"detail":"Invalid or missing API key"}`

- Did you set `QUEUE_API_KEY` in the backend env? (Heroku: `heroku config -a your-app-name`. VPS: check `/opt/queue-backend/.env`.)
- Is your Lua sending `X-API-Key: <same value>` exactly? Header names are case-insensitive but the *value* must match byte-for-byte. No trailing whitespace.
- Heroku Config Vars vs `.env`: on Heroku, `.env` is ignored — only Config Vars are read.

---

## Validation / 422

### `{"detail":[{"loc":["body","region"],"msg":"value is not a valid enumeration member"}]}`

The value isn't in the enum. Check [Admin/schemas.py](../../Admin/schemas.py):
- Region: `EU`, `NA`, `ASIA` (uppercase)
- Position: `GK`, `CB`, `CM`, `ST`, `RDF`, `LDF`, `FW`, `LM`, `RM`
- QueueType: `regular`, `ranked` (lowercase)
- RankedTier: `pro`, `elite` (lowercase)
- TeamFormat: `5v5`, `7v7`

### `{"detail":[{"loc":["body","ranked_tier"],"msg":"field required"}]}`

If `queue_type == "ranked"` you **must** include `ranked_tier`. Same for `player_level` per member.

### `extra fields not permitted`

You sent a field that isn't in the schema. The schema is `extra="forbid"` — typos or extra debug fields are rejected. Check [docs/API_PAYLOADS.md](../API_PAYLOADS.md) for the exact field names.

---

## Roblox / HttpService side

### `HttpService is not enabled`

Roblox Studio → Game Settings → Security → "Allow HTTP Requests" → On. Republish if needed.

### `HTTP 0` / `attempt to call a nil value` on RequestAsync

You're calling from a LocalScript. `HttpService` only works on the server. Move the call to a ServerScript and use a RemoteEvent if a LocalScript needs to trigger it.

### `Trust check failed` / SSL errors

The backend URL must be HTTPS and have a valid certificate. Self-signed certs won't work with HttpService. Use Let's Encrypt (free, automatic via Certbot — see [DEPLOY.md § 11](DEPLOY.md#11-add-https-free-recommended)) or use Heroku (HTTPS included).

### Teleport fails: "Couldn't teleport"

- `MATCH_PLACE_ID` must be the place ID of an **actual published place** in your experience. Not a copy, not a draft.
- `TeleportService:ReserveServer` must be called from a Script in the **lobby** place, not the Studio play test.
- The lobby place and match place must be in the **same experience** (universe).
- The Roblox API key (`accessCode`) returned by `ReserveServer` is what `TeleportToPrivateServer` needs — don't confuse it with `privateServerId` (the latter is what you send to `/queue/{code}/reserve` as `job_id`).

---

## Queue behavior

### "Player is already in a queue" but they shouldn't be

The backend tracks players by `user_id`. If a Studio test crashed mid-queue, the row may still exist. Fix:
```bash
# Hit cleanup (it sweeps stale rows)
curl -X POST https://your-url/queue/cleanup/run -H "X-API-Key: <key>"
```
Or have the player call `/queue/leave/solo` explicitly. Cleanup runs lazily; calling it manually forces it.

### Queue never reaches `teleporting`

Two reasons:
1. Not enough players (5v5 needs 10, 7v7 needs 14). For testing, use:
   ```
   POST /queue/{queue_code}/test-force-start
   ```
   with body `{"job_id": "test-123"}` to jump straight to `teleporting`.
2. Cleanup killed the queue. Empty queues are removed after ~5 minutes.

### Players matched into wrong tier

Ranked queues use level ranges (Pro 30–60, Elite 60–99 per the enum docstring). Make sure `player_level` is being sent and is correct.

---

## "I don't know what's wrong"

Capture logs and share with your AI:

**Heroku:**
```bash
heroku logs --tail -a your-app-name
```

**VPS:**
```bash
journalctl -u queue-backend -n 200 --no-pager
```

The FastAPI app prints validation errors to stdout — they show exactly which field failed.
