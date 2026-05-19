# Roblox Bridge — connecting Lua to this API

This is the **integration layer**. The backend is done; your Roblox game is done; this doc explains the glue.

> AI agent: read the developer's existing Lua codebase first. The example below is a starting point, **not a drop-in**. Names, RemoteEvent locations, UI hooks all depend on the developer's existing structure. Adapt to it.

---

## Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  LOBBY PLACE (Roblox)                                            │
  │                                                                  │
  │  ┌─────────────────┐    RemoteEvent     ┌─────────────────────┐  │
  │  │  LocalScript    │ ────────────────►  │  ServerScript       │  │
  │  │  (Queue UI)     │  "JoinQueue"       │  QueueService       │  │
  │  └─────────────────┘                    │  + QueueClient      │  │
  │                                         └──────────┬──────────┘  │
  │                                                    │             │
  │                                          HttpService:RequestAsync│
  │                                          + X-API-Key header      │
  └────────────────────────────────────────────────────┼─────────────┘
                                                       │ HTTPS
                                                       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  THIS BACKEND (FastAPI + Postgres, deployed at base URL)         │
  │  POST /queue/join/solo   POST /queue/leave/solo                  │
  │  GET  /queue/status/...  POST /queue/{code}/reserve              │
  │  GET  /queue/{code}/teleport-info  ...                           │
  └──────────────────────────────────────────────────────────────────┘
                                                       │
                                                       │ returns:
                                                       │ place_id, job_id (accessCode),
                                                       │ user_ids
                                                       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │  LOBBY ServerScript                                              │
  │  → TeleportService:ReserveServer(MATCH_PLACE_ID)                 │
  │  → TeleportService:TeleportToPrivateServer(...)                  │
  └──────────────────────────────────────────────────────────────────┘
                                                       │
                                                       ▼
                                          MATCH PLACE (the actual game)
```

Key idea: **only the lobby server speaks to the backend**. LocalScripts use RemoteEvents to ask the server to do queue actions. This keeps the API key off the client.

---

## Queue lifecycle — when to call what

| Player action | Lobby server does | Backend returns |
|---------------|-------------------|-----------------|
| Player clicks "Queue solo" | `POST /queue/join/solo` with `user_id`, region, positions, etc. | `queue` object with `status` and `queue_code` |
| Player clicks "Queue with party" | `POST /queue/join/party` with all member info | Same shape |
| Every 2–3 s while queueing | `GET /queue/status/{user_id}` | Updated `queue.status`, `players_in_queue` |
| Queue fills, `status` becomes `teleporting` | 1. `TeleportService:ReserveServer(MATCH_PLACE_ID)` → `accessCode, privateServerId`<br>2. `POST /queue/{code}/reserve` with `job_id = privateServerId`<br>3. `GET /queue/{code}/teleport-info` → list of user_ids and place_id<br>4. `TeleportService:TeleportToPrivateServer(place_id, accessCode, players)` | reservation confirmation; teleport info |
| Player clicks "Cancel" or leaves lobby | `POST /queue/leave/solo` with `user_id` | `{"message": "User left the queue."}` |

Only **one** lobby server in the entire game cluster needs to handle the reserve+teleport for a given queue. The first one to successfully call `/queue/{code}/reserve` wins; subsequent reserves return `{"already_reserved": true}` — that's fine, just stop and let the original server teleport.

---

## Status values you'll see

From [Admin/schemas.py](../../Admin/schemas.py):
- `open` — accepting players, not full yet
- `countdown` — full or nearly full, countdown running
- `starting` — countdown finished, server is being reserved
- `teleporting` — reserved, time to teleport
- `live` — match is running
- `cancelled` — queue died (e.g. all players left)

Your lobby logic only really branches on `teleporting` (do the reserve+teleport flow) and `cancelled` (show "queue dropped, try again" message).

---

## Example QueueClient module (Lua/Luau)

> **AI agent**: this is a template. Adapt naming, error handling, and event wiring to the developer's existing patterns. Don't blindly paste it.

Put this in a `ServerScript`-only location — e.g. `ServerScriptService.QueueClient` or wherever the developer keeps their server modules.

```lua
-- QueueClient.lua  (ServerScriptService)
-- Wraps HttpService calls to the FastAPI queue backend.
-- DO NOT REQUIRE FROM A LOCALSCRIPT — the API key would leak to clients.

local HttpService = game:GetService("HttpService")

local QueueClient = {}

-- CONFIG — replace with real values, or load from a private ModuleScript not shipped in client.
local BASE_URL = "https://YOUR_DEPLOYED_BACKEND_URL"
local API_KEY  = "PASTE_YOUR_QUEUE_API_KEY_HERE"

local function request(method, path, body)
    local url = BASE_URL .. path
    local headers = {
        ["Content-Type"] = "application/json",
        ["X-API-Key"]    = API_KEY,
    }

    local ok, response = pcall(function()
        return HttpService:RequestAsync({
            Url     = url,
            Method  = method,
            Headers = headers,
            Body    = body and HttpService:JSONEncode(body) or nil,
        })
    end)

    if not ok then
        warn("[QueueClient] HTTP error for " .. method .. " " .. path .. ": " .. tostring(response))
        return nil, "network_error"
    end
    if not response.Success then
        warn(("[QueueClient] %s %s → HTTP %d: %s"):format(method, path, response.StatusCode, response.Body))
        return nil, response.Body
    end
    return HttpService:JSONDecode(response.Body)
end

function QueueClient.joinSolo(userId, region, positions, teamFormat, queueType, rankedTier, playerLevel)
    return request("POST", "/queue/join/solo", {
        user_id      = userId,
        region       = region,        -- "EU" | "NA" | "ASIA"
        positions    = positions,     -- {"GK","CB"} — up to 4
        team_format  = teamFormat,    -- "5v5" | "7v7"
        queue_type   = queueType,     -- "regular" | "ranked"
        ranked_tier  = rankedTier,    -- nil unless ranked: "pro" | "elite"
        player_level = playerLevel,   -- 1..99, required if ranked
    })
end

function QueueClient.joinParty(partyId, region, members, teamFormat, queueType, rankedTier)
    -- members = { {user_id=111, positions={"CM","LM"}, player_level=45}, ... }
    return request("POST", "/queue/join/party", {
        party_id     = partyId,
        region       = region,
        members      = members,
        team_format  = teamFormat,
        queue_type   = queueType,
        ranked_tier  = rankedTier,
    })
end

function QueueClient.leave(userId)
    return request("POST", "/queue/leave/solo", { user_id = userId })
end

function QueueClient.status(userId)
    return request("GET", "/queue/status/" .. tostring(userId), nil)
end

function QueueClient.reserve(queueCode, jobId)
    return request("POST", "/queue/" .. queueCode .. "/reserve", { job_id = jobId })
end

function QueueClient.teleportInfo(queueCode)
    return request("GET", "/queue/" .. queueCode .. "/teleport-info", nil)
end

return QueueClient
```

---

## Example: wiring the join → teleport flow

```lua
-- ServerScript that handles a player wanting to queue.
-- Hook the RemoteEvent "QueueJoinRequested" from your existing lobby UI.

local Players          = game:GetService("Players")
local TeleportService  = game:GetService("TeleportService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local QueueClient = require(game.ServerScriptService.QueueClient)
local MATCH_PLACE_ID = 0000000000  -- your match place id

local QueueJoinRequested = ReplicatedStorage:WaitForChild("QueueJoinRequested") -- RemoteEvent

-- Active poll loops: userId -> coroutine that watches their queue
local watchers = {}

local function teleportPlayers(queueCode)
    local info = QueueClient.teleportInfo(queueCode)
    if not info or not info.success then return end

    -- ReserveServer must be called BEFORE TeleportToPrivateServer.
    local ok, accessCode, privateServerId = pcall(function()
        return TeleportService:ReserveServer(MATCH_PLACE_ID)
    end)
    if not ok then warn("ReserveServer failed: " .. tostring(accessCode)) return end

    -- Claim the reservation in the backend. If another lobby server beat us, bail out.
    local claim = QueueClient.reserve(queueCode, privateServerId)
    if not claim or not claim.success then return end
    if claim.already_reserved then return end  -- another lobby is doing it

    -- Collect Player objects from this lobby that match the queue's user list.
    local toTeleport = {}
    for _, uid in ipairs(info.user_ids) do
        local plr = Players:GetPlayerByUserId(uid)
        if plr then table.insert(toTeleport, plr) end
    end

    if #toTeleport > 0 then
        TeleportService:TeleportToPrivateServer(info.place_id, accessCode, toTeleport)
    end
end

local function watch(userId, queueCode)
    while watchers[userId] do
        task.wait(2)
        local status = QueueClient.status(userId)
        if not status or status.in_queue == false then
            watchers[userId] = nil
            return
        end
        local s = status.queue and status.queue.status
        if s == "teleporting" then
            teleportPlayers(queueCode)
            watchers[userId] = nil
            return
        elseif s == "cancelled" then
            watchers[userId] = nil
            return
        end
    end
end

QueueJoinRequested.OnServerEvent:Connect(function(player, region, positions, teamFormat, queueType, rankedTier, playerLevel)
    local res = QueueClient.joinSolo(player.UserId, region, positions, teamFormat, queueType, rankedTier, playerLevel)
    if not res or not res.success then
        -- Tell the client it failed. Use a RemoteEvent your UI already listens to.
        return
    end
    local queueCode = res.queue.queue_id  -- this is the queue_code, e.g. "eu_01"
    watchers[player.UserId] = true
    task.spawn(watch, player.UserId, queueCode)
end)

Players.PlayerRemoving:Connect(function(player)
    if watchers[player.UserId] then
        watchers[player.UserId] = nil
        QueueClient.leave(player.UserId)
    end
end)
```

This is illustrative — adapt to the developer's existing event names and UI.

---

## Integration checklist

For the AI agent and developer to confirm together:

- [ ] `HttpService.HttpEnabled = true` (Studio: Game Settings → Security)
- [ ] `BASE_URL` and `API_KEY` constants set in a **server-only** module (never accessible from LocalScript)
- [ ] `MATCH_PLACE_ID` matches the backend's env var and the actual published match place
- [ ] Lobby UI's "queue" button fires a `RemoteEvent` to the server (not a direct HTTP call)
- [ ] Server calls `QueueClient.joinSolo` / `joinParty` and stores the returned `queue_code`
- [ ] A polling loop watches `/queue/status/{user_id}` every 2–3 s
- [ ] On `status == "teleporting"`, server reserves a private server, calls `/reserve`, calls `/teleport-info`, and teleports
- [ ] `Players.PlayerRemoving` calls `/queue/leave/solo`
- [ ] "Cancel queue" UI also calls `/queue/leave/solo`
- [ ] HTTPS in production (not raw HTTP, not raw IP)

---

## Tip: test from Postman before testing from Roblox

Roblox HttpService failures are awkward to debug. Use [docs/test_requests.http](../test_requests.http) (VS Code REST Client) or Postman to verify the backend works end-to-end first. Then point Roblox at it.

Use `POST /queue/{code}/test-force-start` (dev only) to skip the wait-for-N-players step while testing the teleport flow with just one player.
