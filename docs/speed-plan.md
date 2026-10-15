# Stock page speed plan: under 1 second

**Goal:** a user searches or opens a stock, and the research report (identity, price, score, pillars, key numbers) is on screen in **under 1 second**. The rest of the page fills in progressively, with no visible "buffering".

**Hosting (team decision, Stage 1):**
- Frontend: Cloudflare Pages.
- One VPS in India (DigitalOcean Bangalore 4 GB, or Oracle Cloud Free in Mumbai/Hyderabad) running the API, Postgres + Timescale, Redis and cron workers.
- Storage and backups: Cloudflare R2.
- Model training: offline (laptop or Colab), with the model file stored in R2.

---

## 1. Why it is slow today (from the code)

The live dashboard shows **"REQ 2,139 ms · MODE CACHE"**. That is about 2.1 seconds *even when the server answers from its cache*.

| Where | What happens | Cost |
|---|---|---|
| Request path, before the cache | `_resolve_finedge_stock()` runs **before** the cache check (server.py:2598). `_try_resolve_upstox_instrument()` then tries up to 4 lookups (ISIN, symbol, name, query); failures are not cached, so they repeat on every request. | Network round-trips on every load |
| Cache miss | About 20 sequential FinEdge calls, plus up to 2 per peer (×12), each `requests.get(timeout=18)` and one after another (server.py:2627–2860). | Several seconds, often more |
| Cache design | In-memory dict, 15 min, per process. It is lost on every restart or deploy and not shared between workers. | Frequent cold loads |
| Payload | One large JSON with 6 years of daily prices, every statement, ratio, shareholding period and raw corporate-action field. | Large download and parse |
| Browser CPU | `priceChartData()` recomputes EMA20/50/200, RSI and Bollinger bands over the full daily history, and is called again for every return period (7×) and in several builders. Scores, derived metrics and insights are also computed in the browser. | Main-thread blocking |
| Browser rendering | All 14 sections and every Recharts chart render at once. | Long first render, janky scroll |
| Quarterly view | The page fires **two** full requests (quarterly plus an annual "companion"). | Double the work |

## 2. Target architecture: never call FinEdge on a user request

```
            nightly + intraday cron (VPS)
FinEdge ──► ingest worker ──► Postgres/Timescale (raw + normalized + scores)
Upstox  ──►                   └─► Redis: ready-to-serve JSON per stock and section
                                          │
User ─► Cloudflare (edge cache) ─► FastAPI ─┘  (reads Redis only; ~5–20 ms)
```

1. **Precompute everything.** A worker refreshes every listed stock:
   - **Nightly:** statements, ratios, shareholding, actions, peers, segments, announcements, scores.
   - **Every 1–5 minutes during market hours:** quotes.

   It writes normalised rows to Postgres/Timescale (daily prices in a hypertable) and ready-to-serve JSON to Redis. Scores, derived metrics, insights, returns and indicators (EMA, RSI) are computed **here, once**. The browser no longer computes them.
2. **The request path reads Redis only.** A `GET /api/stocks/{symbol}/summary` call is a Redis `GET` plus plan filtering, about 5–20 ms on the server. If Redis misses, read Postgres (about 30 ms). If the stock has never been ingested, queue it and show a skeleton; never block on FinEdge.
3. **Split the payload by what the screen needs first:**
   - `/summary` (about 8–15 KB gzip): identity, quote, 52-week range, score and pillars, the read, KPIs, a 1-year sparkline.
   - `/financials?period=yearly|quarterly`, `/ratios`, `/ownership`, `/peers`, `/price?range=1y`: fetched in parallel right after the summary, or when the section scrolls into view.
   - Price history goes as compact arrays (`t[]`, `c[]`, `v[]`) at the resolution of the selected range: weekly for 3Y and 5Y, daily for 1Y. Not 6 years of daily rows.
4. **Edge caching.** Public responses carry `Cache-Control: public, max-age=60, stale-while-revalidate=600` and an `ETag`. Cloudflare serves repeat views from the nearest city. Plan-specific variants are cached per plan, never per user.
5. **Instant search.** Ship a compact symbol index (about 5,000 rows: symbol, name, ISIN, sector; ~150 KB gzip, cached forever with a versioned URL). Filter it in the browser, so typing gives results in under 50 ms with no API call.

## 3. Frontend rules for "no buffering"

- Code-split the stock route. The page shell and report skeleton paint immediately.
- Use React Query:
  - **Prefetch** a stock's summary when its search suggestion is highlighted or a peer row is hovered, so opening it is instant.
  - Keep recent stocks cached for back/forward navigation.
- Render sections lazily: draw each chart once when it scrolls near the viewport. The redesigned prototype (`stock.html`) already does this and measured a steady 60 fps.
- Animate only `transform` and `opacity`. Avoid `backdrop-filter`, full-screen blended overlays, and `scrollIntoView` from scroll-spy.
- Show real skeletons that match the final layout, so nothing jumps when data arrives.

## 4. Budget

| Step | Target |
|---|---|
| DNS + TLS + edge (Cloudflare, India PoP) | ≤ 60 ms |
| API `/summary` (Redis) | ≤ 20 ms server, ≤ 120 ms including network from the India VPS |
| JS for the stock route (code-split, cached after the first visit) | ≤ 180 KB gzip |
| **Report visible (score, pillars, KPIs)** | **≤ 1.0 s** on 4G for a first visit; ≤ 300 ms on repeat visits or prefetch |
| All sections loaded | ≤ 1.5 s |
| Switching stocks from search or peers | ≤ 300 ms (prefetched) |

## 5. Quick wins in the current code (before the rebuild)

1. Check the cache **before** symbol resolution, keyed on the normalised query.
2. Cache the Upstox resolution result, including failures, for 24 h.
3. Move the cache from process memory to Redis (or Mongo with a TTL index) so it survives restarts and is shared.
4. Fetch FinEdge endpoints **in parallel** (`ThreadPoolExecutor(max_workers=8)` or `httpx.AsyncClient`). A cold load drops from the sum of all calls to roughly the slowest one.
5. Drop per-peer enrichment from the request path; precompute peers nightly.
6. Down-sample price history on the server (weekly for >1Y) and send indicators precomputed.
7. Memoise `priceChartData` once per payload in the browser, instead of per return period.
8. In quarterly mode, fetch only the extra annual fields needed, not a second full payload.

Quick wins 1–4 alone should take cached loads from about 2.1 s to around 100–300 ms, and cold loads from many seconds to about 1–2 s. The architecture in §2 is what makes the report reliably appear in under 1 second.

## 6. How we will prove it

- `devMetrics` stays (admin only): server ms, cache source, payload KB.
- Add a k6 or `hey` load test against `/summary` (p50/p95 at 50 concurrent users).
- Run Lighthouse and WebPageTest on 4G India profiles for `/stocks/BPCL`, recording LCP, INP and CLS. Targets: LCP < 1.5 s, CLS < 0.05.
- Set a CI budget check: fail the build if the stock route's JS exceeds 180 KB gzip.
