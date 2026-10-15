# Claude Code Prompt — FinLit Stock Research Page (v2, report-first)

> Paste everything below the line into Claude Code from the root of the FinLit front-end repo.
> A working reference prototype of this design lives in `simpdash/stock.html` (static HTML, BPCL data). Open it side by side while building; match it, then improve on it.

---

## Context

Rebuild the **Stock Fundamentals** page of FinLit (finlitventures.com, `/blog/admin/stock-fundamentals`) as a premium, launch-ready research page for many users, including high-net-worth investors. The backend models and data pipeline are finished. This is a **presentation-layer rebuild only**: do not change APIs, contracts or data shapes; reuse existing fetch hooks.

The goal is **stock research that feels simple, calm and trustworthy**. The **Research Report** (composite score, five pillars, notes, insights) is the product's centrepiece and must be the first thing a user sees.

Detected stack on the live page: React + Tailwind + Recharts. **Before writing code, inspect the repo** and report the framework, router, data layer and the API response type for fundamentals.

---

## 1. Visual identity: "Private Bank Research Note"

Deep vault navy, champagne gold used sparingly, an editorial serif for names and scores, and perfectly aligned tabular numbers. Light theme is warm fine paper with navy ink, never a plain white afterthought. No neon, purple gradients, glassmorphism or pill-everything SaaS look.

### 1.1 Colour tokens (rich, not minimal)

Put these as CSS variables on `:root` / `[data-theme="dark"]` / `[data-theme="light"]`, map them into `tailwind.config`, and use no hex values in components. The page is deliberately colourful. Each of the five pillars has its own identity colour, and that colour is reused everywhere the pillar's evidence appears.

| Token | Dark | Light |
|---|---|---|
| `--bg` | `#060C1A` | `#F1F4FB` (cool porcelain with soft blue and gold glows) |
| `--s1 / --s2 / --s3` | `#0D182E / #122039 / #1A2A48` | `#FFFFFF / #F7F9FE / #E9EEF8` |
| `--text / --muted / --faint` | `#EEF2FA / #A7B2CC / #8390AD` | `#0B1530 / #44516D / #5F6B88` |
| `--gold` brand (+ `--gold-2`, `--gold-deep` for gradients) | `#EBC46A` | `#9A6A0E` (text) / `#E3AC34` (fills) |
| `--gain / --loss / --warn / --info` | `#34D399 / #FB7185 / #F59E0B / #60A5FA` | `#047857 / #BE123C / #B45309 / #1D4ED8` |
| Pillars: quality / growth / profitability / balance / technical | `#34D399 / #60A5FA / #A78BFA / #22D3EE / #F472B6` | `#059669 / #2563EB / #7C3AED / #0891B2 / #DB2777` |
| Verdicts: strong / healthy / mixed / weak | `#34D399 / #22D3EE / #F59E0B / #FB7185` | `#047857 / #0E7490 / #B45309 / #BE123C` |
| Plan tiers: basic / bronze / gold | `#38BDF8 / #F59E42 / #FACC15` | `#0284C7 / #C2570C / #B88A04` |

**Bands.** The hero, the Plans section and the footer are full-width "bands" with their own dark-rich palette in both themes:
- **Dark theme:** navy with gold, blue and emerald radial glows, plus a faint 56px grid masked to fade out.
- **Light theme:** the hero and Plans bands become an airy pearl wash. It runs from `#FBF8F1` to the page background `#F1F4FB`, with soft gold, sky and lilac glows and a faint grid, and fades into the page with no hard edge.
- **Dark "jewel" surfaces:** in the light theme, the report card, plan cards and comparison table keep the deep-navy palette. They are the dark focal points on a light page.
- **Footer:** deep navy in both themes.

**Rules:**
- Red and green appear only on signed changes, always with ▲ / ▼ and the sign.
- The report card has a gradient border (gold → blue → emerald).
- Section icons are gradient tiles in their section's colour.

### 1.15 Brand logo
Use the official FinLit logo: a figure climbing stairs inside "FinLit", a gold "Lit", and green rising bars with an arrow. Never redraw or restyle it.

- **Until the official SVG is supplied:** use `assets/finlit-logo-on-dark.png` on dark surfaces and `assets/finlit-logo-on-light.png` on light ones. In the light version the white ink becomes navy, and neither file has a background box.
- **When the official SVG arrives:** swap it in, with the ink colour driven by the theme.

### 1.2 Typography (Google Fonts)

- **Newsreader** (opsz 6–72): company name, section titles, the score numeral (weight 300), the thesis line. Never inside tables.
- **IBM Plex Sans** 400/500/600: all UI and numbers. It keeps continuity with the live site. Use `font-variant-numeric: tabular-nums lining-nums`.
- **IBM Plex Mono**: ticker, ISIN, keyboard hints only.
- Scale (px): 11 label · 13 meta · 15 body · 18 lede · 22 thesis · 30 section · 34 company name · 72–84 score. Labels: 11px, 600 weight, uppercase, `.12em` tracking. Gold eyebrows use `.16em`.
- Render ₹ at `.62em` in muted colour, and units (Cr, %, ×) at `.7em` in muted colour.

### 1.3 Surfaces and motion

- **Radii:** 12 for cards, 8 for tiles and inputs, 6 for buttons, 4 for tags.
- **Card surface:** a faint top-light gradient, a 1px border, and a layered shadow: `inset 0 1px 0 rgba(255,255,255,.045), 0 1px 2px rgba(0,0,0,.35), 0 18px 40px -20px rgba(0,0,0,.6)`, with a warm, lighter version in light mode.
- **Signature details:**
  - A gold hairline across the top of the report card.
  - Section folios ("§ 03" plus a 32px gold rule).
  - A gold underline under table headers.
  - The FinLit rising-bars mark as a ~4% watermark in the report card.
  - A 3.5% paper grain in light mode.
- **Motion** is 140–240ms ease-out, with one orchestrated moment on load:
  - The hairline draws.
  - The score arc sweeps 0→61 while the numeral counts up.
  - The pillar arcs and bars fill with a 70ms stagger.
- Under `prefers-reduced-motion`, show the final state immediately.

---

## 2. Information architecture: answer first, evidence next, raw data last

### 2.1 Top bar (sticky, blurred)
Contents: logo, site links (Screener, Research, Plans), ⌘K / Ctrl K search palette (company, ticker or ISIN, keyboard navigable, `role=dialog`, combobox semantics), the user's **plan badge**, and theme toggle. The prototype adds a "Preview Basic / Bronze / Gold" switch for demos; in production the plan comes from the user's subscription. A market ticker tape runs above the bar.

**Performance.** The prototype was measured scrolling the full page in headless Chromium:
- **Old version:** 95% of frames took up to 33–50 ms, which users feel as stutter.
- **Current version:** 95% of frames take 16.8 ms or less, a steady 60 fps.

Keep it that way:
- No `backdrop-filter` on sticky bars (use solid fills).
- No fixed full-screen overlays with blend modes.
- Never call `scrollIntoView` from a scroll-spy (scroll only the nav strip horizontally).
- Draw charts lazily, once, when they enter the viewport.
- Animate only `transform` and `opacity`.

### 2.2 Identity bar (compact, one line)
Company name (serif, 34px), ticker · exchange · sector · ISIN, last price with change and as-of time, a 52-week range micro-bar, and the buttons **Watchlist** (primary), **Download** (menu, see §4) and **Copy link**. Keep it compact so it never competes with the report.

### 2.3 Research Report card (the hero, above the fold)
One raised card, eyebrow "FinLit Research Report · FY26 · Consolidated", with "Updated · Source · How we score" on the right. Three columns:

1. **Engraved score dial.**
   - 100 tick marks.
   - A thin gold composite arc (61/100) with a knob at its end.
   - Five outer pillar segments of 72° each. Each fills to its pillar score and is coloured by verdict, with labels outside the ring.
   - Centre: the numeral, "out of 100", and the verdict.
   - Hovering a segment highlights its ledger row and the reverse. Clicking opens the row.
2. **Pillar ledger.** Five rows, each with name, evidence line (3 metrics), score (serif), bar and verdict. A row expands to a one-line plain-English explanation and 3 evidence tiles. Rows are buttons with `aria-expanded`.
3. **The read.** A thesis sentence in serif, 3 "what you need to know" points (✓ / !), and the disclaimer line.

A **key-numbers strip** runs along the bottom of the card: Market cap, P/E vs sector, P/B vs sector, EV/EBITDA vs sector, dividend yield, ROE, ROCE, D/E. Each has an ⓘ tip.

At 390px everything stacks in this order: identity, dial (~240px), ledger, read, KPIs in a 2-column grid.

### 2.4 Sticky section nav
Notes · Price · Financials · Ratios · Ownership · Peers · Business · Dividends · Method. Scroll-spy uses a gold underline on the active item. Once the report scrolls away, a mini identity appears on the left (`BPCL ₹293.45 +1.47% 61`).

### 2.5 Sections
1. **Research notes:** three columns, Strengths / Watch-outs / From the model, in plain language with real numbers.
2. **Price & momentum:**
   - **Price chart.** Ink line with a soft gold area. Volume and EMA 50 are in Basic; EMA 20/200 and other indicators are Gold. Ranges 1M–MAX. Mark the high and low. A crosshair tooltip shows date and close.
   - **Bad-tick filter:** drop any close ≤ 0, or more than 35% away from the median of its neighbours. Break the line at the gap and footnote it.
   - **Momentum card:** RSI gauge with 30/70 zones, the gaps to the 50- and 200-day averages, and price-return diverging bars.
3. **Financials:** tabs for P&L / Balance sheet / Cash flow. Each tab has highlight tiles, a one-line lede, a chart and the statement table (all Basic).
   - **P&L:** stacked panels sharing years. Revenue goes on top (₹ L Cr). Net profit and net margin go below. Never use a dual axis, since revenue is about 20× profit. Highlight the latest year in gold and annotate FY23 (0.4% margin) and FY26 (PAT +93.8%).
   - **Balance sheet:** stacked funding bars (equity + reserves, borrowings, other liabilities **net of borrowings**: the feed's "other liabilities" already includes borrowings).
   - **Cash flow:** diverging operating/investing/financing bars with a free-cash-flow line. Annotate "Record ₹50,769 Cr".
   - **Tables:**
     - sticky first column, latest column tinted gold, YoY column, sparkline column;
     - subtotal rows bold;
     - U+2212 minus;
     - Indian grouping;
     - "—" with a "Not reported" tooltip for gaps.
4. **Ratios:** grouped by pillar (Valuation, Quality & returns, Growth, Profitability, Balance sheet, Shareholder returns).
   - Each row shows the value with its unit and an ⓘ. A bullet bar compares it with the sector where a benchmark exists ("Cheaper than sector 15.49×").
   - Every row shows a plain-English definition. Extra rows are Gold (detailed ratio analysis).
   - Nine-year small multiples are Bronze (historical trend); the full nine-year table is Gold.
5. **Ownership:** two 100% stacked bars (a year ago against latest), plus four tiles with change chips in "pts".
6. **Peers:** real peer set. BPCL is pinned with a gold left rule. Market cap is shown with a log-scale size bar. Each row links to that stock.
7. **Business:** a plain-English description, consumer brands as tags, and key facts.
8. **Dividends:** horizontal timeline with amount, type and date. Never show raw Unix timestamps.
9. **Method & data:** how the score works (bands Strong ≥ 80, Healthy 65–79, Mixed 45–64, Weak < 45), source and as-of date, and a **data notes** list of feed inconsistencies.
10. **Footer:** "Have a question about your portfolio?" CTA and the SEBI/AMFI disclaimer.

**Dev metrics** are never shown on the public page. They are admin-only, in a drawer opened from the user menu.

---

## 2.6 Subscription plans and gating

Prices are not decided and the plans are subject to change. Gate content by plan rank: Basic (0) < Bronze (1) < Gold (2).

| Plan | Unlocks on the stock page |
|---|---|
| **Basic (free)** | Company overview, price chart with volume and moving averages, financial statements (P&L, balance sheet, cash flow, 5–10 years), key ratios with sector comparison, ownership, peers, dividends, method. Also includes the stock screener. |
| **Bronze** | Everything in Basic, plus the FinLit research score and five-pillar scorecard (hero report), scorecard notes, the detailed parameter view (pillar drawer), the historical trend for each parameter (nine-year small multiples), watchlist and portfolio tracking, and exports (PDF note, offline HTML, CSV). |
| **Gold** | Everything in Bronze, plus the advanced scoring model, detailed ratio analysis (15+ ratios, quarterly, full nine-year table), technical analysis and momentum score (RSI gauge, EMA 20/50/200 and other indicators), stock bucketing and key triggers, revenue segmentation, advanced peer insights, and priority access. |

**How locked content shows.** Locked content stays in the layout, dimmed (opacity .16 with a fade mask) and `inert`. Over it sits an unlock card containing:
- the tier badge, a title and three benefit bullets;
- an "Unlock with Bronze/Gold" button and a "Compare plans" link.

Never hide a locked section completely; the preview is the sales pitch. Small items, such as the extra ratio rows, collapse into a "+N more ratios · Gold" chip. Watchlist and Download show a lock icon on Basic and route to Plans.

**Plans section**, at the bottom of the stock page (also linkable as its own `/plans` route):
- A band with the headline "Invest smarter with *the right plan*" and a trust row: Data-driven insights · Proprietary scoring model · Reliable & transparent · Built for every investor.
- Three plan cards:
  - **Basic:** "Free" badge, sky blue.
  - **Bronze:** "Popular to get started" badge, bronze-orange.
  - **Gold:** "Most popular" badge, gold, visually featured.
- Each card has a tier icon, description, a price block ("Free" / "Soon · pricing at launch"), a CTA, an "Everything in X, plus" feature list, and a mini preview (Basic: included modules; Bronze: an example score ring and the five pillar icons; Gold: example stock buckets).
- Below the cards, a grouped comparison matrix (Research data · FinLit scoring · Deep analysis · Tools) with tier-coloured check marks.

## 3. Making it simple (UX rules)

1. **ⓘ on every metric.**
   - Hover/focus on desktop, tap on touch, Esc to close; the popover is positioned to stay inside the viewport.
   - Content: name, one-line definition, "how to read", formula.
   - Centralise all of it in one `metricGlossary.ts`. Replace every placeholder description ("Shows a company financial metric used to compar…").
2. **Plans replace Simple/Pro.** What each user sees is set by their plan (§2.6). Every plan keeps the plain-English definitions under ratios.
3. **Copy is research, not advice.** Say "Research summary", never "verdict/recommendation/analyst". Every page and export carries the not-SEBI-registered disclaimer.
4. **Indian formatting** goes through one utility:
   - ₹ with no space, never "Rs.";
   - en-IN grouping (1,27,313), and "L Cr" from 1 lakh crore upward;
   - multiples with "×", changes in "pts";
   - U+2212 minus;
   - FY labels (FY26 = Apr 2025–Mar 2026).
5. **Loading states:** skeletons shaped like the final layout, with no layout shift.
6. **Empty and error states:** an empty value is "—" with a tooltip. An error shows an inline retry card.
7. **Accessibility:**
   - WCAG AA in both themes;
   - full keyboard support, with visible gold focus rings;
   - each chart gets a text `aria-label` summary;
   - pillar bars use `role=meter`;
   - touch targets are at least 24px.
8. **Responsive:**
   - Design at 1440, 1180, 900, 720 and 390.
   - The page never scrolls horizontally; tables scroll inside their own container with a sticky first column.
   - Draw charts at their real container width (`ResizeObserver`), so text never scales down.

---

## 4. Downloads (offline reading)

A **Download** menu in the identity bar offers three formats.

1. **Research note (PDF).** This is a dedicated print layout, not a printout of the web page.
   - Build it in a hidden container on `beforeprint`. Switch to the light palette, redraw the charts into it at fixed width, then restore on `afterprint`. Use `@page { size: A4; margin: 14mm }` and give each page `break-after: page`.
   - Every page has a branded header and a footer with source, as-of date and disclaimer.
   - Pages:
     1. **Cover.** Name, price line, score dial, pillar table with bars and one-liners, the thesis, three key points, the key numbers grid.
     2. **Research notes and price.** The notes, a 1Y price chart, momentum and returns tables.
     3. **Profit & loss.** Chart plus the 6-year table.
     4. **Balance sheet and cash flow.** Both charts plus the cash-flow table with FCF.
     5. **Ratios and more.** Ratios by pillar, 9-year history, shareholding, peers, dividends, method.
   - If the team wants a one-click file instead of the print dialog, generate the same layout server-side (Puppeteer on the existing backend) at `/api/report/:ticker.pdf`.
2. **Offline copy (HTML).** A single self-contained file of the interactive page (inline data, CSS and JS) that opens without internet. Fonts fall back to Georgia and system sans.
3. **Data (CSV).** Score and pillars, P&L, balance sheet, cash flow, ratio history, ratio groups, shareholding, peers, dividends. Use a UTF-8 BOM so Excel shows ₹ correctly.

---

## 5. Data issues to surface (UI layer) and report to the backend

- The dividend return is 89.14% for every period from 3M to 5Y. Hide dividend and total-return figures until it is fixed.
- The 1Y return is −15.41% in one place and −16.2% in another. Use −15.41% and footnote the other.
- Dividend payout is 81.42% in one place and 38% in another. The FY26 dividends declared (₹17.50) over EPS ₹59.57 give 29%.
- Interest coverage is 8.38× in one place and 8.78× in another. From the statements, (PBT + interest) ÷ interest gives about 12.8×.
- ROE is 28.47% vs 25.8%, ROCE 17.3% vs 25.31%, OPM 7.88% vs 6.97%. State the basis (average vs closing equity) in the tooltip.
- "Other liabilities" includes borrowings, and the asset lines do not sum to total assets.
- DII + MF shows "+13.8" in one year. Check whether that is points or a relative %.
- Price history has bad ticks to 0 (Aug 2026). Filter them as in §2.5.
- Corporate actions show raw Unix timestamps.
- The dev metrics panel is visible to the public.

---

## 6. Engineering

- Build the design-system primitives first:
  - layout and data: `Card`, `Kpi`, `MetricRow`, `Verdict`, `Delta`, `SegmentedControl`, `Tabs`, `DataTable`, `Skeleton`;
  - overlays and menus: `InfoTip`, `Popover`, `Sheet`, `CommandPalette`, `DownloadMenu`;
  - charts and gauges: `Sparkline`, `ScoreDial`, `PillarLedger`, `BulletBar`.
  - Use Radix (or shadcn/ui if already present) for accessible overlays.
- Use one chart theme module that reads colours from the CSS variables, so charts follow the theme and print in the paper palette.
- Lazy-load below-the-fold sections and memoise table transforms.
- Use TypeScript strict and no `any` on API data.
- Keep existing routes and make the page serve a future public route `/stocks/:ticker`.
- Targets: Lighthouse Performance ≥ 90 and Accessibility ≥ 95 on desktop.

## 7. How to work

1. **Phase 0, audit:** stop and report the stack, the current component tree, the API types and your plan. Wait for my OK.
2. **Phase 1:** tokens, fonts, theme, primitives.
3. **Phase 2:** top bar, identity bar, Research Report hero, section nav, notes.
4. **Phase 3:** price, financials, ratios.
5. **Phase 4:** ownership, peers, Gold insights, business, dividends, method, plan gating and the Plans section.
6. **Phase 5:** downloads (PDF layout, offline HTML, CSV).
7. **Phase 6:** responsive, a11y, loading/empty/error states, both themes, Lighthouse.

After each phase:
- Run the app and take Playwright screenshots in dark and light at 1440 and 390 (plus the PDF at the end).
- Compare them against `stock.html` and this brief, and fix anything off.
- Commit per phase.

Finish with the list of backend and data issues.
