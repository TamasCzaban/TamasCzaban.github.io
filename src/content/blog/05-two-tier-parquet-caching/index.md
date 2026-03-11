---
title: "Building a 2-Tier Distributed Parquet Cache for a Slow SQL Dashboard"
summary: "When your SQL queries take minutes and your users won't wait, you need a caching layer. Here's the distributed Parquet pattern I engineered to make a Streamlit security dashboard actually usable."
date: "Mar 10 2026"
draft: false
tags:
- Python
- Pandas
- Parquet
- SQL
- Streamlit
- Performance
---

The dashboard worked. The SQL queries returned the right data. The problem was that "worked" meant waiting three minutes every time someone opened it.

The queries were unavoidable — they ran against large internal vulnerability databases, joining across multiple tables, aggregating thousands of records for network infrastructure assets. There was no shortcut in SQL that would get them under a few seconds. The data just took time to retrieve.

The question wasn't how to make the SQL faster. It was how to make the app feel fast regardless.

## The pattern: distributed cold cache with Parquet

The solution is a two-tier caching system. The cache lives on a shared network drive as Parquet files. The logic is simple:

```python
import pandas as pd
from pathlib import Path
from datetime import date

CACHE_DIR = Path("//shared-drive/dashboard-cache")
CACHE_FILE = CACHE_DIR / f"vuln_data_{date.today()}.parquet"

def load_data(engine):
    if CACHE_FILE.exists():
        # Warm cache — skip SQL entirely
        return pd.read_parquet(CACHE_FILE)

    # Cold cache — run queries, write for everyone else
    df = run_sql_queries(engine)
    df.to_parquet(CACHE_FILE)
    return df
```

**Warm cache**: if today's Parquet file exists on the shared drive, the app reads it directly. No SQL. Load time drops from minutes to under two seconds.

**Cold cache**: if the file doesn't exist yet, the app runs the full SQL queries and writes the result. This happens once per day — whichever user opens the app first pays the cost. Every subsequent user gets the warm cache.

The key insight is that the cache is distributed without any central coordination. There's no scheduler, no cache server, no background job. The app is self-maintaining. The shared drive is the coordination mechanism.

## Connection pooling for the cold cache path

When the cold cache is being populated, the SQL queries need to be fast. Connection pooling helps significantly here — rather than opening and closing a database connection for each query, a pool keeps connections alive and reuses them:

```python
from sqlalchemy import create_engine

engine = create_engine(
    connection_string,
    pool_size=5,
    max_overflow=2,
    pool_pre_ping=True  # drop stale connections automatically
)
```

On a multi-query load, this eliminates repeated authentication and handshake overhead. For long-running queries against an enterprise database, the saving is meaningful.

## Time-series Parquet for historical lookback

Beyond the daily cache, I needed trend data — how has the vulnerability count changed over time? The solution is a time-series Parquet store: each day's cache file is kept rather than overwritten, and the app presents a date picker populated with the available snapshot dates.

```python
available_dates = sorted([
    f.stem.split("_")[-1]  # extract date from filename
    for f in CACHE_DIR.glob("vuln_data_*.parquet")
], reverse=True)

selected_date = st.selectbox("Compare against", available_dates)
historical_df = pd.read_parquet(CACHE_DIR / f"vuln_data_{selected_date}.parquet")
```

No database queries for historical data. The Parquet files on disk are the time series. This makes the lookback feature essentially free once the cache exists.

## The delta filter problem — and the rolling 8-day Parquet

Here's where it got interesting.

The dashboard has global slicers — dropdowns that filter by asset group, severity, region. When a user selects a filter, every KPI and chart updates. That part is straightforward.

The delta KPIs are where it breaks. A "week-over-week change" metric needs to compare the current filtered count against the prior week's filtered count. If you store current and prior period as separate DataFrames and apply the filter to each independently, the delta is correct. But if you calculate the delta first and then filter, you get the wrong number — because the pre-filter delta reflects a different population than the post-filter current count.

The solution is a rolling 8-day pruned Parquet: a single file that contains today's data plus the 7 days prior, with only the columns needed for delta calculations. When the user applies a filter, it applies to this combined dataset simultaneously — so the delta is always calculated on the same filtered population as the current period.

```python
# Build the rolling Parquet (run daily alongside the main cache)
today = pd.read_parquet(CACHE_FILE)[delta_columns]
today["snapshot_date"] = date.today()

rolling_path = CACHE_DIR / "rolling_8day.parquet"
if rolling_path.exists():
    existing = pd.read_parquet(rolling_path)
    cutoff = pd.Timestamp(date.today()) - pd.Timedelta(days=7)
    existing = existing[existing["snapshot_date"] >= cutoff]
    rolling = pd.concat([existing, today], ignore_index=True)
else:
    rolling = today

rolling.to_parquet(rolling_path)
```

When a filter is applied, both the current-period and prior-period rows are filtered from the same DataFrame, ensuring delta calculations are always internally consistent.

## Why Parquet

Parquet is the right format for this pattern for a few reasons:

- **Columnar storage**: reading only the columns you need is fast, which matters when the rolling file has many columns but you only need a few for a given chart
- **Compression**: Parquet files are much smaller than equivalent CSVs, which matters on a shared network drive
- **Pandas native**: `pd.read_parquet()` and `df.to_parquet()` are one-liners with no serialisation surprises
- **Partition-friendly**: the date-in-filename pattern is simple but sufficient for this use case

## The result

A dashboard that was previously unusable — three minute load times, no historical data, no trend comparisons — became a tool the team actually opens daily. Cold cache runs once in the morning. Every other load is warm. Historical lookback goes back as far as the cache files exist. Delta KPIs filter correctly.

The caching logic is about 60 lines of Python. The impact on usability is the difference between a prototype and a production tool.
