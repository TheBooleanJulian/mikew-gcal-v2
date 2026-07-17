<div align="center">

# mikew-gcal-v2

**Scrapes a busker's performance schedule daily and syncs it automatically to Google Calendar.**

![Python](https://img.shields.io/badge/-Python-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/-Flask-000000?logo=flask&logoColor=white)
![Redis](https://img.shields.io/badge/-Redis-DC382D?logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/-Docker-2496ED?logo=docker&logoColor=white)
![Zeabur](https://img.shields.io/badge/-Zeabur-6C5CE7)
![License](https://img.shields.io/badge/license-MIT-00D4C8.svg)

</div>

---

## What it does

Scrapes a busker's profile page (JavaScript-rendered) daily at 11 PM Singapore time (GMT+8) using Playwright, then creates the upcoming performance events in a configured Google Calendar. Redis is used for hash-based deduplication, so events are never double-created even after container restarts or redeployments. A lightweight Flask API and HTML status dashboard let you monitor state and trigger manual runs without touching the server.

## Features

- **Playwright scraping** — handles JavaScript-rendered schedule content; falls back to `requests`/BeautifulSoup if Playwright fails
- **Duplicate prevention** — hash-based deduplication stored in Redis, persists across redeployments
- **Distributed locking** — prevents multiple container instances running the job simultaneously
- **Daily cron scheduling** — APScheduler fires at 11 PM SGT; optional sync/reconciliation job to reconcile Redis with Google Calendar
- **Retry logic** — exponential backoff on scraping and Google Calendar API failures
- **Metrics tracking** — scrape count, events created, and error counts stored in Redis
- **Status dashboard** — `status.html` web UI for monitoring state, viewing scraped data, and triggering manual operations
- **Flexible credentials** — service account JSON via file path (local) or `GOOGLE_CREDENTIALS_JSON` env var (Zeabur deployment)
- **Graceful shutdown** — handles container stop signals cleanly

## Tech Stack

| Layer | Choice |
|---|---|
| Backend | Flask + APScheduler |
| Scraping | Playwright (Chromium) + BeautifulSoup fallback |
| Storage | Redis |
| Calendar | Google Calendar API v3 (service account) |
| Frontend | Single-file HTML (`status.html`) |
| Hosting | Zeabur (Docker, GitHub CI/CD) |

## Quick Start

```bash
git clone <repo>
cd mikew-gcal-v2
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
playwright install chromium
cp .env.example .env
# Edit .env with your values (see Configuration below)
python main.py
```

For local development, place your Google service account JSON at `./credentials/service-account.json`.

## Configuration

| Variable | Required | Description |
|---|---|---|
| `BUSKER_URL` | Yes | URL of the busker profile page to scrape |
| `CALENDAR_ID` | Yes | Google Calendar ID to write events into |
| `GOOGLE_CREDENTIALS_PATH` | Local only | Path to service account JSON file (default: `./credentials/service-account.json`) |
| `GOOGLE_CREDENTIALS_JSON` | Deployment | Full service account JSON content (use instead of file path on Zeabur) |
| `REDIS_HOST` | Yes | Redis host (default: `localhost`) |
| `REDIS_PORT` | Yes | Redis port (default: `6379`) |
| `REDIS_PASSWORD` | No | Redis password if auth is enabled |
| `TIMEZONE` | No | Timezone for scheduling (default: `Asia/Singapore`) |
| `LOG_LEVEL` | No | Logging verbosity (default: `INFO`) |
| `EVENT_TTL_DAYS` | No | Days to retain event hashes in Redis (default: `90`) |
| `PORT` | No | Port for the Flask API (default: `8080`) |

> **Note:** `BUSKER_URL` and `CALENDAR_ID` should be treated as sensitive — do not commit them to version control.

## Project Structure

```
mikew-gcal-v2/
|-- main.py              # Entry point: starts scheduler + Flask API thread
|-- api.py               # Flask API (health check, manual trigger endpoints)
|-- scheduler.py         # APScheduler cron job setup
|-- scraper.py           # Playwright scraper with requests fallback
|-- calendar_manager.py  # Google Calendar API integration
|-- redis_manager.py     # Redis connection, deduplication, locking, metrics
|-- sync_manager.py      # Reconciliation logic (Redis <-> Google Calendar)
|-- config.py            # Config loading and validation
|-- utils.py             # Logging helpers
|-- status.html          # Status dashboard frontend
|-- credentials/         # Local service account JSON (gitignored)
|-- Dockerfile
|-- requirements.txt
`-- .env.example
```

## Google Calendar Setup

1. Create a Google Cloud project and enable the Google Calendar API
2. Create a service account with Calendar scope and download its JSON key
3. Share the target calendar with the service account email (grant "Make changes to events")
4. Copy the calendar ID from calendar settings → use as `CALENDAR_ID`
5. For local use: place JSON at `./credentials/service-account.json`
6. For Zeabur: set `GOOGLE_CREDENTIALS_JSON` to the full JSON content as an env var

## Deployment

Deployed on Zeabur via Docker. Push to `main` triggers a build and deploy. Set all required environment variables (including `GOOGLE_CREDENTIALS_JSON`) in the Zeabur dashboard — no file mounts needed.

## Status / Roadmap

- [x] Playwright scraping with requests fallback
- [x] Redis deduplication and distributed locking
- [x] Google Calendar event creation
- [x] APScheduler daily cron (11 PM SGT)
- [x] Flask API with health check and manual trigger
- [x] Status dashboard (`status.html`)
- [x] `GOOGLE_CREDENTIALS_JSON` support for Zeabur deployment
- [ ] Multi-busker support

## Changelog

- **Jan 2026** — Fixed Google Calendar auth on deployed container; stabilised Playwright browser detection across system and Playwright-managed installs; added symlinks in Dockerfile for reliable Chromium resolution
- **Jan 2026 (early)** — Added `requests`/BeautifulSoup fallback scraping path when Playwright fails; multiple Playwright launch fallback strategies; disk usage optimisation in Docker image
- **Jan 2 2026** — Added `status.html` dashboard with formatted scrape data display and copy functionality; added health check and manual trigger API endpoints; added `GOOGLE_CREDENTIALS_JSON` env var support for credential-free deployment
- **Jan 1 2026** — Improved event extraction and schedule parsing with better selector matching and fallback logic; added `pytz` dependency; hid sensitive env vars (`BUSKER_URL`, `CALENDAR_ID`); added manual testing scripts

## License

MIT

---

<div align="center">
<sub>Built by <a href="https://github.com/TheBooleanJulian">@TheBooleanJulian</a></sub>
</div>