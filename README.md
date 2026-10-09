# Cartseen AI

AI grocery planner + price comparison + monthly budget manager for Indian customers.
**All prices are demo data** ("Demo prices — connect live data sources for real-time pricing"). No platform is scraped.

## Run it

```bash
npm install
cp .env.example .env.local        # then set NEXTAUTH_SECRET:  openssl rand -base64 32
npm run db:push                   # creates the local SQLite DB (prisma/dev.db)
npm run dev                       # http://localhost:3000
```

Mobile / email login works locally with a **development-only mock**: the one-time code is shown on screen. No SMS or email is sent.

| Script | Purpose |
|---|---|
| `npm test` | Unit tests for the engines (vitest) |
| `npm run test:e2e` | Browser tests (Playwright; starts the dev server on :3100) |
| `npm run typecheck` / `npm run lint` | Static checks |
| `npm run build` | Production build |

## Architecture

```
src/lib/data/      PriceProvider interface + MockProvider  ← the ONLY place prices come from
src/lib/engine/    pricing · compare · basketOptimizer · valueScore · planner · deliverySplit · adjust  (pure, tested)
src/lib/ai/        intent parsing (Claude tool-use, with rule-based fallback) + executor
src/app/api/       zod-validated route handlers
src/mock/          ~91 demo products × 5 stores, seeded offers (deterministic)
prisma/            user data only (accounts, saved plans, favourites, history)
```

* **Final cost** = items + delivery fee + platform fee + small-cart fee − best coupon.
* **Basket optimizer** enumerates every store subset (≤ 2 stores by default) and returns the exact cheapest total *including fees*.
* **Planner** sizes needs per person, picks variants by a value score (price, quality, availability, delivery, pack fit), cuts back in stages if over budget, and splits deliveries by shelf life.
* **Chat assistant**: the model only extracts *intent*; every number comes from the engines. Without `ANTHROPIC_API_KEY` a keyword parser handles the same requests.

## Connecting real price data

1. Implement `PriceProvider` (`src/lib/data/provider.ts`) against an **approved API or partner feed**.
2. Register it in `src/lib/data/registry.ts`.
3. Set `PRICE_PROVIDERS=yourProvider`.

Nothing else changes. Get legal/partner approval first; do not scrape.

## Before production

See [DECISIONS.md](DECISIONS.md) → "Open items".
