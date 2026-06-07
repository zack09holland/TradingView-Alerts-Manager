# TradingView Alerts Tool 

Automatically add, remove, and monitor custom TradingView alerts in bulk, with a full-featured real-time dashboard, text-to-speech alert announcements, web scraping, news, charting, and market data integrations.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Installation & Setup](#installation--setup)
- [Configuration (config.yml)](#configuration-configyml)
- [CLI Usage](#cli-usage)
- [Dashboard UI](#dashboard-ui)
- [Alert Management](#alert-management)
- [Web Scraping](#web-scraping)
- [Integrations](#integrations)
- [API Reference](#api-reference)
- [Docker Deployment](#docker-deployment)
- [Environment Variables](#environment-variables)

---

## Overview

This tool uses Puppeteer-controlled Chromium to automate TradingView alert creation and deletion in bulk. It also runs a backend API server + React dashboard for real-time webhook alert monitoring, active alert management, HOD stock scanning, news, charting, and more.

**Key capabilities:**
- Bulk create TradingView alerts from a CSV of symbols, across multiple indicators and intervals
- Remove all alerts for a symbol, or target a specific individual alert
- Dashboard to receive and display live TradingView webhook alerts in real-time
- Scrape active alerts from the TradingView UI and manage them from the dashboard
- Scan High-of-Day stocks from MomoScreener and cross-reference with your active alerts
- Fetch financial news via Benzinga
- Stream real-time quotes and OHLC bars via TradeStation
- Text-to-speech alert announcements (Google Cloud TTS / Edge TTS)
- Generic XPath-based web scraping for any site

---

## Requirements

- [Node.js 22+](https://nodejs.org/en/)
- A TradingView account (free or paid)
- `.env` file configured (see [Environment Variables](#environment-variables))
- `config.yml` configured (see [Configuration](#configuration-configyml))

---

## Installation & Setup

```bash
# Install dependencies
npm install

# Build TypeScript
npm run build

# Start the backend server (port 3000)
npm start

# Start the frontend dev server (port 5173)
cd frontend && npm install && npm run dev
```

> The backend must be running for the dashboard and all API features to work.

To run with the browser **visible** (non-headless) for debugging:

```bash
# In .env
HEADLESS=false
```

---

## Configuration (config.yml)

All alert automation is driven by `config.yml` in the project root.

```yaml
files:
  input: stock_symbols.csv       # CSV with symbols to add alerts for
  exclude: exclude.csv           # (optional) symbols to skip

tradingview:
  chartUrl: https://www.tradingview.com/chart/YOUR_CHART_ID/
  username: your@email.com       # TradingView login email
  password: yourpassword         # TradingView password

alerts:
  - name: MA crossover           # Human-readable name for this alert config
    active: true                 # Set to false to skip this indicator during bulk add
    intervals:
      - 1m
      - 2m
    condition:
      primaryLeft: "MA crossover"          # Indicator name as it appears in TradingView dropdown
      primaryRight: null
      secondary: Any alert() function call  # Condition (use "Any alert() function call" for script alerts)
    actions:
      notifyOnApp: false
      showPopup: false
      sendEmail: false
      webhook:
        enabled: true
        url: https://your-webhook-url.com/tradingview-alert

  - name: Ultimate RSI MTF
    active: false                # Inactive â€” skipped during bulk runs
    intervals:
      - 5m
      - 2m
      - 30s
    condition:
      primaryLeft: "Ultimate RSI MTF"
      secondary: Any alert() function call
    actions:
      webhook:
        enabled: true
        url: https://your-webhook-url.com/tradingview-alert
```

### Multiple Alerts Per Run

You can define as many alert templates as you like. Only those with `active: true` will be used during a bulk add run. Each template can have its own:
- **Indicator** (`primaryLeft`) â€” must exist on the chart at `chartUrl`
- **Intervals** â€” e.g. `[1m, 2m, 5m]`; an alert is created for each interval
- **Condition** â€” primary, secondary, tertiary conditions
- **Actions** â€” webhook URL, email, popup, app notification

### Indicator Name Matching

Indicator names use **exact text matching** (after typing the full name to filter the dropdown). If two indicators share a prefix (e.g. `@MA Crossover` vs `@MACD Alerts`), the full name is typed before selecting to avoid mis-selection.

You can also use **regular expressions** for more precise matching:

```yaml
condition:
  primaryLeft: /^MA crossover$/
```

### Token Replacement in Alert Names/Messages

Use `{{column_name}}` in alert names or messages to substitute values from your CSV:

```yaml
condition:
  primaryLeft: "DSMA ({{DSMAsetting}}, 50)"
```

---

## CLI Usage

### Add Alerts from CSV (Batch Mode)

```bash
npm run legacy
# or
./tvalerts add-alerts csv
./tvalerts add-alerts csv config.btc.yml   # use a different config file
./tvalerts add-alerts csv -d 2000          # add 2000ms delay between symbols
```

Reads symbols from the CSV at `files.input`, navigates TradingView to each, and creates alerts for all `active: true` indicators at all configured intervals.

### Interactive CLI Mode

```bash
npm run cli
# or
./tvalerts add-alerts cli
```

Launches an interactive prompt. You can type symbol names one at a time to add or remove alerts without restarting the script.

### Global Options

| Flag | Description |
|---|---|
| `-l, --loglevel <level>` | Log verbosity 1â€“5 (default: 3) |
| `-d, --delay <ms>` | Base delay between operations in ms |

---

## Dashboard UI

The React frontend (port 5173) provides a tabbed interface with the following pages:

### Live Alerts
Real-time table of all TradingView webhook alerts received. Displays:
- Symbol, price, exchange, volume
- Timeframe/interval
- Condition/indicator name
- Timestamp
- Alert action type

Alerts are pushed instantly via WebSocket (Socket.IO).

### Active Alerts Manager
View and manage alerts currently set in TradingView:
- Lists all active alerts scraped from the TradingView UI, grouped by symbol
- Add alerts for a new symbol directly from the dashboard
- Remove all alerts for a symbol
- Remove a specific individual alert by name
- Auto-refreshes every 30 seconds (uses local cache to avoid unnecessary scraping)

### HOD Charts
High-of-Day stock scanner:
- Scrapes HOD and Momentum stock lists from [MomoScreener](https://momoscreener.com/scanner)
- Displays symbol, price, % change, float, volume, SPR, time
- Highlights symbols that have active TradingView alerts
- Embedded mini-charts for each HOD stock
- Auto-refreshes every 30 seconds

### Charts Watcher
Monitor multiple TradingView charts in a grid layout simultaneously.

### Web Scraping
Manual web scraping interface:
- Enter any URL and XPath selectors to extract data
- Navigate to a site and perform manual login, then scrape
- Table scraping â€” extract HTML tables as structured data
- Health check / status display for the scraping browser session

### Alert Config Panel
Configure and trigger alert creation from the UI with indicator and interval selection.

### Stats Panel
Aggregate statistics about received webhook alerts.

---

## Alert Management

### How Bulk Alert Creation Works

1. Reads symbols from the CSV file
2. Navigates the Puppeteer browser to each symbol on your configured chart
3. Opens the New Alert dialog
4. For each `active: true` indicator in `config.yml`:
   - Selects the indicator from the dropdown by typing the full name for exact matching
   - Sets the secondary condition
   - Configures webhook URL and actions
5. Saves the alert, repeats for each interval
6. Moves to next symbol

### Invalid Symbol Handling

If TradingView reports a symbol as invalid, the tool logs the error and skips to the next symbol without crashing.

### Removing Alerts

**Remove all alerts for a symbol:**
```
DELETE /api/alerts/remove/:symbol
```
Scrapes the alerts panel, finds all alerts matching the symbol, and deletes them one by one.

**Remove a specific alert by ID:**
```
DELETE /api/alerts/remove-specific
Body: { alertId, alertIndex, symbol, alertName }
```
Targets a precise alert by its position in the panel (from prior scraping), verifies the symbol matches before deleting.

### Modal Handling

A blocking modal check runs automatically before every alert panel interaction (open, scrape, delete). If a promotional or upgrade popup is present, the close button is clicked before proceeding. Falls back to `Escape` if the button is not found.

---

## Web Scraping

The backend includes a generic scraping service powered by Puppeteer.

### Endpoints

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/scrape` | Scrape a single XPath from a URL |
| POST | `/api/scrape/multi` | Scrape multiple XPaths in one page load |
| POST | `/api/scrape/table` | Extract an HTML table as headers + rows |
| POST | `/api/scrape/navigate` | Navigate to URL for manual login |
| POST | `/api/scrape/login-custom` | Auto-login with custom XPath selectors |
| GET | `/api/scrape/status` | Current browser auth status |
| GET | `/api/scrape/health` | Scraping service health check |

### Navigation Timeout Behaviour

Sites like TradingView and MomoScreener maintain persistent WebSocket connections for real-time data. The scraper uses `waitUntil: 'load'` instead of `networkidle2` to avoid navigation timeouts on these sites.

---

## Integrations

### TradeStation

Real-time market data streaming via the TradeStation API:
- OHLC bars via HTTP streaming at configurable intervals
- Live quote streaming via WebSocket
- Historical bar data retrieval
- Access token management with automatic refresh

Configure in `.env`:
```
TRADESTATION_API_KEY=...
TRADESTATION_API_SECRET=...
TRADESTATION_REFRESH_TOKEN=...
TRADESTATION_REDIRECT_URI=http://localhost:3000
```

### Benzinga News

Fetch financial news via the Benzinga API:
- `GET /api/news/:symbol` â€” news for a specific ticker
- `GET /api/news` â€” latest market news
- `POST /api/news/search` â€” search by date range, channels, page size

Configure in `.env`:
```
BENZINGA_API_KEY=...
```

### TradingView Webhooks

Point your TradingView alert webhook URL at:
```
https://your-server/tradingview-alert
```

Incoming alerts are parsed, stored in SQLite, broadcast to all connected dashboard clients via Socket.IO, and optionally announced via text-to-speech.

### Text-to-Speech (TTS)

Alert announcements are spoken aloud when a webhook alert is received. Supported engines:
- **Google Cloud TTS** â€” configure via service account JSON credentials
- **Edge TTS** â€” no API key required

### WebSockets (Socket.IO)

- Real-time push of incoming alerts to all connected dashboard tabs
- Tab visibility tracking â€” detects active vs backgrounded tabs
- Client heartbeat monitoring
- Connection statistics at `GET /ws-status`

---

## API Reference

### Alerts

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/alerts` | Historical alerts from DB (`?limit=100&symbol=AAPL`) |
| GET | `/api/alerts/active` | Scrape active alerts from TradingView UI |
| POST | `/api/alerts/add` | Add alerts for a symbol |
| DELETE | `/api/alerts/remove/:symbol` | Remove all alerts for a symbol |
| DELETE | `/api/alerts/remove-specific` | Remove one specific alert by ID |

### News

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/news/:symbol` | News for a ticker |
| GET | `/api/news` | Latest market news |
| POST | `/api/news/search` | Search news (dateFrom, dateTo, channels) |

### Scraping

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/scrape` | Single XPath scrape |
| POST | `/api/scrape/multi` | Multi-XPath scrape |
| POST | `/api/scrape/table` | HTML table scrape |
| GET | `/api/scrape/health` | Scraper health |

### System

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Backend health + WebSocket stats |
| GET | `/ws-status` | WebSocket connection stats |
| POST | `/tradingview-alert` | Receive TradingView webhook alerts |

---

## Docker Deployment

A `Dockerfile` is included for containerized deployment:

```bash
docker build -t tradingview-alerts .
docker run -p 3000:3000 --env-file .env tradingview-alerts
```

The image is based on `node:22.11-bookworm-slim` and includes all Chromium/Puppeteer dependencies and font packages.

For nginx reverse proxy configuration, see `nginx/nginx.conf`.

---

## Environment Variables

Create a `.env` file in the project root:

```dotenv
# Server
PORT=3000

# Frontend (CORS)
FRONTEND_URL=http://localhost:5173

# Benzinga news API
BENZINGA_API_KEY=your_key_here

# TradeStation API
TRADESTATION_API_KEY=your_key_here
TRADESTATION_API_SECRET=your_secret_here
TRADESTATION_REFRESH_TOKEN=your_token_here
TRADESTATION_REDIRECT_URI=http://localhost:3000

# Run Puppeteer browser in headless mode (true/false)
HEADLESS=true
```

Set `HEADLESS=false` to see the browser window during automation â€” useful for debugging alert creation or diagnosing TradingView UI changes.
