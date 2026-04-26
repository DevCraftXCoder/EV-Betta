# EV Betta

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Cloudflare Workers](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat&logo=cloudflare&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=black)
![Vite](https://img.shields.io/badge/Vite-646CFF?style=flat&logo=vite&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat&logo=nodedotjs&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

**Expected-value sports picks aggregator. Daily odds sync, EV calculation engine, and a real-time picks board.**

> A full-stack sports analytics pipeline — scraper to edge API to React UI. Aggregates picks from multiple sources, calculates expected value on each play, and presents a prioritized board updated daily. Built edge-first on Cloudflare Workers with a D1-backed picks store.

---

## Table of Contents

- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Features](#features)
- [Security](#security)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Recent Additions](#recent-additions)
- [API](#api)
- [Running This](#running-this)
- [Disclaimer](#disclaimer)

---

## Architecture

```
Scraper (Node.js · cron)                Odds Seed (cron)
  │                                        │
  │  POST /api/picks/sync                  │  POST /api/odds/seed
  ▼  (secret-authenticated)               ▼  (secret-authenticated)
TypeScript + Hono  (Cloudflare Workers)
  │
  ├── D1  (curated_picks · odds · sports metadata)
  ├── Cache API  (picks board · 5-min TTL)
  └── /api/picks/board  (public read · cached)
        │
        ▼
React Board UI  (Vite · deployed to Cloudflare Pages)
  │
  ├── BoardPage       (active date picks · sport filter · EV sort)
  ├── DetailPage      (pick breakdown · confidence · reasoning · secondary models)
  └── Date Navigator  (ET-aware calendar — handles midnight UTC/ET gap)
```

---

## Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| API | TypeScript + Hono | Cloudflare Workers edge runtime |
| Database | Cloudflare D1 (SQLite) | Picks store, odds, sports metadata |
| Cache | Cloudflare Cache API | 5-min TTL on board; invalidated on sync |
| Frontend | React + Vite | Deployed to Cloudflare Pages |
| Scraper | Node.js | Cron-scheduled, multiple source aggregation |
| Scheduling | PM2 cron expressions | ET-aware job scheduling |

---

## Features

### EV Calculation Engine
- Pulls implied probability from multiple sportsbooks
- Computes true probability from aggregate consensus
- EV = `(true_prob × payout) − (1 − true_prob)` — displayed as a percentage
- Picks sorted by EV descending; negative-EV plays surfaced but sorted below breakeven
- Per-sport EV breakdown with model confidence bands

### Pick Sources
- Action Network (XHR-based odds API)
- Oddsshark (DOM-scraped consensus lines)
- Multiple model-based pick sources (inference pipeline)
- Source attribution on every pick — no black box

### Supported Sports
- **NBA** — player props, game lines
- **MLB** — run lines, totals, player props
- **NHL** — puck lines, goal totals
- **NCAAB** — spread and totals
- **EPL** — match result and goals markets

### Secondary Model Outputs
- Win probability alongside EV on every pick
- Stat-type-aware AI reasoning (reasoning matches the prop's `stat_type`, not just the player)
- Injury-aware analysis pipeline: player availability factored into confidence scores
- Confidence bands calibrated per sport and bet type

### Board UI
- ET-aware date navigator
- Sport filter (NBA, MLB, NHL, NCAAB, EPL)
- EV badge, confidence bar, and book comparison on each pick
- Detail page with full reasoning, secondary model outputs, and historical accuracy
- Fallback to previous day's picks when current day is empty

### Data Pipeline
- **Odds seed** (06:00 ET daily): pulls raw lines from sportsbooks
- **Picks sync** (10:30 ET daily): runs scraper, calculates EV, writes to D1
- **Tomorrow pre-seed** (03:30 UTC daily): seeds next-day picks before midnight ET rollover
- **Injury reports** (daily cron): fetches player availability; feeds into model confidence
- Empty sync protection: sync with zero picks never triggers a DELETE

---

## Security

### Write Endpoint Authentication
All write endpoints (`/api/picks/sync`, `/api/odds/seed`) require a shared secret in the `Authorization` header — authenticated via timing-safe comparison. These endpoints are never called from the public UI.

### Empty Sync Guard
A `POST /api/picks/sync` with zero picks is a no-op — it never executes a `DELETE` against D1. This prevents the board from being wiped by a scraper failure or upstream source outage.

### Input Validation
All sync payloads validated with Zod before any D1 write. Malformed or out-of-range values are rejected with a `400` before reaching the database.

### Public Read Path
Board read endpoints are fully unauthenticated — no auth complexity on the hot path. Writes are isolated behind secret-key auth. The two paths are architecturally separate.

### No User PII
The picks board is read-only and public. No user accounts, no stored personal data, no session state.

### Cache Integrity
Cache is invalidated on successful sync writes only. The cache layer cannot be poisoned from the public read path.

---

## Key Engineering Decisions

### ET-aware date handling throughout
Sports schedules run on Eastern Time. All date keys use an ET locale formatter (`etDateKey()`) — not `toISOString().slice(0, 10)` (UTC). A naive UTC date in a cron at 10:30 ET produces the correct date; but the odds seed at midnight ET would produce tomorrow's UTC date, creating a 5.5-hour gap where the board shows no data.

### Empty sync protection
The sync endpoint checks `body.picks.length === 0` before any write. An empty payload from a scraper failure is silently ignored — the board retains the last valid sync.

### AI reasoning stat_type matching
Earlier versions matched AI reasoning to the player name only. The engine now routes each pick to a reasoning prompt that includes the exact `stat_type`, producing prop-specific analysis (rebounding reasoning for rebounds props, not generic player context).

### Cache API at the edge
The picks board response is cached at the Cloudflare edge with a 5-minute TTL. Under normal read traffic, D1 is never queried — the cache serves the response.

### Tomorrow pre-seed
A dedicated cron at 03:30 UTC generates next-day picks each night. When users open the board after midnight ET, picks are already available.

---

## Recent Additions

- Deploy-status pill on signup page — live board preview with Cloudflare deploy state indicator
- Admin Discord buttons on board root tab for Top-3 EV and daily summary notifications
- EV Betta signup page WebGL cleanup and input sanitization
- Matchup preview image on signup page replacing animated card swap

---

## API

### Public (no auth)
```http
GET /api/picks/board?date=2026-04-20&sport=NBA
GET /api/picks/:id
GET /health
```

### Write (secret-authenticated)
```http
POST /api/picks/sync
Authorization: Bearer <sync-secret>

{
  "date": "2026-04-20",
  "sport": "NBA",
  "picks": [
    {
      "team": "Boston Celtics",
      "line": "-4.5",
      "book": "DraftKings",
      "implied_prob": 0.62,
      "true_prob": 0.67,
      "ev": 0.083,
      "confidence": "high",
      "source": "action_network",
      "stat_type": "points",
      "win_probability": 0.67
    }
  ]
}
```

---

## Running This

```bash
# Worker API
cd ev-betta-worker
npm install
npx wrangler dev          # local dev
npx wrangler deploy       # deploy to CF Workers

# Scraper
cd ev-betta-scraper
npm install
npm run picks             # manual picks run
npm run odds-seed         # manual odds seed

# UI
cd ev-betta-ui
npm install
npm run dev               # Vite dev server
npm run build             # production build
```

See `.env.example` for required environment variables.

---

## Disclaimer

EV Betta is for **entertainment and informational purposes only.** Aggregated picks represent analysis from public sources — not financial, gambling, or investment advice. Always gamble responsibly and within legal jurisdictions.

---

## License

MIT — see [LICENSE](LICENSE)
