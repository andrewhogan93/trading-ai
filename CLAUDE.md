# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A single-file HTML trading dashboard (`dashboard.html`) that parses Schwab account statement CSV exports and visualises trade history. No build step, no dependencies to install — open the file directly in a browser.

## Development workflow

- **Run locally:** Open `dashboard.html` in a browser. Use a file like `account_statements_data/2026-03-25-AccountStatement.csv` to test by uploading via the UI.
- **After every change:** Commit with a descriptive message and push to `origin/master`. Never stage `account_statements_data/` — it contains personal trading data.

```bash
git add dashboard.html
git commit -m "description of change"
git push
```

## Architecture

Everything lives in `dashboard.html` — one HTML file with embedded `<style>` and `<script>`. External dependencies load from CDN:
- **flatpickr** — date range picker
- **Chart.js** — line charts on the analytics page
- **TradingView Lightweight Charts v4** (`unpkg.com/lightweight-charts@4`) — candlestick charts in the trade popup. Pinned to `@4`; v5 removed `addCandlestickSeries`.

### Data flow

```
CSV upload (FileReader)
  → parseCashBalanceSection()     isolates the Cash Balance section from the multi-section Schwab CSV
  → parseLegDescription()         regex-parses each trade row into a typed leg object
  → matchTrades()                 FIFO queue matching: pairs open/close legs into roundtrip trades
  → mergeSubs()                   collapses all partial fills for one position lifecycle into a single row
  → state.allTrades               array of completed trade objects (closed)
  → state.openTrades              array of open/unmatched trade objects (_isOpen: true)
  → refresh()                     applies filters → sorts → renders metrics, table, and charts
```

### Trade matching logic (critical)

Trades are **individual rows** in the CSV (each buy and sell is separate). The matching algorithm:
- Maintains a per-symbol FIFO queue
- Assigns a `tradeGroupId` when a position opens (queue empty → non-empty)
- All sub-trades share the same `tradeGroupId` until the queue drains back to zero
- This means scale-ins and partial exits all collapse to **one table row** per position lifecycle

Option contracts use a composite position key (`SYMBOL_STRIKE_PUTCALL_YYYY-MM-DD`) to avoid collisions between different contracts on the same underlying.

`EXP` rows (option assignment/expiry) are included in parsing via `ASSIGN_RE = /^(BOT|SOLD)\s+([\d.]+)\s+([A-Z0-9]+)\s+UPON\b/i`. Price is derived as `Math.abs(leg.amount) / leg.qty`.

Trades from before the statement period (e.g. positions opened in a prior CSV) will appear as open trades with no entry data — they cannot be auto-resolved without uploading earlier statements. Use the edit modal to manually close them.

### App state

```js
state = {
  allTrades,       // closed trades
  openTrades,      // open/unmatched trades (_isOpen: true)
  startBalance, minDate, maxDate,
  sortKey,         // default: 'openDate'
  sortDir,         // default: 'desc' (newest first)
  filters: { dateFrom, dateTo, type, side, statuses: Set }
}
```

`refresh()` is the single re-render entry point — call it any time filters or sort state change. Metrics and analytics only operate on closed trades; open trades appear in the table only when the "Open" status filter is active.

State is persisted to `localStorage` under key `trading_dashboard_v1`. Polygon API key stored separately under `polygon_api_key`.

### Page structure

```
#upload-screen   (shown until CSV is loaded)
#app             (flex row)
  #sidebar       (icon nav: page-1 = trades, page-2 = analytics, page-3 = market intelligence)
  #main          (flex column)
    .filter-bar  (shared across all pages)
    #pages
      #page-1    (trades table + headline metrics)
      #page-2    (Chart.js line charts: cumulative P&L, win rate, profit factor;
                  rule adherence cards for SPY rule and VWAP rule)
      #page-3    (Market Intelligence: SPY D1 chart + Claude AI analyst chat)
#edit-modal      (edit trade fields; open trades can have closeDate/exitPrice set here,
                  which promotes them from openTrades → allTrades)
#chart-modal     (candlestick chart popup; two panels side-by-side: main symbol + SPY)
#sync-modal      (GitHub sync: PAT input, pull/push/clear buttons)
```

### Key parsing notes

- The Schwab CSV has multiple sections separated by blank lines. Only the `Cash Balance` section is parsed.
- `REF #` values are formatted as `="123456"` (Excel formula wrappers) and must be stripped.
- `AMOUNT` and `BALANCE` fields use quoted comma-separated numbers like `"27,616.92"` — requires a custom `parseCSVLine()` tokeniser, not `split(',')`.
- Starting account balance comes from the first `BAL` row.
- Options are identified by presence of expiry/strike/PUT/CALL in the description field.

### Chart modal

Clicking a trade row opens a candlestick chart popup powered by Polygon.io (requires a free API key, stored in localStorage).

**Data source:** `polygonFetch(symbol, mult, span, from, to, apiKey)` calls `api.polygon.io/v2/aggs/ticker/...`. After-hours candles are stripped client-side via `isRegularHours()` (keeps 9:30–16:00 EST, Mon–Fri only); daily-interval charts are not filtered.

**Interval selection based on hold time:**
| Hold time | mult | span     |
|-----------|------|----------|
| < 1 day   | 5    | `minute` |
| 1–7 days  | 1    | `hour`   |
| > 7 days  | 1    | `day`    |

**Chart layout:** Two panels rendered side-by-side — `#chart-main` (traded symbol) and `#chart-spy` (SPY for context). Both use `makeChart(container, candles, markers, visibleFrom, visibleTo)` which creates a LightweightCharts instance with the dashboard's dark theme and EST time labels via `localization.timeFormatter`.

**Day separator lines:** `addDayLines(chart, container, candles)` overlays absolutely-positioned `<div>` elements at each EST day boundary. Lines reposition on scroll/zoom via `subscribeVisibleTimeRangeChange`. Only called for intraday intervals.

**Entry/exit markers:**
- Long: entry = green arrowUp below bar, exit = red arrowDown above bar
- Short: entry = red arrowDown above bar, exit = green arrowUp below bar
- SPY chart shows the same markers (without price labels) so timing context is visible
- Open trades: only an entry marker, no exit

**Chart lifecycle:** `chartInstance` and `spyChartInstance` are module-level refs; always call `.remove()` before re-creating to avoid memory leaks. Container `.innerHTML = ''` is cleared before each new chart render.

**`_fp` fingerprint:** Each trade has a `_fp` field used to look it up across `allTrades` and `openTrades` when opening the edit or chart modal.

### Metric cards (page 1 headline)

- **Account P&L** — total dollar P&L + avg return % subtitle. Avg uses `Math.abs()` so wins and losses don't cancel (e.g. -1% and +1% average to 1%, not 0%).
- **Avg Cost** — average position cost per trade (replaces the old "Net P&L" card).
- Win Rate, Avg Hold, Trade Count cards remain unchanged.

### Rule adherence (page 2)

`computeRuleStats(trades, cache)` splits closed trades into followed/violated buckets for the SPY rule and VWAP rule based on `rulesCache` entries. Two cards rendered at the bottom of page 2, each showing trade count, win rate, avg return %, and total P&L per bucket.

### GitHub sync

Annotations (trade setups + rule results cache) are persisted to a private GitHub repo (`andrewhogan93/trading-ai`) as `annotations.json` via the GitHub Contents API.

- **Token:** Fine-grained PAT with Contents: Read & Write, stored in `localStorage` under `gh_pat`.
- **SHA tracking:** `_ghSha` module-level ref; always fetched before first PUT to avoid 409 conflicts.
- **Auto-load:** `showDashboard()` pulls from GitHub if a token exists.
- **Auto-save:** `scheduleGithubSave()` — 3s debounce, called from `saveRuleResult` and `saveEditModal`.
- **`☁ Sync` button** in the header opens `#sync-modal` for manual pull/push/clear.

### Market Intelligence (page 3)

**SPY D1 chart** — `loadIntelSpyChart()` fetches ~400 days of daily candles via `polygonFetch('SPY', 1, 'day', ...)`, renders with `makeChart()`, then overlays SMA lines directly on `intelSpyChart` (not via `makeChart` — those are added after). SMA colors: SMA50 = blue, SMA100 = orange, SMA200 = pink.

**`computeSMA(candles, period)`** — plain loop SMA from candle closes; returns `[{time, value}]`.

**Claude context (`intelSpySummary`)** — rebuilt on every chart load; includes current SMA50/100/200 values and a 60-row candle table with per-row SMA columns. Passed as part of the user message on every API call.

**AI chat** — `callClaude(text)` and `sendIntelMessage(text)` both call `https://api.anthropic.com/v1/messages` directly from the browser (`anthropic-dangerous-direct-browser-access: true` header required). Model: `claude-sonnet-4-6`, max tokens: 1500. Anthropic API key stored under `anthropic_api_key` in localStorage.

**System prompt** — includes full text of 13 PDFs from `The system/Long term Market Analysis/`, pre-baked into `market_docs.js` as `window.MARKET_DOCS_TEXT`. Loaded via `<script src="market_docs.js">` — no upload required.

**`market_docs.js`** — generated file (~85KB). To regenerate: run `pdftotext` (found at `C:\Program Files\Git\mingw64\bin\pdftotext.exe`) on each PDF, wrap output in a JS template literal as `window.MARKET_DOCS_TEXT = \`...\``. Never commit `docs_extracted.txt` (intermediate file).

**Chat history** — `intelHistory = [{role, content}]`, module-level, not persisted. `renderPage3()` auto-fires initial analysis when `intelHistory.length === 0 && intelSpySummary` is set. Clear Chat button resets history and re-triggers analysis.

### Hold time / fingerprint matching

`_fp` is derived from trade fields including exact open/close timestamps. An old `parseDateTime` bug (minutes always = 0) baked wrong timestamps into fingerprints stored in localStorage. On CSV re-upload, `loadCSV` runs a fuzzy fallback: if exact `_fp` doesn't match, falls back to `symbol|openDate.toDateString()|side` key (with collision detection — if two trades share the same fuzzy key, neither is updated).
