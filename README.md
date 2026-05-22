# EV Betta

![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white)
![Cloudflare Workers](https://img.shields.io/badge/Cloudflare_Workers-F38020?style=flat&logo=cloudflare&logoColor=white)
![React](https://img.shields.io/badge/React-61DAFB?style=flat&logo=react&logoColor=black)
![Hono](https://img.shields.io/badge/Hono-E36002?style=flat&logo=hono&logoColor=white)

**Expected-value sports picks aggregator — daily odds sync, EV engine, real-time board.**

> Scraper to edge API to React board: aggregates picks from 6 sources daily, calculates expected value per pick, and serves an ET-timezone-aware board via a Cloudflare Worker with 5-min Cache API TTL.

## Architecture

```
Scraper (Node.js, daily cron)
  └── Hono Worker API (Cloudflare Workers)
        └── D1 (picks store)
              └── React Board UI (Vite, Cloudflare Pages)
```

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Scraper | TypeScript + Node.js — 6 data sources |
| API | TypeScript + Hono — Cloudflare Worker |
| Database | Cloudflare D1 (SQLite) |
| UI | Vite + React — Cloudflare Pages |
| Process management | Scheduled cron jobs |

## Pipeline

1. **Odds seeder** — syncs daily odds from multiple sources (03:30 UTC + morning ET)
2. **Scraper** — aggregates picks from 6 sources daily at 09:00 ET
3. **EV engine** — calculates expected value per pick: `decimal_odds × hit_rate − 1`
4. **API** — serves picks board with 5-min Cache API TTL
5. **UI** — ET-aware date navigation, EV-sorted board, sport filter, primary/secondary pick split

## Key Engineering

- **ET timezone handling** — `etDateKey()` formatter prevents UTC/ET midnight gap (boards blank 0-5:30 AM ET)
- **Empty-picks guard** — sync with zero picks never triggers DELETE; protects against scraper downtime
- **Tomorrow pre-seed** — overnight cron seeds next day picks before midnight UI shows them
- **Primary/secondary split** — picks split by category into ranked tiers; display and notification logic uses guardedPicks array, never the raw display-capped array
- **Secret-authenticated** internal endpoints between scraper and Worker

## Disclaimer

> EV Betta is for entertainment and research purposes only. Expected value calculations are based on historical data and do not constitute financial advice or gambling recommendations.

