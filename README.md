# RS Rating — IBD-Style Relative Strength

IBD-style Relative Strength percentile ranking (0–99) for ~6,000 US-listed stocks, recomputed every weekday via GitHub Actions and published as CSVs in this repo.

> Self-hosted mirror of [Fred6725/relative-strength](https://github.com/Fred6725/relative-strength) (itself derived from [skyte/relative-strength](https://github.com/skyte/relative-strength) via [maximbelyayev](https://github.com/maximbelyayev/relative-strength)). Full credit to those authors for the methodology and pipeline. This mirror exists so the data source is under my control — the main consumer is my trading dashboard, which reads the raw CSV URLs below.

**Differences from upstream:** output CSVs are committed directly into this repo (`output/`) instead of being pushed to a separate log repo, and the TradingView Pine Seeds publishing step is removed. The calculation itself is unchanged.

---

## Daily Outputs

Generated every weekday at 23:30 UTC (after US market close), ~15–25 min per run.

| File | Content |
|------|---------|
| [`output/rs_stocks.csv`](output/rs_stocks.csv) | Full dataset — all ~6,000 stocks |
| [`output/rs_stocks_1.csv`](output/rs_stocks_1.csv) | Percentile 50–99 (strongest half, renders on GitHub) |
| [`output/rs_stocks_2.csv`](output/rs_stocks_2.csv) | Percentile 0–49 |
| [`output/rs_industries.csv`](output/rs_industries.csv) | Industry-level rankings with constituent tickers |
| [`output/RSRATING.csv`](output/RSRATING.csv) | Percentile threshold series (TradingView seed format) |

### Raw URLs (for programmatic use / Google Sheets)

```
https://raw.githubusercontent.com/lssee003/relative-strength/main/output/rs_stocks.csv
https://raw.githubusercontent.com/lssee003/relative-strength/main/output/rs_industries.csv
```

Google Sheets — paste in `A1`:

```
=IMPORTDATA("https://raw.githubusercontent.com/lssee003/relative-strength/main/output/rs_stocks.csv",",",0)
```

---

## Methodology

```
RS Score (stock) = 40% × P3 + 20% × P6 + 20% × P9 + 20% × P12
RS Score (SPY)   = 40% × P3 + 20% × P6 + 20% × P9 + 20% × P12
Final RS         = (1 + RS Score stock) / (1 + RS Score SPY) × 100
```

`P3` = cumulative return over the trailing 3 months (63 trading days), `P6`/`P9`/`P12` likewise. The windows overlap, so the most recent quarter effectively carries the heaviest weight. Final RS values are then percentile-ranked cross-sectionally across the whole universe into a 0–99 score (99 = strongest).

The `1M/3M/6M_RS_Percentile` columns recompute the score with the price series truncated 1/3/6 months back, ranked against today's cohort — useful for spotting improving vs. deteriorating momentum.

**Universe:** all common stocks from [nasdaqtrader.com](https://www.nasdaqtrader.com/dynamic/symdir/nasdaqtraded.txt) (ETFs and test issues excluded; ~6,000 names across NYSE, NASDAQ, NYSE ARCA, BATS, AMEX). Requires ≥120 trading days of history, so recent IPOs are excluded. Benchmark: SPY.

**Industry rankings** (`rs_industries.csv`): mean RS of member stocks per industry (industries with ≥2 members), percentile-ranked; `Tickers` column lists members sorted strongest-first.

---

## Output Columns (`rs_stocks.csv`)

| Column | Description | Source |
|--------|-------------|--------|
| Rank | Overall rank (1 = strongest) | Calculated |
| Ticker | Stock symbol | NASDAQ list |
| Sector | Sector (Yahoo taxonomy, 11 sectors) | Yahoo Finance |
| Industry | Industry (Yahoo taxonomy, ~145 industries) | Yahoo Finance |
| Exchange | NYSE / NASDAQ / etc. | NASDAQ list |
| Relative Strength | Raw RS score | Calculated |
| Percentile | 0–99 percentile rank | Calculated |
| 1M/3M/6M_RS_Percentile | RS percentile 1/3/6 months ago | Calculated |
| Price | Last close | Yahoo Finance |
| MarketCap | Market capitalisation | Yahoo Finance |
| Float | Float shares | Yahoo Finance |
| ShortFloatPct | Short % of float (Yahoo updates ~2×/month) | Yahoo Finance |
| PctFrom52WkHigh | % distance from 52-week high (negative = below) | Calculated |
| AvgVol10/30/50 | Average daily volume over 10/30/50 days | Calculated |
| RevenueGrowth | Most recent YoY revenue growth | Yahoo Finance |

Note: Sector/Industry are Yahoo Finance's own classification, not GICS — thematic groupings (e.g. cybersecurity, AI) don't exist as industries here.

---

## Automation

Two workflows, both also runnable manually via `workflow_dispatch`:

- **`output.yml` — Generate RS Ratings**: weekdays 23:30 UTC. Downloads 18 months of daily candles for the full universe (batched `yfinance`), computes rankings, commits `output/*.csv` and the `data_persist/ticker_info.json` metadata cache.
- **`update_stocks.yml` — Weekly ticker info refresh**: Sundays 00:00 UTC. Refreshes sector/industry/market-cap metadata for the whole universe.

The daily commits keep GitHub's 60-day scheduled-workflow inactivity timer permanently reset.

---

## Known Issues (inherited from upstream)

- Yahoo close prices occasionally miss a very recent split — affected tickers' RS may be temporarily off.
- `ShortFloatPct` lags sources like Finviz (Yahoo updates it ~2×/month).
- `Float` is missing for some small caps.
- A handful of tickers per run may be skipped due to Yahoo rate limiting; no meaningful impact on the percentile distribution.
- Percentiles rank against *all* listed stocks, so the top buckets include illiquid micro-caps — filter by `MarketCap` / `AvgVol50` for tradable screens.

---

## Running Locally

Python 3.10+ (3.11 recommended):

```bash
git clone https://github.com/lssee003/relative-strength.git
cd relative-strength
pip install -r requirements.txt
python relative-strength.py true
```

Outputs land in `output/`. Config knobs (universe, benchmark, min percentile) are in `config.yaml`.
