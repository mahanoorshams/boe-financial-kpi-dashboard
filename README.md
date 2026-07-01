# Bank of England Macroeconomic Dashboard

A two-page Power BI dashboard tracking key UK macroeconomic indicators — mortgage lending, consumer credit, and broad money supply — using publicly available Bank of England data.

Built as a portfolio piece to show macro analysis skills alongside the [NHS RTT dashboard](https://github.com/mayasyed/powerbi-dashboard-builder).

---

## Dashboard Overview

**Page 1 — Mortgage & Bank Rate**

- Current Bank Rate: 3.75%
- Total UK mortgage outstanding: £1,713 bn
- Fixed vs. variable mortgage rate trends from 2015 Q1 onwards

**Page 2 — Consumer Credit & Money Supply**

- Consumer credit: pre-COVID peak £225.3 bn, COVID contraction −£28.6 bn, latest outstanding ~£254 bn
- M4 broad money supply: COVID surge £303.1 bn, latest outstanding £3,028 bn
- Consumer credit YoY growth (showing GFC and COVID contractions)
- Normalised growth index comparing consumer credit, M4, and mortgage stock since 2007

---

## Data Sources

All data from the [Bank of England Statistical Database](https://www.bankofengland.co.uk/statistics) — no credentials or API keys required.

| Table | Series | Frequency | From |
|---|---|---|---|
| `bank_of_england_database` | Mortgage outstanding (£ bn) | Monthly | 2016 |
| `bank_rate_monthly_clean` | Official Bank Rate (%) | Monthly | 1975 |
| `mlar_mortgage_rates_clean` | Fixed vs. variable rates by lender type | Quarterly | 2007 Q1 |
| `consumer_credit_historical_clean` | Consumer credit outstanding (£ bn) | Monthly | Feb 1994 |
| `m4_historical_clean` | M4 broad money supply (£ bn) | Quarterly | Mar 1998 |
| `boe_monthly_snapshot_clean` | Latest CC & M4 snapshot | Monthly | — |

---

## Stack

- Power BI Desktop — PBIP project format (PBIR JSON visuals, TMDL semantic model)
- Power Query M — data load and type coercion for BoE's legacy CSV formats
- DAX — all KPI measures, index calculations, and YoY growth

---

## Problems & How I Solved Them

The interesting part of any dashboard build is not what the finished product looks like — it's the decisions made when things didn't work as expected.

### 1. TMDL rejects `"£"` inside format strings

**Problem**: To display values as `£1,713 bn`, the natural DAX format string is `"£"#,0 bn`. But TMDL (Power BI's text-based semantic model format) treats any format value beginning with `"` as a double-quoted string. The closing `"` of `"£"` at position 3 is then treated as an unescaped quote marker, producing:

```
InvalidValueFormat: '"£"#,0 bn" is expected to be an escaped string quoted with '"',
but in position 2 it has an un-escaped quote marker!
```

**Fix**: Backslash-escape the pound sign — `\£#,0 bn;-\£#,0 bn;\£#,0 bn`. The leading `\` prevents TMDL from entering double-quoted string mode, so the entire value is parsed as a raw (unquoted) format string. This is not documented anywhere obvious; it took tracing the exact TMDL parse error to find it.

---

### 2. `labelSuffix` silently does nothing on the new card visual type

**Problem**: PBI's newer `cardVisual` type does not support the `labelSuffix` property, even though it exists in the underlying JSON schema. Setting `"labelSuffix": {"expr": {"Literal": {"Value": "' bn'"}}}` in the visual JSON has zero effect. The Format pane confirms this: searching for "suffix" returns "Formatting options are not found."

**Fix**: Used `customFormatString` in the `value` array's **metadata** selector within the visual JSON, scoped to the specific measure. This overrides the display format for that measure on that card only, without changing the measure definition itself.

---

### 3. Visual-level filters in PBIR require PBI's internal query language — and it fails silently

**Problem**: Three charts had data starting before my intended date range: Consumer Credit outstanding from 1990 (data starts 1994), M4 from the early 1980s (data starts 1998), and mortgage rates from 1992 Q1 (I only wanted 2015 onwards). I needed visual-level filters to trim the x-axis without affecting other visuals.

**How it works in PBIR**: Filters live in a `filterConfig` object at the root of each visual.json — *not* inside `visual`. The filter structure wraps PBI's internal query language: a `From` clause with entity alias, a `Where` clause with `Comparison` objects, and integer column values must use the `L` suffix (`"1994L"`, not `"1994"`). Text column comparisons use single-quoted literals (`"'2015 Q1'"`).

**The silent failure problem**: If any part of the structure is wrong (wrong alias, wrong value format, misplaced field), the filter simply doesn't apply. No error is shown in the UI. The only way to debug this was to compare working filter JSON (from PBI's own saved state) against my handwritten version.

---

### 4. CC and M4 date columns loading as strings broke all relationships

**Problem**: The BoE's historical CC and M4 CSV files use a UK date format (DD/MM/YYYY). Power Query loaded the Date column as text, which broke the relationship to Date_Table. This meant all time-based measures returned blank — but the charts showed a flat line rather than an error, making it hard to spot.

**Fix**: In Power Query, explicitly set `type date` with a `"en-GB"` locale culture. Also added `annotation UnderlyingDateTimeDataType = Date` to the TMDL column definitions to match what PBI expects when creating relationships.

---

### 5. M4 data is quarterly, but Date_Table is daily — average vs. sum matters

**Problem**: M4 outstanding is a stock variable reported at end-of-quarter. Aggregating with `SUM` over a year would sum four quarterly values and give a number roughly 4× the true outstanding balance. A measure like `SUM(m4_historical_clean[M4_Outstanding_GBP_bn])` for 2023 would return approximately £11,000 bn instead of £2,800 bn.

**Fix**: All outstanding balance measures use `AVERAGE` rather than `SUM`. For the index measures (comparing against 2007 baseline), I take the average over the whole base year — which for quarterly M4 data averages across four end-of-quarter readings. This is analytically cleaner than taking January only (which for M4 would be blank, since there's no January data in a quarterly series).

---

### 6. Indexed Growth chart: mortgage line starts at 130 in 2016, not 100

**Problem**: All three series in the Indexed Growth chart use 2007 as the base year (= 100). Mortgage data only exists from 2016 onwards, so the mortgage line starts at ~130 — because UK mortgage stock was roughly 30% higher in 2016 than in 2007. This is correct, but it looks like a miscalibration compared to the consumer credit and M4 lines which both start from 2007 at 100.

**The decision**: Changing the mortgage base year to 2016 would make all three lines start at 100 — but it would destroy the point of the chart, which is to compare *relative growth trajectories* from a common baseline. If the mortgage line starts at 100 in 2016, you lose the 2007–2016 context for what was happening with household debt while CC and M4 were being tracked.

**Fix**: Kept the 2007 base year consistent across all three series. Added a chart subtitle — "Mortgage data from 2016 only" — to signal to readers that the mortgage line's starting position (~130) is intentional, not an error.

---

### 7. PBI doesn't re-read files from disk during an open session

**Problem**: Power BI caches the project's in-memory state when a .pbip project is loaded. Editing a visual.json file on disk and then using File → Recent has no effect — PBI serves the in-memory version, not the updated file. This meant testing JSON changes required a full application restart each time.

**Fix**: Force-kill the PBI process in PowerShell (`Stop-Process -Name "PBIDesktop" -Force`) before reopening the project file. This bypasses any auto-save and guarantees that all visual JSON, TMDL, and theme files are read fresh from disk on the next open.

---

## How to Open

Requires **Power BI Desktop** (July 2024 or later, for PBIR support).

Open `BoE dashboard.pbip` directly — all data loads via embedded Power Query connections to the BoE's public data files. No credentials required.
