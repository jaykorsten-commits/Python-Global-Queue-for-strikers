# Deploy

You have two practical options. Pick **one** and skip the other.

| Option | Cost | Difficulty | Good for |
|--------|------|------------|----------|
| **A. Heroku** | ~$5–10/mo | Easiest. Mostly clicking. | First-time deployers. No Linux skills required. |
| **B. VPS** (Hetzner / Contabo / DigitalOcean / Vultr) | ~$4–7/mo | Some terminal work. | Cheaper long-term. Full control. |

Both end with the same result: a public URL like `https://your-app.com` that responds to `GET /health` with `{"status":"ok"}`.

---

## Before you start (both options)

You need:

1. **A Postgres database.** You can use:
   - Heroku Postgres mini ($5/mo, automatic if you go Heroku)
   - Supabase free tier (https://supabase.com)
   - Neon free tier (https://neon.tech)
   - Postgres installed on your own VPS (free, see Option B)
2. **An API key.** Generate one now and save it somewhere safe:
   ```bash
   python -c "import secrets; print(secrets.token_urlsafe(48))"
   ```
   This becomes your `QUEUE_API_KEY`. The Roblox lobby server sends it on every request.
3. **Your Roblox MATCH place_id.** Open Roblox Studio → Game Settings → Places. It's the number next to the match place (not the lobby).

---

## Option A — Heroku

### 1. Install the Heroku CLI

https://devcenter.heroku.com/articles/heroku-cli — pick your OS, install, then `heroku login`.

### 2. Clone the repo and create the app

```bash
git clone <this-repo-url> queue-backend
cd queue-backend
heroku create your-app-name
```

### 3. Add Postgres

```bash
heroku addons:create heroku-postgresql:mini -a your-app-name
```

This sets `DATABASE_URL` automatically.

### 4. Set the other env vars

```bash
heroku config:set QUEUE_API_KEY="<paste your generated key>" -a your-app-name
heroku config:set MATCH_PLACE_ID="<your match place id>" -a your-app-name
```

### 5. Deploy

```bash
git push heroku main
```

### 6. Run database migrations

```bash
heroku run "alembic upgrade head" -a your-app-name
```

### 7. Verify

```bash
curl https://your-app-name.herokuapp.com/health
# → {"status":"ok"}
```

Your base URL is `https://your-app-name.herokuapp.com`. Give this + the API key to your AI agent.

> Don't use the free dyno — it sleeps. The Eco dyno ($5/mo) stays awake.

---

## Option B — VPS (Ubuntu 22.04 / 24.04)

This guide assumes a fresh Ubuntu VPS. Works the same on Hetzner, Contabo, DigitalOcean, Vultr, Linode, AWS Lightsail, etc.

### 1. SSH into your VPS

```bash
ssh root@YOUR_VPS_IP
```

### 2. Update and install dependencies

```bash
apt update && apt upgrade -y
apt install -y python3.12 python3.12-venv python3-pip git postgresql postgresql-contrib nginx ufw
```

### 3. Set up Postgres

```bash
sudo -u postgres psql -c "CREATE USER queue_user WITH PASSWORD 'STRONG_PASSWORD_HERE';"
sudo -u postgres psql -c "CREATE DATABASE queue_db OWNER queue_user;"
```

Save the password.

### 4. Clone the repo

```bash
cd /opt
git clone <this-repo-url> queue-backend
cd queue-backend
python3.12 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 5. Create the .env file

```bash
cp .env.example .env
nano .env
```

Fill in:
```
DATABASE_URL=postgresql://queue_user:STRONG_PASSWORD_HERE@localhost:5432/queue_db
QUEUE_API_KEY=<paste your generated key>
MATCH_PLACE_ID=<your match place id>
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

### 6. Run migrations

```bash
alembic upgrade head
```

### 7. Test the app runs

```bash
uvicorn Admin.main:app --host 0.0.0.0 --port 8000
```

In another terminal:
```bash
curl http://YOUR_VPS_IP:8000/health
# → {"status":"ok"}
```

Stop it with Ctrl+C.

### 8. Run as a systemd service (so it stays up)

Create `/etc/systemd/system/queue-backend.service`:
```ini
[Unit]
Description=Queue Backend FastAPI
After=network.target postgresql.service

[Service]
User=root
WorkingDirectory=/opt/queue-backend
EnvironmentFile=/opt/queue-backend/.env
ExecStart=/opt/queue-backend/venv/bin/gunicorn -w 2 -k uvicorn.workers.UvicornWorker -b 127.0.0.1:8000 Admin.main:app
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable queue-backend
systemctl start queue-backend
systemctl status queue-backend       # should say "active (running)"
```

### 9. Put nginx in front (for HTTPS later)

Create `/etc/nginx/sites-available/queue-backend`:
```nginx
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/queue-backend /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

### 10. Open the firewall

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
```

### 11. Add HTTPS (free, recommended)

If you have a domain pointing to the VPS:
```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d your.domain.com
```

Certbot auto-renews. Your base URL is now `https://your.domain.com`.

If you don't have a domain yet, you can use the raw IP over HTTP for testing, but get a domain + HTTPS before going live — Roblox HttpService handles HTTPS fine and it stops MITM attacks against your API key.

### 12. Verify

```bash
curl https://your.domain.com/health
# → {"status":"ok"}
```

---

## Database

The default is Postgres (any version 13+). The code uses SQLAlchemy + Alembic, so swapping to another DB is **possible but not zero-effort** — SQLite would work for local dev, MySQL would need migration edits, and DynamoDB/Mongo would require rewriting the data layer.

**Recommendation:** stay on Postgres. It's free or near-free everywhere and the migrations are already written.

If you want managed Postgres instead of self-hosting:
- **Supabase** (free up to 500 MB) — sign up, create project, copy the connection string into `DATABASE_URL`.
- **Neon** (free up to 0.5 GB) — same idea.
- **Heroku Postgres mini** ($5/mo) — automatic if you use Heroku.

Whatever you pick, set `DATABASE_URL` and run `alembic upgrade head` once.

---

## After deploy — hand these to your AI

Your AI agent needs three things to wire the bridge:

1. **Base URL** — e.g. `https://your-app-name.herokuapp.com` or `https://your.domain.com`
2. **`QUEUE_API_KEY`** — the one you generated above
3. **`MATCH_PLACE_ID`** — your Roblox match place ID (the same one set in backend env, and used in your Lua `TeleportService` call)

Now go to [ROBLOX_BRIDGE.md](ROBLOX_BRIDGE.md).
