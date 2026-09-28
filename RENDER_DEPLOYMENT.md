# 🚀 Deploying Air Compressor Management on Render.com

This guide provides instructions to run the **Air Compressor Management** SCADA dashboard, OEE engine, and AI diagnostic system on **[Render.com](https://render.com)**.

---

## 🌟 What Runs on Render?

When deployed to Render:
- **Real-Time WebSockets**: Live telemetry streamed over secure WebSockets (`wss://<your-subdomain>.onrender.com/ws/telemetry`).
- **Digital Twin Simulator**: Realistic Atlas Copco GA-75 screw compressor physics (pressure cycles, thermal curves, idle power draw) runs autonomously in the background.
- **SQLite Database**: Automatically initialized and seeded with 24 hours of operational telemetry, hourly OEE history, and maintenance event logs.
- **AI Diagnostic Chatbot**: Built-in Industrial Reliability Heuristic Rule Engine evaluates live SQL telemetry and OEE scores without needing external GPU or Ollama servers.
- **Siemens S7-1200 Diagnostics (`/plc`)**: Full Data Block memory viewer, watchdog handshake monitor, and ISO-on-TCP socket probing.

---

## ⚡ Method 1: 1-Click Blueprint Deploy (Recommended)

This repository includes a pre-configured `render.yaml` Blueprint specification.

### Step 1: Push Code to Your GitHub Repository
Ensure the latest code is pushed to your GitHub repo:
```bash
git add .
git commit -m "Configure Render.com deployment"
git push origin main
```

### Step 2: Create a Blueprint Instance on Render
1. Log in to your **[Render Dashboard](https://dashboard.render.com/)**.
2. Click the **"New +"** button in the top navigation bar and select **"Blueprint"**.
3. Connect your GitHub account and select your repository (**`comp-render`**).
4. Render will automatically parse [render.yaml](file:///render.yaml) and display the service configuration:
   - **Service Name**: `comp-render`
   - **Runtime**: `Python`
   - **Plan**: `Free`
   - **Build Command**: `pip install --upgrade pip && pip install -r requirements.txt`
   - **Start Command**: `uvicorn run:app --host 0.0.0.0 --port $PORT`
   - **Health Check**: `/api/plc/status`
5. Click **"Apply"**. Render will build and deploy your service in ~1 minute!

---

## 🛠️ Method 2: Manual Web Service Setup

If you prefer to configure the Web Service manually via the Render UI:

### Step 1: Create a New Web Service
1. In your Render Dashboard, click **"New +"** $\rightarrow$ **"Web Service"**.
2. Select your GitHub repository (**`comp-render`**).

### Step 2: Configure Service Details
Fill in the following settings in the setup form:

| Setting | Value |
| :--- | :--- |
| **Name** | `comp-render` (or any custom name) |
| **Language / Runtime** | `Python 3` |
| **Region** | Select the region closest to you (e.g., *Oregon (US West)*, *Frankfurt (EU)*, *Singapore*) |
| **Branch** | `main` |
| **Build Command** | `pip install --upgrade pip && pip install -r requirements.txt` |
| **Start Command** | `uvicorn run:app --host 0.0.0.0 --port $PORT` |
| **Instance Type** | `Free` |

### Step 3: Add Environment Variables
Under the **Environment Variables** section, add the following key-value pairs:

| Key | Value | Note |
| :--- | :--- | :--- |
| `PYTHON_VERSION` | `3.11.9` | Ensures consistent Python 3.11+ runtime |
| `DB_TYPE` | `sqlite` | Uses built-in SQLite database |
| `PLC_ENABLED` | `false` | Enables built-in Digital Twin Simulator |
| `LLM_FALLBACK_TO_HEURISTIC` | `true` | Enables offline rule engine for chatbot |
| `APP_NAME` | `Air Compressor Management` | App display title |

### Step 4: Configure Health Check (Advanced Settings)
Under **Advanced**:
- **Health Check Path**: `/api/plc/status`
- Render pings this endpoint to confirm zero-downtime deployment before routing traffic.

### Step 5: Click "Create Web Service"
Render will pull your code, install dependencies, run database migrations, and deploy the application.


---

## 🔍 Verifying Your Deployment

Once Render displays **"Live"**, open your application URL (e.g., `https://comp-render.onrender.com`):

1. **SCADA Dashboard (`/`)**:
   - Status badge should show `SIMULATOR LIVE` in green.
   - Gauges, Canvas strip-chart, and power curves update live every 1.5 seconds via WebSocket (`wss://...`).
2. **Siemens S7-1200 Diagnostics (`/plc`)**:
   - Check real-time Data Block registers (`DB1.DBD4`, `DB1.DBD32`, etc.).
   - Verify watchdog counter handshaking (`DB1.DBW70` $\leftrightarrow$ `DB1.DBW72`).
3. **SQL Data Historian (`/data`)**:
   - Browse seeded telemetry, event logs, and hourly OEE records.
   - Export filtered telemetry to CSV.
4. **Interactive Swagger API Docs (`/docs`)**:
   - Inspect and test all REST endpoints directly in your browser.

---

## 💡 Render Platform Notes & Tips

### 1. Free Tier Sleep Behavior
- Render's Free tier spins down web services after **15 minutes of inactivity**.
- When a new visitor requests the URL, Render wakes the container up within **30–50 seconds**.
- Subsequent requests and WebSocket streaming will remain responsive and fast.

### 2. Persistent SQLite Data (Optional)
- By default on Render's Free tier, the local filesystem is ephemeral (resets on redeploy).
- Because `init_db()` automatically recreates and re-seeds historical data on boot, the app **never fails to start**.
- If you want SQLite records and telemetry to persist permanently across redeploys:
  1. Upgrade to a paid plan on Render.
  2. In the service settings, add a **Disk**:
     - **Mount Path**: `/app/data`
     - **Size**: 1 GB
  3. Set environment variable: `SQLITE_PATH=/app/data/compressor.db`.

### 3. Connecting to Remote Ollama or Cloud LLMs
- If you have an external Ollama instance accessible over the internet (or via a Cloudflare Tunnel), set:
  - `OLLAMA_BASE_URL=https://your-ollama-tunnel-domain.com`
  - `OLLAMA_MODEL=llama3.2`
- If unset or unreachable, the application automatically uses its built-in **Industrial Heuristic Rule Engine** with zero performance degradation.
