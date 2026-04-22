# EV Betta

**Expected-value sports picks aggregator. Daily odds sync, EV calculation engine, and a real-time board UI.**

> A full-stack sports analytics tool that aggregates picks from multiple sources, calculates expected value on each bet, and presents a prioritized board updated daily. Built edge-first on Cloudflare Workers with a D1-backed picks store and a React UI.

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

| Layer | Technology |
|---|---|
| API | TypeScript + Hono, Cloudflare Workers |
| Database | Cloudflare D1 (SQLite) |
| Cache | Cloudflare Cache API (5-min TTL) |
| Frontend | React + Vite, Cloudflare Pages |
| Scraper | Node.js, scheduled cron |
| Odds Source | Multiple sportsbook APIs + DOM scraping |
| Scheduling | ET-aware cron jobs |

---

## Features

### EV Calculation Engine
- Pulls implied probability from multiple books
- Computes true probability from aggregate consensus
- EV = (true_prob × payout) − (1 − true_prob) — displayed as a percentage
- Picks sorted by EV descending; negative-EV picks filtered by default
- Per-sport EV breakdown section with model confidence bands

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
- **EPL** — match result and goals markets (added 2026-04)

### Secondary Model Outputs
- Win probability displayed alongside EV on each pick
- Per-stat-type AI reasoning (AI reasoning matches the prop's stat_type — not just the player)
- Injury-aware analysis pipeline: player availability factored into confidence scores
- Confidence bands calibrated per sport and bet type

### Board UI
- Date navigator with ET-aware calendar
- Sport filter (NBA, MLB, NHL, NCAAB, EPL)
- EV badge, confidence bar, and book comparison on each pick
- Detail page with full reasoning, secondary model outputs, and historical accuracy
- Fallback to previous day's picks when current day is empty (handles the midnight-UTC-to-10:30am-ET gap)

### Data Pipeline
- **Odds seed** (runs daily at 06:00 ET): pulls raw lines from sportsbooks
- **Picks sync** (runs daily at 10:30 ET): runs scraper, calculates EV, writes to D1
- **Tomorrow pre-seed** (runs daily at 03:30 UTC): seeds next-day picks before midnight ET rollover
- **Injury reports** (daily cron): fetches player availability from official sources; feeds into model confidence
- Empty sync protection: `POST /api/picks/sync` with zero picks never triggers a DELETE — prevents silent wipe when a source is down

---

## Key Engineering Decisions

### ET-aware date handling throughout
Sports schedules run on Eastern Time. All date keys use an ET locale formatter (`etDateKey()`) — not `toISOString().slice(0, 10)` (UTC). A naive UTC date in a cron at 10:30 ET would produce the correct date; but the odds seed at midnight ET would produce tomorrow's UTC date, creating a 5.5-hour gap where the board shows no data.

### Empty sync protection
The sync endpoint checks `body.picks.length === 0` before executing any DELETE. An empty payload from a scraper failure or source outage is silently ignored — the board retains the last valid sync. This prevents the pattern where a failed scrape wipes the board clean.

### AI reasoning stat_type matching
Earlier versions matched AI reasoning to the player name only — a player with both points and assists props would get the same reasoning block for both. The engine now routes each pick to a reasoning prompt that includes the exact `stat_type`, producing prop-specific analysis (e.g., rebounding reasoning for a rebounds prop, not generic player context).

### Cache API at the edge
The picks board response is cached at the Cloudflare edge with a 5-minute TTL. Under normal read traffic, D1 is never queried for the board — the cache serves the response. Cache is invalidated on successful sync writes.

### Secret-authenticated write endpoints
All write endpoints (`/api/picks/sync`, `/api/odds/seed`) require a `X-Sync-Secret` header matched via timing-safe comparison. Public read endpoints are unauthenticated. This eliminates auth complexity for the read path while protecting writes.

---

## Recent Additions

- **Win probability on every pick** — secondary model outputs now include win probability alongside EV
- **EPL support** — English Premier League markets added (replaces NFL off-season slot)
- **Stat-type-aware AI reasoning** — AI analysis now matches the prop's stat_type, not just the player
- **Injury reports pipeline** — player availability fetched daily, factored into pick confidence
- **Per-sport EV breakdown** — detail page shows EV calculation broken down by sport + bet type

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
      "source": "action_network",
      "stat_type": "points",
      "win_probability": 0.67
    }
  ]
}
```

---

## Security

- Write endpoints authenticated via `X-Sync-Secret` header (timing-safe comparison)
- No admin credentials exposed in any public-facing route
- D1 queries use parameterized statements — no string concatenation
- Cache invalidation on write — stale data never served after a successful sync

---

## Disclaimer

This tool is for entertainment and informational purposes only. Not financial advice. Always gamble responsibly.

---

## License

MIT — see [LICENSE](LICENSE)
