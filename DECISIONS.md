# Decisions, assumptions and open items

## Choices made during the build
* **Stack**: Next.js 16 (App Router) + TypeScript, Tailwind v4, zustand, zod, Prisma 6 + SQLite, Auth.js **v4**. `cacheComponents`/`partialPrefetching` from the scaffold were turned off (not needed).
* **Auth.js v4, not v5**: v5 is still a beta. Chosen by the project owner.
* **Prisma 6.19 pinned**: Prisma's `latest` tag points at 7.x/8 RC; 6.x is the stable line with simple SQLite support.
* **No shadcn/ui, no recharts**: small hand-written Tailwind primitives; charts are plain HTML/CSS bars (single series + target tick, with a table view).
* **Catalog is not in the DB**: products/prices come from the `PriceProvider`. Prisma holds user data only.
* **`family` field on `Product`** (e.g. "rice"): lets the planner substitute interchangeable products.
* **Adjustments replay the plan** (`input.pinned`): "remove rice" keeps everything else as-is instead of re-planning and re-spending the saving.
* **Allocation by category is advisory.** Real prices decide the final split; the UI shows allocated vs planned and says so.
* **Budget bands** use budget per person per 30 days: < ₹1,250 basic, < ₹2,000 standard, else premium.
* **`Buy Now`** opens the store *homepage* (no deep links / affiliate tracking).

## Security notes
* **`nodemailer` was deliberately NOT installed** (13 high-severity advisories, no fix inside next-auth v4's peer range). Email login uses the same one-time-code flow as mobile.
* **OTP**: codes stored HMAC-hashed, 5-min expiry, 5 attempts, 30 s resend cooldown, per-IP and per-identifier rate limits. The mock sender is refused when `NODE_ENV=production`.
* **Every `/api/me/*` query is scoped by the session user id** (verified by an e2e test that a second user gets 404 on another user's plan).
* Plan totals are **recomputed server-side** on save; client totals are never trusted.
* No secrets committed. `.env.local` is git-ignored; `.env.example` lists variables.
* Chat messages are sent to the Anthropic API only when `ANTHROPIC_API_KEY` is set; users are told not to share personal details.

## Known limitations
* **Delivery 1 is large** (all long-life staples, per the spec's shelf-life rule). Later deliveries are mostly fresh items and small.
* **Rate limiter** is in-memory and keys on `x-forwarded-for`, which is only trustworthy behind a proxy that overwrites that header. Use Redis + trusted-proxy config in production.
* **The LLM path of the chat assistant typechecks but has not been run against the live API** (no key was available). The keyword fallback is tested.
* **Google login** is wired but untested (needs credentials).
* `npm audit`: remaining findings are `deepmerge-ts` inside the Prisma **CLI** (build tooling), plus dev-only lint tooling (`braces`). Re-check before release.
* Demo "quantity needs" per person are heuristics, not nutrition advice.

## Open items for human decision / approval
1. **Approve third-party packages** against Chalo's software list: next, react, tailwindcss, zustand, zod, lucide-react, next-auth@4, @next-auth/prisma-adapter, prisma/@prisma/client@6, @anthropic-ai/sdk, vitest, @playwright/test, tsx.
2. **SMS / email vendor** for real OTP delivery (IT/Security approval) — implement `OtpSender` in `src/lib/otp.ts`.
3. **Phone numbers are PII** stored on `User.phone`; confirm the data-protection treatment.
4. **Real price feeds**: legal/partner approval before connecting any platform.
5. **Production path**: feature branch → PR with ≥ 1 reviewer → CI (lint, typecheck, unit + e2e) → staging QA → release. Move SQLite → Postgres and set `NEXTAUTH_SECRET` per environment.
