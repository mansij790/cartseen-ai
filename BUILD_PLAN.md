# SmartGrocery AI — Build Plan

> **For the building agent:** This is the full spec. Build it **milestone by milestone** (Section 12), and check each milestone's acceptance criteria before you start the next one. Where this document is silent, choose the simplest option that works and record it in `DECISIONS.md`.
>
> **Status:** A human should approve the stack and architecture choices (Sections 2 and 3) before the build starts. They are recommendations, not final decisions.

---

## 0. Product summary

SmartGrocery AI is a responsive web app for customers in India. It does three things:

1. **AI Grocery Planner.** It turns a monthly budget, household size, number of deliveries and preferences into a monthly grocery list that stays within budget. The list is split across 3–4 deliveries, with shelf life taken into account.
2. **Price Comparison.** It compares product prices, fees and delivery times across grocery platforms (Blinkit, Zepto, Swiggy Instamart, BigBasket, Flipkart Minutes, and later Amazon Fresh and JioMart).
3. **Budget Manager.** It shows a savings dashboard and keeps saved plans and history.

Value proposition: *"Tell us your grocery budget. AI plans what you need, splits it across the month, compares stores, and helps you spend less."*

### Non-negotiable rules

- **Never fake live prices.** The MVP uses **mock data only**. Every page that shows prices must display this banner: **"Demo prices — connect live data sources for real-time pricing."**
- **Do not scrape** any grocery platform. Scraping may violate their Terms of Service. Real data will come later from approved APIs or partner feeds, through the adapter interface in Section 4.
- If a store has no price for a product, show **"Price unavailable"**. Never guess a price.
- Never hotlink brand or product images from the platforms. Use local placeholder images (emoji, SVG or a generic illustration per category).
- No secrets in code. Commit `.env.example` only.

---

## 1. Tech stack (recommended — needs human approval)

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js (App Router) + TypeScript (strict)** | Frontend and backend API routes in one repo; easy to deploy |
| Styling | **Tailwind CSS** + **shadcn/ui** components | Fast, consistent, mobile-first UI |
| Icons | `lucide-react` | Clean icon set |
| Charts | `recharts` | Budget donut and progress charts |
| Validation | `zod` | Validate every API input and form |
| DB / ORM | **Prisma + SQLite** in dev (Postgres-ready schema) | Zero setup locally; switch the datasource for prod |
| Auth | **Auth.js (NextAuth v5)**: Google OAuth + email magic link; mobile OTP behind a provider interface (mock in dev) | Covers all three login methods in the spec |
| AI | `@anthropic-ai/sdk` for the chat assistant, model ID read from env `ANTHROPIC_MODEL` | Natural-language intent parsing only (see Section 7) |
| State | React Server Components + `zustand` for the client-side plan and basket state | Simple |
| Tests | `vitest` (engine unit tests) + `playwright` (one smoke test per page) | |
| Lint/format | ESLint + Prettier | |

Alternatives, if the human reviewer prefers them: a separate FastAPI backend (better for a later Python ML pipeline, but adds a second deployable), or Supabase for auth and DB (less code, adds vendor lock-in).

All third-party packages must be checked against Chalo's approved software list before use. Raise an IT/Security request for any package that is not on it.

---

## 2. Folder structure

```
groceryApp/
├── .env.example                 # ANTHROPIC_API_KEY=, ANTHROPIC_MODEL=, AUTH_SECRET=, AUTH_GOOGLE_ID=, AUTH_GOOGLE_SECRET=, DATABASE_URL=, EMAIL_SERVER=
├── prisma/
│   ├── schema.prisma
│   └── seed.ts                  # loads mock catalog + store offers
├── src/
│   ├── app/
│   │   ├── page.tsx                     # Home
│   │   ├── planner/page.tsx             # AI Grocery Planner form + result
│   │   ├── compare/page.tsx             # Search + single/multi product compare
│   │   ├── product/[id]/page.tsx        # Product detail across stores
│   │   ├── my-plan/page.tsx             # Current plan, deliveries, edits
│   │   ├── savings/page.tsx             # Savings dashboard
│   │   ├── profile/page.tsx             # Account, saved plans, favourites, preferred apps
│   │   ├── login/page.tsx
│   │   └── api/
│   │       ├── products/search/route.ts
│   │       ├── compare/route.ts
│   │       ├── basket/optimize/route.ts
│   │       ├── plan/generate/route.ts
│   │       ├── plan/[id]/route.ts
│   │       ├── chat/route.ts
│   │       └── auth/[...nextauth]/route.ts
│   ├── components/          # ui/ (shadcn), layout/, planner/, compare/, dashboard/, chat/
│   ├── lib/
│   │   ├── data/
│   │   │   ├── types.ts             # domain types (Section 3)
│   │   │   ├── provider.ts          # PriceProvider interface
│   │   │   ├── mockProvider.ts      # reads mock data / DB
│   │   │   └── registry.ts          # picks provider(s) by env/pincode
│   │   ├── engine/
│   │   │   ├── pricing.ts           # final cost calc
│   │   │   ├── compare.ts           # single + multi product compare
│   │   │   ├── basketOptimizer.ts   # store-subset optimizer
│   │   │   ├── valueScore.ts        # composite value score
│   │   │   ├── planner.ts           # budget allocation + list generation
│   │   │   ├── deliverySplit.ts     # shelf-life aware delivery scheduling
│   │   │   └── adjust.ts            # reduce budget, substitute, remove, add
│   │   ├── ai/
│   │   │   ├── intentParser.ts      # LLM → structured action (with rule-based fallback)
│   │   │   └── ruleParser.ts
│   │   ├── auth.ts
│   │   └── validation.ts            # zod schemas
│   └── mock/
│       ├── stores.ts
│       ├── products.ts              # 80 products
│       └── offers.ts                # generated per store, deterministic seed
└── tests/
    ├── engine/*.test.ts
    └── e2e/*.spec.ts
```

---

## 3. Data model

### 3.1 Domain types (`lib/data/types.ts`)

```ts
type Category =
  | 'staples' | 'pulses' | 'vegetables' | 'fruits' | 'dairy'
  | 'breakfast_snacks' | 'oil_spices' | 'meat_eggs' | 'household' | 'beverages';

type ShelfLife = 'long' | 'medium' | 'short';   // long > 30d, medium 7–30d, short < 7d

interface Product {
  id: string;
  name: string;            // "Fortune Sunflower Oil"
  brand: string;
  category: Category;
  quantity: number;        // 1
  unit: 'g' | 'kg' | 'ml' | 'l' | 'pcs' | 'pack';
  baseUnitQty: number;     // normalized to g / ml / pcs for per-unit comparison
  isVeg: boolean;
  shelfLife: ShelfLife;
  qualityTier: 1 | 2 | 3;  // 1 = budget, 2 = standard, 3 = premium
  tags: string[];          // synonyms for search: ["oil", "refined oil", "sunflower"]
  image: string;           // local path or emoji
}

interface Store {
  id: 'blinkit' | 'zepto' | 'instamart' | 'bigbasket' | 'flipkart_minutes' | string;
  name: string;
  deliveryFee: number;             // ₹, applied when below freeDeliveryAbove
  freeDeliveryAbove: number | null;
  platformFee: number;             // ₹ per order
  smallCartFee: number;            // ₹ when below minOrderValue
  minOrderValue: number;
  typicalDeliveryMins: number;     // e.g. 12; BigBasket slot = "Tomorrow" → use 24*60 and a label
  deliveryLabel?: string;
  servesPincodes: string[] | '*';
}

interface StoreOffer {
  productId: string;
  storeId: string;
  pincode: string | '*';
  mrp: number;
  price: number | null;            // null => "Price unavailable"
  discountPct: number;
  available: boolean;
  deliveryMins: number;
  lastUpdated: string;             // ISO timestamp
  source: 'mock' | 'api' | 'feed';
}

interface Coupon {
  storeId: string; code: string; minCart: number;
  flatOff?: number; pctOff?: number; maxOff?: number;
}
```

### 3.2 Prisma models

`User`, `Account`, `Session`, `VerificationToken` (Auth.js), plus:

- `Product`, `Store`, `StoreOffer`, `Coupon` (mirror the types above; `StoreOffer` has a unique key on `(productId, storeId, pincode)`)
- `GroceryPlan` (userId, budget, people, deliveries, pincode, preferences JSON, totals, createdAt)
- `PlanItem` (planId, productId, qty, chosenStoreId, unitPrice, deliveryNo)
- `Favourite` (userId, productId)
- `SearchHistory` (userId, query, createdAt)
- `PreferredStore` (userId, storeId)
- `BudgetHistory` (userId, month, budget, spent)

---

## 4. Data provider layer (ready for real APIs later)

```ts
interface PriceProvider {
  id: string;
  getOffers(productIds: string[], pincode: string): Promise<StoreOffer[]>;
  searchProducts(query: string, pincode: string): Promise<Product[]>;
  getStores(pincode: string): Promise<Store[]>;
}
```

- `MockProvider` is the only implementation in the MVP.
- `registry.ts` returns the provider list from env (`PRICE_PROVIDERS=mock`). Adding a real feed later means adding a new class and nothing else.
- The engine code must never import mock data directly. It only goes through the provider.
- Every price shown in the UI also shows `lastUpdated` and a source badge (`Demo`).

---

## 5. Mock data requirements

- **Stores (5):** Blinkit, Zepto, Swiggy Instamart, BigBasket, Flipkart Minutes. Give them realistic but clearly fictional fee structures, for example:
  - Blinkit: delivery ₹30 under ₹199, platform fee ₹2, ~12 min
  - Zepto: delivery ₹25 under ₹99, small-cart fee ₹15 under ₹99, ~15 min
  - Instamart: delivery ₹35 under ₹199, platform fee ₹5, ~18 min
  - BigBasket: free delivery above ₹300 (else ₹40), slot delivery "Tomorrow"
  - Flipkart Minutes: delivery ₹29 under ₹199, ~14 min
- **Products (~80)** across all categories. Include common Indian staples and brands: Aashirvaad Atta 5kg, India Gate Basmati 1kg, Tata Salt 1kg, Fortune Sunflower Oil 1L, Tata Sampann Toor Dal 1kg, Amul Taaza Milk 500ml, Amul Butter 100g, Maggi 4-pack, Britannia Bread, Parle-G, Tata Tea Gold 250g, Everest spices, onions/potato/tomato 1kg, seasonal fruits, eggs 6/12 pcs, chicken 500g (non-veg), and some store/budget brands. Include **2–3 alternatives per staple** at different quality tiers so the planner can substitute.
- **Offers:** generate them deterministically (seeded PRNG) from a base price, varying ±3–12% per store. About 5–10% of offers should be `available: false` or `price: null` so the "Price unavailable" path gets exercised.
- **Coupons:** 1–2 per store, for example "₹50 off above ₹499".
- All pincodes are served (`'*'`), but the pincode input must be validated (6 digits, first digit not 0).

---

## 6. Engines (pure TypeScript functions, unit tested)

### 6.1 Final cost (`pricing.ts`)

```
itemsSubtotal  = Σ price × qty
deliveryFee    = subtotal >= freeDeliveryAbove ? 0 : store.deliveryFee
smallCartFee   = subtotal <  minOrderValue     ? store.smallCartFee : 0
couponDiscount = best applicable coupon (respect minCart, maxOff)
finalCost      = itemsSubtotal + deliveryFee + platformFee + smallCartFee − couponDiscount
```

Return a full breakdown object; the UI shows every line.

### 6.2 Single and multi product compare (`compare.ts`)

- **Single:** a row per store with price, MRP, discount %, delivery time, availability and the per-unit price (₹/kg or ₹/L). Flag **Lowest Price** and **Fastest Delivery** separately.
- **Multi (2–10 items):**
  - Cheapest store per item, plus the sum of item prices.
  - **Cheapest Overall (split):** the result of the basket optimizer, *including fees*.
  - **Most Convenient (single store):** the best single store by `finalCost`. Only stores that have every item available qualify; otherwise show which items are missing.
  - Show the ₹ difference between the two options.

### 6.3 Smart basket optimizer (`basketOptimizer.ts`)

With 5 stores there are only 31 non-empty store subsets, so **enumerate them all** and get the exact answer:

```
for each non-empty subset S of stores:
    for each item: assign to the store in S with the lowest available price (skip S if any item is unavailable in all of S)
    cost(S) = Σ over stores in S of finalCost(store, assignedItems)
pick argmin cost(S); tie-break by fewer stores, then faster max delivery
```

- Optional parameter `maxStores` (default 2) to limit how many stores a basket is split across. A setting lets the user change it.
- Return the chosen assignment, the per-store breakdown, and the runner-up single-store option.
- Required test case (Spec §7): Store A has items ₹500 with ₹0 fees, and Store B has items ₹480 with ₹49 delivery. The optimizer **must pick Store A**.

### 6.4 Value score (`valueScore.ts`)

This is used to pick *which* product variant goes into the plan. It must not simply pick the cheapest.

```
valueScore = w1·(1 − normPricePerUnit) + w2·qualityTier/3 + w3·availabilityRatio
           + w4·(1 − normDeliveryMins) + w5·packSizeFit
defaults: w1 .40, w2 .25, w3 .15, w4 .10, w5 .10
```

- `packSizeFit` rewards pack sizes that match the household's required quantity, so a 10kg rice pack isn't bought for 1 person.
- Keep the weights in one config object, and explain them in a tooltip ("Why this product?").

### 6.5 Planner (`planner.ts`)

**Input** (zod-validated):

```ts
{ budget: 500..50000, people: 1..10, days: 7|14|30, deliveries: 1..6, pincode,
  diet: 'veg'|'nonveg'|'both', mealsPerDay?: 1..4, dietary?: string[] /* jain, no-onion-garlic, low-sugar, high-protein */,
  haveAtHome: string[], mustInclude: string[], exclude: string[] }
```

**Algorithm:**

1. **Requirements.** A `NEEDS` table gives monthly per-person quantities per category, for example rice 3kg, atta 3kg, dal 1.5kg, oil 0.75L, milk 12L, vegetables 8kg, fruits 3kg, sugar 0.6kg, tea 0.15kg, salt 0.25kg, eggs 15 (non-veg), chicken 1kg (non-veg). Scale it by `people × days/30`. Apply `mealsPerDay` as a multiplier (3 is the baseline). Remove `haveAtHome` and `exclude` items, and force `mustInclude` items in.
2. **Tiers.** Every need is tagged `essential` (staples, dal, oil, salt, onion/potato, milk), `important` (vegetables, fruits, tea, sugar, eggs) or `nice` (snacks, breakfast cereal, premium variants, beverages).
3. **Budget allocation.** Use the default category weights below. Shift weight toward `nice` and higher quality tiers as the budget per person grows.

   | Category | Weight |
   |---|---|
   | Staples (rice/atta) | 17.5% |
   | Dal/Pulses | 15% |
   | Vegetables | 22.5% |
   | Fruits | 12.5% |
   | Dairy | 12.5% |
   | Snacks/Breakfast | 10% |
   | Oil/Spices/Other | 10% |

   For non-veg diets, carve a `meat_eggs` share (~12%) proportionally out of the others. Allocations must sum exactly to the budget (put any rounding remainder in the largest category).
4. **Budget bands**, based on budget ÷ (people × days/30):
   - **< ₹1,200:** Basic. Essentials only, tier-1 brands, minimal fruit.
   - **₹1,200–2,000:** Standard. Adds fruit and some snacks.
   - **> ₹2,000:** Premium. Tier-2/3 brands, more variety, larger quantities.

   With 2 people this gives ₹2,000 → Basic/Standard, ₹3,000 → more variety, ₹5,000 → premium.
5. **Selection.** For each need, pick the variant with the highest `valueScore` whose cost fits the category allocation. Price it using the basket optimizer per delivery (step 7).
6. **Fit loop.** While `total > budget`, apply these steps in order and re-price after each:
   1. Downgrade the quality tier of `nice` items, then `important`, then `essential`.
   2. Reduce `nice` quantities, then drop `nice` items.
   3. Reduce `important` quantities by up to 30%.
   4. Never cut essentials below 70% of the requirement. If the plan still doesn't fit, return `status: 'over_budget'` with the shortfall and suggestions. Never return an over-budget plan silently.

   If there's money left (more than 8% unused), upgrade or add `nice` items while staying under budget.
7. **Delivery split** (`deliverySplit.ts`):
   - `long` shelf-life items (rice, atta, dal, oil, spices, salt, sugar, tea) go in **Delivery 1**.
   - `medium` items (onion, potato, eggs, butter, biscuits) are split across 2 deliveries (1 and ceil(n/2)+1).
   - `short` items (leafy vegetables, tomatoes, fruits, milk, bread, paneer, chicken) are split evenly across **all** deliveries.
   - Label each delivery with a week ("Delivery 1 — Week 1") and a target date.
   - Run the basket optimizer for each delivery so fees are counted per order, and avoid splitting a small delivery across stores if the fees outweigh the savings.
8. **Output:**

```ts
{ status: 'ok'|'over_budget', budget, totalEstimated, savings, budgetUsedPct,
  allocation: {category, allocated, planned}[],
  deliveries: {no, label, date, items: PlanItem[], storeBreakdown, subtotal, fees, total}[],
  itemsCount, notes: string[] /* human-readable reasoning e.g. "Bought rice once in Delivery 1 — long shelf life" */ }
```

### 6.6 Adjustments (`adjust.ts`)

These are pure functions on a plan, and both the chat assistant and the UI buttons use them:

- `reduceBy(plan, amount)` runs the fit loop against `total − amount`
- `substituteCheaper(plan, scope?)` swaps items to the next-lower tier variant, and reports each swap with the ₹ saved
- `removeItems(plan, names[])` and `addItems(plan, names[])` use fuzzy matching on name and tags
- `changeBudget(plan, newBudget)` re-runs the planner with the same inputs

---

## 7. AI chat assistant ("Grocery AI Assistant")

**Architecture: the LLM only parses intent. Deterministic engines do all prices and maths.** This keeps numbers accurate and stops prices from being invented.

1. `POST /api/chat` receives `{ message, planId? }`.
2. `intentParser.ts` calls the Anthropic Messages API with **tool use**, defining a single tool `grocery_action` whose input schema is a discriminated union:
   - `create_plan {budget, people, days, diet?, deliveries?}`
   - `cheapest_product {query}`
   - `reduce_budget {amount}`
   - `substitute_cheaper {category?}`
   - `remove_items {items[]}`
   - `add_items {items[]}`
   - `change_budget {budget}`
   - `smalltalk_or_help {}`
3. Validate the tool input with zod, run the matching engine function, and return `{ reply, updatedPlan?, comparison? }`. The reply text is composed from engine results using a template. Optionally, a second short LLM call can phrase a summary from the engine JSON, but the prompt must forbid adding numbers that aren't in that JSON.
4. **Fallback:** if `ANTHROPIC_API_KEY` is not set or the call fails, `ruleParser.ts` takes over. It uses regex and keyword matching for ₹ amounts, "people", "days/week/month", "cheapest", "reduce", "remove", "already have", "replace … cheaper". The app must work fully without an API key.
5. **Safety:** cap message length (500 chars), rate-limit per IP/user, never log full user messages that contain personal data, and keep the system prompt server-side only.
6. **UI:** a floating chat button on all pages that opens a drawer. Show suggested prompts (the six examples from the spec). When the plan changes, show a diff card ("−₹210 · swapped 3 items · removed Rice, Oil") with an **Undo** button.

Required examples that must work, with and without an API key:
- "I have ₹2,000. Make me a grocery plan for 2 people."
- "Which app has the cheapest cooking oil?"
- "I need groceries for 7 days under ₹500."
- "Can you reduce my grocery list by ₹200?"
- "Replace expensive products with cheaper alternatives."
- "I already have rice and oil. Remove them from my plan."

---

## 8. Pages and UI

**Visual identity:** fresh grocery look. Primary green (`#16A34A`), accent saffron/orange (`#F97316`) for price badges and savings, neutral slate text, a white or very light green background, `rounded-2xl` cards, soft shadows, and the Inter or Plus Jakarta Sans font. Mobile-first, with a bottom tab bar on mobile and a top nav on desktop. Format all prices as `₹1,925` using `Intl.NumberFormat('en-IN')`. Support dark mode.

**Navigation:** Home · AI Grocery Planner · Compare Prices · My Plan · Savings · Profile

| Page | Contents |
|---|---|
| **Home** | Hero: "Plan Your Monthly Groceries. Stay Within Your Budget." Subheading as in the spec. CTAs: **Start Grocery Planning** (→ /planner) and **Compare Prices** (→ /compare). A 5-step flow visual: ₹2,000 Budget → AI Planning → Multiple Stores → Lowest Price → Monthly Grocery Plan (icon cards with arrows, stacked vertically on mobile). Then a "How it works" section, a feature cards grid, and the demo-data banner. |
| **Planner** | A multi-step form (Basics → Preferences → Items) with every field from Spec §2. Chip inputs for have-at-home / must-include / exclude, with autocomplete from the catalog. A **budget slider (₹1,000–₹10,000, step ₹100)** plus a number input. The result appears below or beside the form: an AI Recommendation card, an allocation donut, delivery cards and the item list. **The slider re-runs the planner live** (debounced 300ms, client-side or via the API) and shows a band label (Basic / Standard / Premium). Buttons: Save plan, Reduce ₹200, Cheaper alternatives. |
| **AI Recommendation card** | "You can complete this month's grocery plan for approximately ₹1,925, saving ₹75 from your ₹2,000 budget." Stat chips for Estimated savings, Deliveries, Products and Budget used %. |
| **Compare Prices** | Search bar "Search any grocery product…" with debounced autocomplete. A toggle between **Single** and **Basket (2–10 items)** modes. Single mode shows a comparison table (App, Price, MRP/discount, Delivery, Availability, ₹/unit, Final price, Best badges), plus a "Lowest Price: X — ₹" and "Fastest Delivery: Y — Z min" highlight. Basket mode shows the cheapest store per item, then two side-by-side option cards (**Cheapest Overall** split vs **Most Convenient** single store), each with a full fee breakdown and the ₹ difference. |
| **Search results / product card** | Image, name, quantity, a price row per platform, delivery time, discount badge, final price, **Buy Now**, and a heart (favourite). Buy Now opens the store's **homepage** in a new tab (no deep links, no affiliate tracking in the MVP) with a "Prices may differ on the store" note. |
| **Filters** | Sort by Lowest price, Fastest delivery, Highest discount or Best value (value score). Filter by Brand, Quantity/pack size, Vegetarian, Available only, Store, and Category. Sync filters to URL query params. |
| **My Plan** | The current or saved plan. Delivery timeline (Week 1–4 cards with items, store split, subtotal, fees, total). Edit quantity, remove and swap per item. Export the shopping list (copy to clipboard or print view). |
| **Savings** | Monthly Budget, Planned Spending, Estimated Savings, Products and Deliveries tiles. A budget progress bar (green under 90%, amber 90–100%, red over budget). Message "You are ₹75 under budget 🎉" (or "₹X over budget" with a "Fix it" button). A category allocated-vs-planned bar chart, and month-over-month history for logged-in users. |
| **Profile** | Account details, saved plans, favourites, recent searches, budget history, shopping history (plans marked "purchased"), preferred apps (which also bias the optimizer), and a delete-my-data button. |
| **Login** | Continue with Google, Email (magic link), and Mobile (OTP). Guests can use the planner and compare without logging in. Their plan is kept in localStorage and offered for saving after login. |

**Accessibility:** all inputs labelled, contrast at WCAG AA or better, keyboard-navigable, `aria-live` region for plan updates.

---

## 9. API contracts

All endpoints validate input with zod and return `{ error: { code, message } }` with the right status on failure. Never return stack traces to the client.

| Method & path | Body / query | Returns |
|---|---|---|
| `GET /api/products/search` | `q, pincode, filters…, sort, page` | `{ products: (Product & {offers})[], total }` |
| `POST /api/compare` | `{ items: {query or productId, qty}[1..10], pincode }` | single/multi comparison (6.2) |
| `POST /api/basket/optimize` | `{ items, pincode, maxStores? }` | optimizer result (6.3) |
| `POST /api/plan/generate` | planner input (6.5) | plan output |
| `GET/PUT/DELETE /api/plan/[id]` | auth required, owner only | plan |
| `POST /api/chat` | `{ message, plan? \| planId? }` | `{ reply, updatedPlan?, comparison?, action }` |

---

## 10. Security and quality requirements

- Validate every input with zod, on both the server and the client.
- Auth: Auth.js sessions with httpOnly, secure, sameSite cookies. Every plan, favourite and history route checks that the requester owns the record (protects against IDOR).
- **Mobile OTP:** the MVP ships with a `MockOtpProvider` that only works in dev and logs "OTP sent" without the code value. Production needs an approved SMS vendor, which is out of scope, so flag it in `DECISIONS.md`. Put OTP rate limiting and expiry (5 min, 5 attempts) in place now.
- Rate-limit `/api/chat` and the auth endpoints (an in-memory token bucket is fine for the MVP; note that prod needs Redis or similar).
- Security headers via `next.config` (CSP, X-Frame-Options, Referrer-Policy). Outbound "Buy Now" links use `rel="noopener noreferrer"`.
- Never log PII (email, phone), tokens or full chat messages. Mask phone numbers in the UI (`98xxxxxx21`).
- Commit no secrets. `.env*` goes in `.gitignore`, and only `.env.example` is committed.
- Error boundaries on every page, plus loading skeletons and empty states ("No products found", "Price unavailable").
- Performance: debounce search, memoize engine results by input hash, and lazy-load the charts and chat drawer.

---

## 11. Tests (minimum)

`vitest` unit tests for the engines:
- `pricing`: free-delivery threshold, small-cart fee, coupon `minCart` and `maxOff`.
- `basketOptimizer`: **the Store A ₹500/₹0 vs Store B ₹480+₹49 case must pick A**, an unavailable item forces another store, `maxStores` is respected.
- `compare`: lowest-price and fastest-delivery flags are computed separately, and `price: null` shows as unavailable.
- `planner`: ₹2,000 / 2 people / veg / 4 deliveries gives `total ≤ 2000`, `deliveries.length === 4`, no long-shelf-life item outside Delivery 1, and fresh items in every delivery. ₹500 / 1 person / 7 days returns a valid plan. A budget that is impossible for 6 people returns `over_budget` with suggestions. Excluded and have-at-home items never appear. Allocation sums to the budget exactly.
- `planner` monotonicity: more budget never produces fewer essential items.
- `ruleParser`: each of the six example prompts maps to the correct action.

`playwright` smoke tests: each page loads, the planner form submits, the slider updates totals, compare shows a table, and chat replies to an example prompt in no-API-key mode.

---

## 12. Milestones (build in this order)

| # | Milestone | Acceptance criteria |
|---|---|---|
| 1 | Scaffold: Next.js + TS + Tailwind + shadcn, layout, nav (top and bottom tab bar), theme, demo banner | `npm run dev` works, all 6 nav routes render placeholders, lint passes |
| 2 | Types, mock data (5 stores, ~80 products, seeded offers, coupons), Prisma schema and seed, `PriceProvider` + `MockProvider` | `npx prisma db seed` works, product search returns results |
| 3 | Engines: pricing, compare, basket optimizer, value score, with tests | All engine tests pass |
| 4 | Compare Prices page: search, filters, single and basket modes, product cards | Spec §5–7 and §10–11 behaviours visible in the UI |
| 5 | Planner engine + delivery split + adjust, with tests | Planner tests pass |
| 6 | Planner page, AI Recommendation card, budget slider (live), My Plan page | Changing the slider from ₹2,000 → ₹3,000 → ₹5,000 visibly changes the band, items and totals |
| 7 | Savings dashboard | Tiles, progress bar, under/over-budget message, category chart |
| 8 | Auth (Google, email, mock OTP) + persistence (plans, favourites, searches, budgets, preferred stores) + Profile | A logged-in user can save, reload and delete a plan; the guest plan migrates after login |
| 9 | Chat assistant (LLM intent parsing + rule fallback), diff card and undo | All 6 example prompts work with and without an API key |
| 10 | Polish: responsiveness (360px → desktop), a11y pass, empty, error and loading states, security headers, README | Lighthouse mobile ≥ 90 for Performance and Accessibility; README covers setup, env vars, architecture and how to add a real price provider |

---

## 13. Out of scope for the MVP (document as future work)

- Real price feeds or partner APIs (Amazon Fresh, JioMart, and others) through new `PriceProvider` implementations, after legal and partner approval
- Real SMS OTP vendor; deep links and affiliate checkout
- Price history and alerts; nutrition-based planning; regional festival or seasonal planning
- Production deployment. It must go through a feature branch, a PR with at least one reviewer, CI (lint, typecheck, tests) and staging QA before any production release.

---

## 14. Deliverables checklist

- [ ] Working app covering Spec sections 1–18 at MVP level
- [ ] Mock catalog with ~80 products across 5 stores, clearly labelled as demo
- [ ] Engines with passing unit tests; Playwright smoke tests
- [ ] `README.md` (setup, scripts, env vars, architecture diagram, adding a provider)
- [ ] `DECISIONS.md` (assumptions and choices made during the build)
- [ ] `.env.example`; no secrets in the repo
