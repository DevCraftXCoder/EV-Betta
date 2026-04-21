# Security Policy

## Reporting a Vulnerability

Do **not** open a public GitHub issue for security vulnerabilities.

Contact via GitHub: [@DevCraftXCoder](https://github.com/DevCraftXCoder)

---

## Security Architecture

### Write Endpoint Authentication

All data write endpoints are secret-authenticated:

- `POST /api/picks/sync` — requires `X-Sync-Secret` header
- `POST /api/odds/seed` — requires `X-Sync-Secret` header
- Secret validated via `crypto.subtle.timingSafeEqual` — prevents timing attacks on the header comparison
- Requests with missing or incorrect secret return `401` with no additional detail

### Public Read Endpoints

- Board read endpoints are unauthenticated — public read is intentional
- No write operations accessible without the sync secret
- No admin functions accessible via public routes

### Data Integrity

- Empty sync protection: `body.picks.length === 0` check prevents silent data wipe when a source is down
- All D1 writes use parameterized queries — no SQL injection vector
- Pick data validated with Zod before writing to D1

### Infrastructure Security

- Scraper subprocess spawn calls use array form with `--` separator (prevents argument injection on URL inputs)
- No environment secrets in scraper output or logs
- Sync secret stored in environment variables only — never in code or config files committed to version control
- Cloudflare Cache API used for public board responses — D1 is not queried on every public request

### No PII Handled

- No user accounts, no authentication for read access
- No personal data collected or stored
- Analytics are aggregate (pick counts, EV summaries) — no individual user tracking
