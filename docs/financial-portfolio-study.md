# FinLit `Financial-Portfolio` repository: technical study

Studied at commit `6023c0d` (Merge PR #30, `codex/ui-improvements`), Oct 2026. It describes what exists today, how the stock research data flows, the bugs behind the numbers seen on the live page, and what has to be built for the redesigned, subscription-based research product.

---

## 1. What the repository is

- **Product:** the finlitventures.com website, covering marketing pages, a blog/CMS, a portfolio showcase, an AI chat assistant, and an admin-only stock fundamentals terminal.
- **Origin:** it was generated on Emergent (`.emergent/`, `test_result.md`, `memory/PRD.md`) as a portfolio site for "Nishant Jain PMS". It was then rebranded to FinLit and extended through Codex PRs (around 30 merged).
- **Size:** about 9k lines of app code. Two files dominate:
  - `frontend/src/pages/StockFundamentalsAdminPage.jsx` (3,379 lines)
  - `backend/server.py` (3,221 lines)

## 2. Architecture

```
Browser ── React 19 SPA (CRA + craco, Tailwind, shadcn/Radix, Recharts, framer-motion, lenis)
   │        hosted on Vercel (frontend/vercel.json rewrites everything to index.html)
   │        talks to  REACT_APP_BACKEND_URL + /api
   ▼
FastAPI (backend/server.py) + MongoDB (motor)
   ├─ FinEdge API  (data.finedgeapi.com)   → company profile, statements, ratios, prices, shareholding, actions, peers
   ├─ Upstox API                            → market ticker quotes; fallback for fundamentals and quotes
   ├─ Google Gemini (gemini-3.8-flash)      → /api/chat "FinLit AI" assistant
   └─ Resend                                → email alerts for contact and service enquiries
```

**Environment variables**
- **Backend:** `MONGO_URL`, `DB_NAME`, `FINEDGE_API_KEY`, `UPSTOX_ACCESS_TOKEN`, `UPSTOX_MARKET_TICKER_INSTRUMENTS`, `GEMINI_API_KEY`, `RESEND_API_KEY`, `SENDER_EMAIL`, `CONTACT_RECIPIENT_EMAIL`, `CMS_ADMIN_EMAIL`, `CMS_ADMIN_PASSWORD_HASH`, `CMS_JWT_SECRET`, `CORS_ORIGINS`, `ALLOWED_HOSTS`, `PORTFOLIO_DEPLOYED_CAPITAL`.
- **Frontend:** `REACT_APP_BACKEND_URL`, `REACT_APP_CONTACT_EMAIL`, `REACT_APP_LINKEDIN_URL`.

**CI.** `.github/workflows/webpack.yml` only runs `npm ci && npm run build` for the frontend (Node 18/20/22). No tests run in CI, and the backend is not built or tested.

## 3. Frontend map

| Route | Page | Notes |
|---|---|---|
| `/` | Home | Hero, Performance, Process, Services, Philosophy, Testimonial, Contact sections |
| `/why-we-exist`, `/services`, `/financial-product`, `/about`, `/portfolio`, `/contact` | Marketing pages | `/what-we-do` redirects to `/why-we-exist` |
| `/blog`, `/blog/:slug` | Public blog | Content from `/api/content` |
| `/blog/admin/login` | Admin login | Single admin account from env; JWT kept in `sessionStorage` (`cms_admin_token`) |
| `/blog/admin`, `/new`, `/edit/:id`, `/preview/:id` | CMS (`AdminRoute`) | TipTap rich-text editor and uploads |
| **`/blog/admin/stock-fundamentals`** | **Stock terminal (admin only)** | The page being redesigned |

**Global shell** (`App.js`):
- `MarketTicker` (Upstox), `Navbar`, `Footer`, the `FinLitAssistant` chat bubble and a Sonner toaster.
- Lenis smooth scrolling over the whole app.
- A `ThemeProvider` that sets `data-theme` on `<html>`.

**Design system**
- **Colours:** CSS variables in `src/index.css` (`--app-bg #050E1D`, `--home-gold #D4AF57`, `--navy-900 #0A1E3F`, …) with `:root[data-theme="light"]` overrides.
- **Fonts:** Cormorant Garamond for headings, IBM Plex Sans for UI (`design_guidelines.json`).
- **Logo:** the official brand logo is in the repo at `src/assets/brand/finlit-logo-transparent.png`. Use it, not a redraw.

**Data libraries installed**
- `@tanstack/react-query` is configured in `index.js`, but nothing uses it yet (the stock page uses manual `fetch` and `useState`).
- `axios` is used only by the contact and services forms. `swr` is installed but unused.

## 4. Backend API

| Method and path | Auth | Purpose |
|---|---|---|
| `GET /api/` | public | health |
| `GET /api/content`, `GET /api/content/{slug}`, `GET /api/content/assets/{key}` | public | blog |
| `POST /api/content/admin/login` | public (rate-limited) | admin JWT |
| `GET/POST/PUT/DELETE /api/content/admin/items…`, `POST /api/content/admin/upload` | admin | CMS |
| `GET /api/portfolio/report` | public | portfolio showcase JSON (parsed from uploaded broker PDF, `portfolio_report.py`) |
| `GET /api/portfolio/admin/report`, `POST /api/portfolio/admin/upload` | admin | portfolio data upload |
| **`GET /api/stocks/admin/fundamentals/search?query=`** | **admin** | symbol search (FinEdge universe) |
| **`GET /api/stocks/admin/fundamentals?query=&statement_type=&period=`** | **admin** | the full stock payload |
| `GET /api/market-ticker` | public | Upstox quotes, 60 s cache |
| `POST /api/chat` | public (10/min per IP) | Gemini assistant |
| `POST /api/contact`, `GET /api/contact` (admin), `POST /api/service-enquiry` | mixed | leads (Mongo + Resend email) |

**Tests.** 29 pytest cases in `backend/tests/`, 7 of them on stock normalisation:
- bank ROCE;
- daily vs derived P/E;
- negative-P/E handling;
- TTM profit;
- depreciation backfill.

## 5. The stock fundamentals pipeline (`_fetch_stock_fundamentals`, server.py:2596)

1. **Resolve the symbol.** Resolve against the FinEdge symbol universe, then try to match an Upstox instrument (via ISIN, symbol or name) for fallbacks.
2. **Check the cache.** Use an in-memory dict, 15 minutes per `symbol:statement_type:period`. The cache is per process, so it is lost on restart and not shared between workers.
3. **Call FinEdge sequentially** (a `requests.get` with an 18 s timeout each):
   - company profile;
   - P&L, balance sheet and cash flow, plus standalone P&L when consolidated is requested (it falls back to standalone if consolidated has no revenue);
   - 4 ratio families (`ef`, `le`, `li`, `pr`);
   - daily price ratios;
   - 3 financial-metric families (`cu`, `gr`, `av`);
   - basic financials;
   - live quote;
   - quarterly P&L (for TTM);
   - shareholding pattern;
   - corporate actions;
   - dividends;
   - peers;
   - daily quotes for 6 years.

   That is about 20 calls, and peer enrichment adds up to 2 more per peer (×12). A cold load is therefore 20–40 sequential HTTP calls; the live page showed about 2.1 s even on a warm path.
4. **Use Upstox as a fallback** for ratios, income statement, balance sheet, cash flow, profile, shareholding, corporate actions, competitors and quote. Every fallback is recorded in `sources` and `dataQuality`.
5. **Reconcile ratios** (`_reconcile_valuation_ratios`): P/E is preferred from daily price ratios, then TTM, then derived. Dividend yield is derived if missing.
6. **Return this response shape:**
   - `provider`, `sources`, `statementType`, `generatedAt`, `cached`;
   - `instrument`, `quote`, `priceHistory`, `profile`, `highlights`, `ratios`, `profitabilityRatios`;
   - `incomeStatement` / `balanceSheet` / `cashFlow` (units in ₹ Cr);
   - `shareholding`, `corporateActions`, `competitors`, `dataQuality`, `devMetrics`.

**FinEdge endpoints the backend does not use yet.** All of these returned HTTP 200 in the probe (`finedge_output.txt`, 24 of 24 succeeded):
- `segment-revenue`: per-segment revenue and EBIT. For example, Reliance FY26 O2C is ₹5.05 L Cr and Oil & Gas is ₹23,765 Cr. This powers the Gold "Revenue segmentation".
- `corp-announcements`: credit ratings, results, board meetings, with PDF links. This powers Gold "Key triggers".
- `annual-price-ratios`: historical P/E and P/B, for valuation-band charts.
- `notes`: financial statement notes.
- `shareholdings/ownership-current`: current ownership.
- `peers?group=sector`: sector peer grouping.

## 6. Where the FinLit research score comes from

It is computed **in the browser** (`buildResearchReport`, StockFundamentalsAdminPage.jsx:1271). The backend never sees or stores it.

Each metric is mapped to a score of 18 / 38 / 58 / 76 / 92 by four thresholds (`scoreFromThresholds`, line 906). A pillar is the plain average of its metrics, and the overall score is the plain average of the five pillars.

| Pillar | Inputs and thresholds (weak / fair / good / great) |
|---|---|
| Quality | ROE 6/10/14/18 · ROCE 7/12/16/22 · CFO/PAT 40/70/95/120 % |
| Growth | 5Y sales CAGR 2/6/10/16 · 5Y profit CAGR 2/8/14/22 · latest sales growth 0/5/10/18 % |
| Profitability | OPM 6/10/15/22 · NPM 4/7/10/16 % · asset turnover 0.3/0.6/0.9/1.3 |
| Balance sheet | liabilities/assets 80/65/50/35 % (lower is better) · CFO/PAT |
| Technical | price vs EMA50 −8/−2/2/8 % · vs EMA200 −12/−3/3/12 % · RSI 35/45/55/65 |

**Score bands** (`scoreLabel`): **Strong ≥ 78, Healthy ≥ 62, Mixed ≥ 45, Watch < 45.** The prototype in `stock.html` used 80/65/45 and "Weak". Align the prototype to the real bands and the "Watch" label, or agree new ones.

**Implications for the subscription plans**
- There is only one model today. The "Advanced scoring model" (Gold) does not exist yet and must be specified.
- Because scoring runs on the client and the API returns everything, any plan gating done only in the UI is cosmetic. Scores and Gold data must be computed and filtered **on the server**.

## 7. Root causes of the wrong numbers on the live page

| Symptom (BPCL) | Root cause | Fix |
|---|---|---|
| Dividend return is 89.14% for every period from 3M to 5Y | FinEdge dividend dates look like `05-Jun-2026`. `parseActionDate` builds `new Date("05-Jun-2026T00:00:00")`, which is **invalid**, so the date becomes `null`. The filter `if (date && date < cutoff)` then never excludes anything, so every dividend in the 6-year window counts for every period (jsx:1159–1179). Dividends also arrive twice, from both `/corporate-actions/all` and `/dividend`, merged without de-duplication (server.py:2225), so double counting is likely. | Parse `DD-Mon-YYYY` (and ISO) dates on the server and emit ISO `ex_date`. De-duplicate on (date, amount, type). Unit-test both. |
| Dividend payout shows 81.42% in Market Metrics but 0.38 in ratios | `dividendPayout = safeRatio(yield, PE, 100)` computes yield ÷ P/E × 100 = 5.96 ÷ 7.32 × 100 (jsx:1593). Payout is **yield × P/E**: 5.96 × 7.32 ÷ 100 ≈ 43.6%. FinEdge's own value is 0.38, a fraction meaning 38%. | Use FinEdge's payout ratio, displayed as a percentage, or the correct formula. |
| 1Y return shows −15.41% in one place and −16.2% in another | Two implementations. The report uses `priceData[len − 252]` (jsx:1301); snapshots use `priceReturn(history, 252)`, which indexes `len − 1 − 252`, one session apart, on differently filtered rows. | One server-side `returns` block, used everywhere. |
| Interest coverage shows 8.38× and 8.78× | Two sources: a FinEdge ratio vs a value derived from statements. | Pick one canonical source and show its basis in the tooltip. |
| Placeholder descriptions ("Shows a company financial metric used to compar…") | `ratioDescription` falls back to generic text for unmapped keys (jsx:757). | Build a complete glossary. |
| Raw Unix timestamps in corporate actions | `event_details` passes every raw field through (server.py:2241). | Whitelist display fields on the server. |
| Price chart dips to ₹0 (Aug 2026) | Bad ticks in `daily-quotes` are not filtered. | Drop closes ≤ 0 and isolated >35% outliers on the server, and flag them in `dataQuality`. |
| Dev metrics panel shown in the page | `DevMetricsPanel` renders for anyone who can open the admin page. | Show it only behind a debug toggle. |

## 8. Gap analysis: what the redesigned product needs that the repo lacks

| Needed for the product | Status in repo |
|---|---|
| Public stock page (`/stocks/:symbol`) | **Missing.** The only stock view is admin-only under `/blog/admin`. |
| User accounts (sign-up / login) | **Missing.** There is only one env-configured admin. |
| Plans (Basic / Bronze / Gold), entitlements, payments | **Missing.** No users, plans, subscriptions or payment gateway. |
| Server-side gating of plan features | **Missing.** The API returns everything and scoring is client-side. |
| Stock screener with filters (Basic) | **Missing.** Only symbol search exists. A screener needs ratios for the whole universe, so it needs a nightly batch job. |
| Peer comparison with P/E, ROE and so on | **Partial.** Peers carry only name and market cap. |
| Watchlist and portfolio tracking (Bronze) | **Missing.** The portfolio page is a single static showcase. |
| PDF / offline / CSV export (Bronze) | **Missing** (the prototype has a client-side version). |
| Advanced model, 15+ quarterly ratios, technical score, bucketing, segmentation, triggers (Gold) | **Missing.** Segment revenue and announcements are available from FinEdge but not used. |
| Fast, shared caching | **Missing.** The cache is in-memory, per process, and every call is sequential. |
| Tests in CI | **Missing.** CI only builds the frontend. |

## 9. Recommended build plan

**Phase 0: correctness (small, high value)**
- Fix the dividend date parsing and de-duplication, the payout formula and the bad-tick filter.
- Unify returns into one calculation.
- Complete the ratio glossary.
- Add backend tests and a GitHub Actions job (`pytest`).

**Phase 1: data layer**
- Create a `stock_snapshots` collection in Mongo, keyed by `symbol:statement:period`, with `fetchedAt` and a TTL.
- Fetch FinEdge in parallel with a thread pool or `httpx.AsyncClient`, with retries and per-endpoint timeouts.
- Run a nightly batch over the NSE universe that stores latest ratios and scores. This also powers the screener, peer ratios and stock buckets.
- **Move scoring to the server** as a versioned module (`score_model_v1`, the current thresholds), so scores are consistent, testable and gateable.

**Phase 2: public research page**
- Add a `/stocks/:symbol` route that implements the `stock.html` design: report-first hero, pillar tiles, sections, Simple-language copy.
- Use React Query for data.
- Reuse the existing theme variables, fonts and the real logo asset.

**Phase 3: accounts and plans**
- **Auth:** email OTP or Google sign-in → user record with `plan` → JWT carrying `plan`.
- **Payments:** Razorpay subscriptions, the common choice in India, with webhooks updating `users.plan`.
- **Gating on the server:** the API takes the user's plan and **omits** fields above it (it does not merely hide them). The UI shows lock cards for the omitted sections.

**Phase 4: Gold features**
- Advanced scoring model (spec needed).
- 15+ quarterly ratios.
- Technical score with EMA20/50/200, RSI, support and resistance.
- Bucketing from the nightly batch.
- Revenue segmentation (`segment-revenue`).
- Key triggers (`corp-announcements`).
- Sector benchmarking.

**Phase 5: tools**
- Watchlist (Mongo).
- Server-rendered PDF research note (Playwright or WeasyPrint), CSV export.
- "Notify me when the score changes" alerts (Resend).

## 10. Open questions for the team

1. **Advanced model.** What makes it "advanced": more inputs, sector-relative thresholds, or different weights?
2. **Score bands.** Keep 78/62/45 with "Watch", or change them?
3. **Payments and auth.** Is Razorpay acceptable? Is email OTP enough for sign-in, or is Google login needed?
4. **Hosting.** Where is the backend deployed (Emergent, Render, a VM)? This decides the cache and batch-job options (Redis, cron).
5. **FinEdge plan limits.** What are the rate limits and quota? This sets how often the nightly batch can refresh the universe.
6. **Compliance.** Public research content means wording must stay "research, not advice" (FinLit is not SEBI-registered).
