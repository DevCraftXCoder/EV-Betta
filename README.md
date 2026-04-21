# EV Betta

**Expected-value sports picks aggregator. Daily odds sync, EV calculation engine, and a real-time board UI.**

> A full-stack sports analytics tool that aggregates picks from multiple sources, calculates expected value on each bet, and presents a prioritized board updated daily. Built edge-first on Cloudflare Workers with a D1-backed picks store and a React UI.

---

## Architecture

```
Scraper (Node.js · PM2 · cron)          Odds Seed (PM2 · cron)
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
  ├── BoardPage  (active date picks · sport filter · EV sort)
  ├── DetailPage  (pick breakdown · confidence · reasoning)
  └── Date Navigator  (ET-aware calendar — handles midnight UTC/ET gap)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| API | TypeScript + Hono, Cloudflare Workers |
| Database | Cloudflare D1 (SQLite) |
| Cache | Cloudflare Cache API (5-min TTL) |
| Frontend | React + Vite, Cloudflare Pages |
| Scraper | Node.js, PM2 |
| Odds Source | Multiple sportsbook APIs + DOM scraping |
| Scheduling | PM2 cron (ET-aware) |

---

## Features

### EV Calculation Engine
- Pulls implied probability from multiple books
- Computes true probability from aggregate consensus
- EV = (true_prob × payout) − (1 − true_prob) — displayed as a percentage
- Picks sorted by EV descending; negative-EV picks filtered by default

### Pick Sources
- Action Network (XHR-based odds API)
- Oddsshark (DOM-scraped consensus lines)
- Multiple model-based pick sources (Ollama inference)
- Source attribution on every pick — no black box

### Board UI
- Date navigator with ET-aware calendar
- Sport filter (NBA, NFL, MLB, NHL, NCAAB)
- EV badge, confidence bar, and book comparison on each pick
- Detail page with full reasoning and historical accuracy for the pick type
- Fallback to previous day's picks when current day is empty (handles the midnight-UTC-to-10:30am-ET gap)

### Data Pipeline
- **Odds seed** (runs daily at 06:00 ET): pulls raw lines from sportsbooks
- **Picks sync** (runs daily at 10:30 ET): runs scraper, calculates EV, writes to D1
- **Tomorrow pre-seed** (runs daily at 03:30 UTC): seeds next-day picks before midnight ET rollover
- Empty sync protection: `POST /api/picks/sync` with zero picks never triggers a DELETE — prevents silent wipe when a source is down

---

## Key Engineering Decisions

### ET-aware date handling throughout
Sports schedules run on Eastern Time. All date keys use an ET locale formatter (`etDateKey()`) — not `toISOString().slice(0, 10)` (UTC). A naive UTC date in a cron at 10:30 ET would produce the correct date; but the odds seed at midnight ET would produce tomorrow's UTC date, creating a 5.5-hour gap where the board shows no data.

### Empty sync protection
The sync endpoint checks `body.picks.length === 0` before executing any DELETE. An empty payload from a scraper failure or source outage is silently ignored — the board retains the last valid sync. This prevents the pattern where a failed scrape wipes the board clean.

### Cache API at the edge
The picks board response is cached at the Cloudflare edge with a 5-minute TTL. Under normal read traffic, D1 is never queried for the board — the cache serves the response. Cache is invalidated on successful sync writes.

### Secret-authenticated write endpoints
All write endpoints (`/api/picks/sync`, `/api/odds/seed`) require a `X-Sync-Secret` header matched via timing-safe comparison. Public read endpoints are unauthenticated. This eliminates auth complexity for the read path while protecting writes.

---

## API

### Public (no auth)
```http
GET /api/picks/board?date=2026-04-20&sport=NBA
GET /api/picks/:id
GET /api/health
```

### Write (secret-authenticated)
```http
POST /api/picks/sync
X-Sync-Secret: <secret>

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
      "source": "action_network"
    }
  ]
}
```

---

## Security

- Write endpoints authenticated via `X-Sync-Secret` header (timing-safe comparison)
- No admin credentials exposed in any public-facing route
- D1 queries use parameterized statements — no string concatenation
- Scraper spawn calls use array form with `--` separator (prevents argument injection)
- Cache invalidation on write — stale data never served after a successful sync

---

## License

MIT — see [LICENSE](LICENSE)
