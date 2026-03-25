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
  #sidebar       (icon nav: page-1 = trades, page-2 = analytics)
  #main          (flex column)
    .filter-bar  (shared across all pages)
    #pages
      #page-1    (trades table + headline metrics)
      #page-2    (Chart.js line charts: cumulative P&L, win rate, profit factor)
#edit-modal      (edit trade fields; open trades can have closeDate/exitPrice set here,
                  which promotes them from openTrades → allTrades)
#chart-modal     (candlestick chart popup; two panels side-by-side: main symbol + SPY)
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
