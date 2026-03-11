---
title: "Manifest-Based Parquet Caching for a Streamlit Dashboard with Shifting Requirements"
summary: "SQL queries that take minutes. Stakeholder requirements that change weekly. No sprint planning. Here's the caching architecture I built to handle all three."
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

And every week, the director would add something new. A new data source. A new KPI. A new comparison. The requirements weren't stable — they were a live negotiation, revised at every weekly call.

I built this dashboard alone. My manager and the other team member don't work in Python, so there was no code review, no shared architectural decisions, no one to catch a bad design before it became expensive to change. The caching system had to be fast for users, easy to extend as requirements shifted, and debuggable by me alone when something went wrong.

Here's what I built.

## The two-tier architecture

The cache has two layers that serve different purposes:

**Tier 1 — Streamlit's in-memory cache (`@st.cache_data`)**: once a DataFrame is loaded into the app process, Streamlit holds it in memory. Any subsequent interaction — changing a filter, switching pages — returns the cached object instantly without touching disk or network. This is what makes the dashboard feel snappy during a session.

**Tier 2 — Parquet files on a shared network drive**: persistent across users and sessions. When a new process starts and Streamlit's memory cache is cold, the app checks for today's Parquet file before ever attempting SQL. This is what makes the dashboard fast across the team — the first user of the day populates the Parquet, and every subsequent user loads from disk rather than running the queries themselves.

The combined effect: SQL runs once per day at most, disk is hit once per session at most, and everything after that is served from memory.

### Manifest-based multi-file Parquet cache

Rather than a single monolithic Parquet file, the system uses multiple files — one per data domain — tracked by a central manifest. The manifest records what exists, when it was written, and its status.

```python
import streamlit as st
import pandas as pd
import json
from pathlib import Path
from datetime import date

CACHE_DIR = Path("//shared-drive/dashboard-cache")
MANIFEST_FILE = CACHE_DIR / "manifest.json"

def load_manifest():
    if MANIFEST_FILE.exists():
        return json.loads(MANIFEST_FILE.read_text())
    return {}

@st.cache_data
def load_domain(domain: str):
    """Tier 1: Streamlit holds this in memory after the first call."""
    manifest = load_manifest()
    today = str(date.today())
    entry = manifest.get(domain, {})

    if entry.get("date") == today and entry.get("status") == "ok":
        # Tier 2 hit — read from shared Parquet, Streamlit caches the result
        return pd.read_parquet(entry["path"])

    if entry.get("path") and Path(entry["path"]).exists():
        # Graceful degradation — today's file not ready, serve latest available
        return pd.read_parquet(entry["path"])

    # Both caches cold — run SQL, write Parquet, update manifest
    df = run_sql_query(domain)
    path = str(CACHE_DIR / f"{domain}_{today}.parquet")
    df.to_parquet(path)
    manifest[domain] = {"date": today, "path": path, "status": "ok"}
    MANIFEST_FILE.write_text(json.dumps(manifest, indent=2))
    return df
```

The `@st.cache_data` decorator is doing the tier-1 work: Streamlit hashes the function arguments and stores the return value in memory. The first call per session hits the Parquet. Every call after that — across reruns, filter changes, page navigations — returns the cached DataFrame from memory without any I/O.

**Graceful degradation**: if today's Parquet doesn't exist yet and a previous file is in the manifest, serve it. Users always get data. The UI surfaces a timestamp so stakeholders can see whether they're looking at today's numbers or yesterday's.

**File-specific troubleshooting**: each domain has its own file and manifest entry. You can see exactly which domain last wrote successfully, invalidate one file without affecting others, and trace failures to a specific data source. When you're debugging alone with no one to escalate to, this matters.

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

## Designing for shifting requirements

The manifest-based approach turned out to be the right call for a second reason beyond performance: the requirements never stopped changing.

Every week, the director wanted something new. A new data source, a new breakdown, a new KPI. In a system with a single monolithic cache, adding a new data domain means touching the cache logic for everything. With the manifest approach, adding a new domain is one new `load_domain("new_source", engine)` call and one new entry in the manifest. The existing domains are untouched and their cache files remain valid.

The architecture absorbed week-by-week requirement changes with minimal rework. That was unplanned — but it's the right outcome when you're building alone with no sprint structure and a director who treats the weekly call as a requirements session.

## The result

A dashboard that was previously unusable — three-minute load times, no historical data, no trend comparisons, no delta filtering — became a tool the team opens daily. Cold cache runs once in the morning. Every other load is warm. Individual domain failures are isolatable and recoverable. Historical lookback goes as far back as the cache files exist. Delta KPIs filter correctly because of the rolling Parquet.

The caching logic is about 80 lines of Python. The impact on usability is the difference between a prototype and a production tool that people actually depend on.
