# SimpleFIN Bridge Setup
### Actual Budget Auto-Sync — Docker Compose Edition

---

This guide sets up `actual-auto-sync` — the service that pulls transactions
from your bank (via SimpleFIN) and pushes them into Actual Budget on a
schedule. The n8n workflows in this bundle read from Actual Budget, so
**this bridge must be running before any workflow will see your transactions.**

---

## What This Does

```
Your Bank
    │
    │  (SimpleFIN aggregator — connects most US banks)
    ▼
SimpleFIN Bridge
    │
    │  (actual-auto-sync container — runs twice daily by default)
    ▼
Actual Budget
    │
    │  (Actual Budget HTTP API — port 5006)
    ▼
n8n Workflows
```

---

## Step 1: Get Your SimpleFIN Token

1. Go to [https://beta-bridge.simplefin.org](https://beta-bridge.simplefin.org)
2. Create an account (~$15/year)
3. Connect your banks via the SimpleFIN interface
4. Navigate to **Settings → Access URL**
5. Copy the full access URL — it looks like:
   ```
   https://USERNAME:TOKEN@beta-bridge.simplefin.org/simplefin
   ```
   **Keep this private. Treat it like a password.**

---

## Step 2: Get Your Actual Budget Sync ID

In Actual Budget:
1. Go to **Settings** (gear icon)
2. Find **Sync** or **Advanced Settings**
3. Copy the **Sync ID** — a short alphanumeric string

---

## Step 3: Docker Compose

Create a file called `docker-compose.yml` in a folder of your choice
(e.g. `~/actual-sync/docker-compose.yml`):

```yaml
version: "3.8"

services:
  actual-auto-sync:
    image: ghcr.io/sakowicz/actual-auto-sync:latest
    container_name: actual-auto-sync
    restart: unless-stopped
    environment:
      # ── Actual Budget ────────────────────────────────────────
      # Your Actual Budget server URL (no trailing slash)
      # If Actual Budget is also in Docker on the same host,
      # use the service name: http://actual_server:5006
      ACTUAL_SERVER_URL: "http://YOUR_ACTUAL_BUDGET_URL:5006"

      # Your sync ID from Actual Budget → Settings
      ACTUAL_SYNC_ID: "YOUR_SYNC_ID_HERE"

      # Password for your Actual Budget instance (if set)
      # Leave blank ("") if you don't use a password
      ACTUAL_PASSWORD: ""

      # ── SimpleFIN ────────────────────────────────────────────
      # Your full SimpleFIN access URL (from Step 1)
      SIMPLEFIN_URL: "https://USERNAME:TOKEN@beta-bridge.simplefin.org/simplefin"

      # ── Sync schedule ────────────────────────────────────────
      # Cron expression for how often to sync
      # Default: twice daily at 6am and 6pm
      # Format: "MINUTE HOUR * * *"
      SYNC_CRON: "0 6,18 * * *"

      # How many days of transaction history to fetch on first run
      START_DATE_DAYS_AGO: "30"

      # ── Logging ──────────────────────────────────────────────
      # "debug" | "info" | "warn" | "error"
      LOG_LEVEL: "info"

    # Expose port if you want to check sync status via the API
    # Remove this block if you don't need external access
    ports:
      - "3788:3000"

    # Optional: persist logs between restarts
    volumes:
      - actual-sync-data:/app/data

volumes:
  actual-sync-data:
```

---

## Step 4: Start the Bridge

```bash
# Navigate to your compose folder
cd ~/actual-sync

# Start in the background
docker compose up -d

# Watch the first sync run
docker compose logs -f actual-auto-sync
```

A successful first sync looks like:
```
[INFO] Starting actual-auto-sync
[INFO] Connecting to Actual Budget at http://...
[INFO] Sync ID: xxxxxx — budget found
[INFO] Fetching transactions from SimpleFIN...
[INFO] Fetched 47 transactions from 3 accounts
[INFO] Pushing to Actual Budget...
[INFO] Done. Next sync at 18:00
```

---

## Step 5: Verify in Actual Budget

After the first sync:
1. Open Actual Budget
2. Check your accounts — you should see recent transactions
3. Some may be uncategorized — that's what Workflow 01 (Categorizer) is for

---

## Version Pinning (Recommended)

The `latest` tag works but can break when the image updates if your
Actual Budget server version doesn't match. To pin to a specific version:

```yaml
# Check https://github.com/sakowicz/actual-auto-sync/releases for latest version
image: ghcr.io/sakowicz/actual-auto-sync:26.4.0
```

**Important:** The sync image version should match your Actual Budget
server version. If you upgrade one, upgrade the other at the same time.
Mismatched versions (e.g. sync on 26.3.0, server on 26.4.0) will cause
sync failures or silently skip transactions.

To check your current Actual Budget server version:
```bash
docker exec actual_server cat package.json | grep '"version"'
```

---

## Troubleshooting

### "Cannot connect to Actual Budget server"
- Make sure your `ACTUAL_SERVER_URL` is reachable from inside the container
- If both services are in Docker, use the container/service name, not `localhost`
  - ✅ `http://actual_server:5006`
  - ❌ `http://localhost:5006`  (localhost inside the container = the container itself)
- If using an external URL, make sure port 5006 is accessible

### "Invalid sync ID"
- Double-check the sync ID from Actual Budget → Settings
- Make sure you copied the whole string with no extra spaces

### "SimpleFIN authentication failed"
- Re-copy your access URL from SimpleFIN — they expire if regenerated
- Make sure the URL has the full format: `https://USER:TOKEN@beta-bridge.simplefin.org/simplefin`

### Transactions aren't appearing in Actual Budget
- Check logs: `docker compose logs actual-auto-sync`
- Confirm the sync ran (look for "Done" in logs)
- SimpleFIN sometimes has a 24–48hr delay for newly connected banks
- Check that your bank accounts appear in SimpleFIN's dashboard

### Version mismatch errors
- Pin both containers to matching versions (see Version Pinning above)
- After updating: `docker compose pull && docker compose up -d`

---

## Running Multiple Budgets

If you have more than one Actual Budget file (e.g. personal + business),
run a separate `actual-auto-sync` container for each with a different sync ID
and a different port mapping:

```yaml
services:
  actual-sync-personal:
    image: ghcr.io/sakowicz/actual-auto-sync:latest
    environment:
      ACTUAL_SYNC_ID: "your-personal-sync-id"
      # ...
    ports:
      - "3788:3000"

  actual-sync-business:
    image: ghcr.io/sakowicz/actual-auto-sync:latest
    environment:
      ACTUAL_SYNC_ID: "your-business-sync-id"
      # ...
    ports:
      - "3789:3000"   # different host port
```

---

## Connecting the Bridge to n8n

The n8n workflows communicate directly with **Actual Budget's HTTP API**,
not with the sync bridge. The bridge's only job is keeping Actual Budget
populated with fresh transactions.

You do not need to point n8n at port 3788. Point n8n at your Actual Budget
server URL (port 5006) — that's the `actualBudgetUrl` in every CONFIG node.

```
n8n → Actual Budget (:5006)   ✅  correct
n8n → actual-auto-sync (:3788) ❌  wrong
```

---

## Adding to an Existing Docker Compose Stack

If you already run Actual Budget via Docker Compose, add the sync service
to your existing file rather than creating a new one:

```yaml
# In your existing docker-compose.yml, add under services:

  actual-auto-sync:
    image: ghcr.io/sakowicz/actual-auto-sync:latest
    container_name: actual-auto-sync
    restart: unless-stopped
    environment:
      ACTUAL_SERVER_URL: "http://actual_server:5006"
      ACTUAL_SYNC_ID: "YOUR_SYNC_ID"
      ACTUAL_PASSWORD: ""
      SIMPLEFIN_URL: "https://USERNAME:TOKEN@beta-bridge.simplefin.org/simplefin"
      SYNC_CRON: "0 6,18 * * *"
      START_DATE_DAYS_AGO: "30"
      LOG_LEVEL: "info"
    networks:
      - your-existing-network   # same network as your Actual Budget container
```

---

*actual-auto-sync by [@sakowicz](https://github.com/sakowicz/actual-auto-sync)*
*SimpleFIN by [SimpleFIN.org](https://simplefin.org)*
*This setup guide is part of the n8n Finance Automation Bundle*
