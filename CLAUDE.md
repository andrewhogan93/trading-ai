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

### Data flow

```
CSV upload (FileReader)
  → parseCashBalanceSection()     isolates the Cash Balance section from the multi-section Schwab CSV
  → parseLegDescription()         regex-parses each trade row into a typed leg object
  → matchTrades()                 FIFO queue matching: pairs open/close legs into roundtrip trades
  → mergeSubs()                   collapses all partial fills for one position lifecycle into a single row
  → state.allTrades               array of completed trade objects
  → refresh()                     applies filters → sorts → renders metrics, table, and charts
```

### Trade matching logic (critical)

Trades are **individual rows** in the CSV (each buy and sell is separate). The matching algorithm:
- Maintains a per-symbol FIFO queue
- Assigns a `tradeGroupId` when a position opens (queue empty → non-empty)
- All sub-trades share the same `tradeGroupId` until the queue drains back to zero
- This means scale-ins and partial exits all collapse to **one table row** per position lifecycle

Option contracts use a composite position key (`SYMBOL_STRIKE_PUTCALL_YYYY-MM-DD`) to avoid collisions between different contracts on the same underlying.

### App state

```js
state = {
  allTrades, openPositions, startBalance, minDate, maxDate,
  sortKey, sortDir,
  filters: { dateFrom, dateTo, type, side, statuses: Set }
}
```

`refresh()` is the single re-render entry point — call it any time filters or sort state change.

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
```

### Key parsing notes

- The Schwab CSV has multiple sections separated by blank lines. Only the `Cash Balance` section is parsed.
- `REF #` values are formatted as `="123456"` (Excel formula wrappers) and must be stripped.
- `AMOUNT` and `BALANCE` fields use quoted comma-separated numbers like `"27,616.92"` — requires a custom `parseCSVLine()` tokeniser, not `split(',')`.
- Starting account balance comes from the first `BAL` row.
- Options are identified by presence of expiry/strike/PUT/CALL in the description field.
