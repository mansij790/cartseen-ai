# SmartGrocery AI — Project Synopsis

## 1. Introduction
SmartGrocery AI is a responsive web application for grocery shoppers in India. It combines an AI-assisted monthly grocery planner, a multi-store price comparison tool and a budget/savings manager. The user states a monthly budget, household size and preferences; the app produces a budget-fit grocery list, splits it across deliveries, and finds the cheapest way to buy it across quick-commerce platforms (Blinkit, Zepto, Swiggy Instamart, BigBasket, Flipkart Minutes).

> All prices in the current version are demo data. No platform is scraped, and every price page shows a "Demo prices" banner.

## 2. Problem Statement
Households plan monthly groceries by guesswork and usually overspend. Prices, delivery fees, platform fees, small-cart charges and coupons differ across apps, so the lowest sticker price is often not the lowest final cost. There is no single tool that plans purchases within a budget, schedules them by shelf life and compares total cost across stores.

## 3. Objectives
- Generate a monthly grocery plan that stays within a user-defined budget.
- Split purchases across 1–6 deliveries according to shelf life (staples first, fresh items spread out).
- Compare single products and baskets (2–10 items) across stores using the true final cost: items + delivery + platform + small-cart fees − best coupon.
- Find the exact cheapest store combination for a basket.
- Let users adjust plans through chat ("reduce by ₹200", "replace with cheaper", "remove rice").
- Show budget usage and savings, and persist plans, favourites and history for logged-in users.

## 4. Scope and Key Modules
| Module | Description |
|---|---|
| AI Grocery Planner | Multi-step form (basics, preferences, items), live budget slider, category allocation, budget bands (Basic / Standard / Premium), delivery cards |
| Price Comparison | Search with filters and sorting; single-product and basket modes; "Lowest Price" and "Fastest Delivery" flags; "Price unavailable" handling |
| Basket Optimizer | Enumerates every store subset (default max 2 stores) and returns the exact cheapest total including fees |
| My Plan | Delivery timeline, per-item edit/remove/swap, shopping-list export |
| Savings Dashboard | Budget meter, planned vs allocated spend, under/over-budget message |
| Chat Assistant | Natural-language plan adjustments; the LLM only extracts intent and all numbers come from deterministic engines; a keyword-parser fallback works without an API key |
| Auth & Profile | Mobile/email one-time-code login, Google login, saved plans, favourites, search history, preferred stores |

## 5. Technology Stack
- **Frontend/Backend:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4
- **State & validation:** zustand, zod
- **Database:** Prisma 6 with SQLite (user data only; Postgres-ready)
- **Auth:** Auth.js (next-auth v4) with Prisma adapter; HMAC-hashed OTPs
- **AI:** Anthropic SDK (tool-use intent parsing) with rule-based fallback
- **Testing:** Vitest (engine unit tests), Playwright (e2e)

## 6. System Architecture
- `src/lib/data` — PriceProvider interface and MockProvider; the only source of prices, so real feeds can be plugged in later without other changes.
- `src/lib/engine` — pure, tested engines: pricing, compare, basketOptimizer, valueScore, planner, deliverySplit, adjust.
- `src/lib/ai` — intent parser, rule parser and executor.
- `src/app/api` — zod-validated route handlers (plan, compare, chat, user data).
- `src/mock` — about 91 demo products across 5 stores with deterministic seeded offers.
- `prisma` — accounts, saved plans, favourites, search history, preferred stores.

## 7. Methodology
1. Needs are computed per person per category, scaled by household size and number of days.
2. The budget is allocated across categories and quality bands.
3. Each product variant is chosen by a value score (price per unit, quality tier, availability, delivery time, pack-size fit).
4. A fit loop downgrades, trims or drops lower-priority items in stages until the plan fits the budget; essentials are never cut below 70%. If it still does not fit, an explicit over-budget result with suggestions is returned.
5. Items are scheduled across deliveries by shelf life, and each delivery is optimized across stores.

## 8. Security and Quality Measures
- Every API input is validated with zod; plan totals are recomputed server-side.
- All user-data routes are scoped to the session user (protection against IDOR).
- OTP: hashed storage, 5-minute expiry, 5 attempts, resend cooldown, rate limits; the mock sender is refused in production.
- No secrets in code; no PII or full chat messages logged.
- Automated unit and end-to-end tests.

## 9. Software Requirements
- Node.js and npm; a modern web browser.
- Optional: `ANTHROPIC_API_KEY` for LLM-based chat (the app works without it).

## 10. Expected Outcome
A working MVP that lets a user enter a budget and get a complete, budget-fit, delivery-scheduled grocery plan, compare true final costs across stores, and track savings, with a clear path to live data.

## 11. Limitations and Future Scope
- Prices are demo data; real feeds require approved APIs or partner agreements and legal sign-off.
- A real SMS/email OTP vendor, Google login credentials and the live LLM path still need setup and verification.
- The rate limiter is in-memory; production needs Redis and a trusted-proxy setup.
- Future work: Amazon Fresh/JioMart, price history and alerts, nutrition-based and seasonal planning, deep links, and production deployment via feature branch, peer-reviewed PR, CI and staging QA.
