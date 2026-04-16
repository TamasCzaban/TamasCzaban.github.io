---
title: "KEV Explorer"
summary: "Interactive dashboard over the 1,559 vulnerabilities in CISA's Known Exploited Vulnerabilities catalogue, enriched with EPSS exploit-probability scores. Build-time ETL joins KEV + EPSS nightly, filters URL-sync for shareable views, drill-down fetches live NVD data. Built with React, TypeScript, Recharts, and GitHub Actions."
date: "Apr 12 2026"
draft: false
tags:
- React
- TypeScript
- Recharts
- Security
- KEV
- EPSS
- NVD
- GitHub Actions
demoUrl: "https://tamasczaban.github.io/kev-explorer/"
repoUrl: "https://github.com/TamasCzaban/kev-explorer"
---

A public-facing portfolio piece that turns the CISA Known Exploited Vulnerabilities catalogue into something you can actually explore. KEV is the authoritative list of CVEs observed being exploited in the wild — 1,559 entries and counting — but the official JSON feed is a flat file with no UI. This dashboard is that UI: filter by vendor or ransomware association, overlay EPSS scores to see exploit probability alongside the "known exploited" fact, drill down to any single CVE for live NVD data.

## The problem

Two public feeds hold the critical information for vulnerability triage:

- **CISA KEV** — "this CVE has been exploited; patch by this date"
- **EPSS** (Exploit Prediction Scoring System from FIRST.org) — "this CVE has an N% probability of being exploited in the next 30 days"

They're complementary. KEV is retrospective and authoritative. EPSS is forward-looking and probabilistic. Together they give you a much better triage signal than either alone. But the two datasets live in different places, in different formats, updated on different schedules — so most defenders look at one or the other.

KEV Explorer joins them at build time and puts both dimensions on screen.

## Architecture

### Build-time ETL

A Node script (`scripts/fetch-data.ts`) runs on every build:

1. Fetches the KEV JSON feed from CISA
2. Fetches the EPSS daily CSV (gzip-compressed, ~3 MB) from `epss.cyentia.com`
3. Decompresses and parses the EPSS CSV with `pako`
4. Joins EPSS scores onto KEV entries on `cveID`
5. Computes derived fields (`isOverdue`, `addedMonth`, `addedYear`)
6. Writes the enriched dataset to `public/data/kev.json`

The output is ~1.2 MB of pre-joined JSON. The browser never does the join — it loads a single static file. This trades a bit of storage for instant dashboard startup and zero CORS headaches with the EPSS gzip endpoint.

### Nightly refresh via GitHub Actions

A scheduled workflow runs at 06:00 UTC daily:

```yaml
on:
  schedule:
    - cron: '0 6 * * *'
  push:
    branches: [master]
```

Each run re-fetches KEV and EPSS, rebuilds the site, and redeploys to GitHub Pages. The live dashboard is never more than 24 hours stale without any server infrastructure. 100% EPSS match rate on the latest catalogue — every CVE in KEV has a current EPSS score attached.

### URL-synced filter state

Every filter control — vendor multiselect, ransomware toggle, overdue toggle, date range, EPSS threshold — writes to the URL query string via `history.replaceState`. Sharing a link shares the exact filtered view. Reloading the page preserves state. Bookmarking specific cuts of the catalogue just works.

## Key features

- **Filter panel** — vendor multi-select with in-panel search, ransomware-only toggle, overdue-only toggle, year range sliders, EPSS minimum threshold, free-text search across CVE ID and vulnerability name
- **KPI tiles** — total filtered count, percentage overdue, ransomware-associated count, additions in the last 30 days
- **Additions over time chart** — stacked bar by month across the last 36 months, split by ransomware association
- **Top 15 vendors** — horizontal bar chart, filter-aware, updates as you adjust the panel
- **EPSS × days-overdue scatter** — the signature view, with a highlighted "high-risk quadrant" (high EPSS + overdue) and ransomware entries coloured separately
- **Due-date histogram** — distribution of days until or past the CISA due date, red for overdue, green for upcoming
- **Paginated data table** — 50 rows per page, sortable columns, EPSS bar sparklines inline, overdue badges
- **Drill-down panel** — slide-in from the right, fetches live NVD data for CVSS score (rendered as a gauge ring) and description, shows timeline, required action, CWEs, and external links

## Tech stack

| Layer | Technology |
|---|---|
| UI | React 18 + TypeScript (strict mode) |
| Build | Vite 5 |
| Styling | Tailwind CSS (dark theme) |
| Charts | Recharts |
| CSV parsing | Custom, gzip via `pako` |
| Data sources | CISA KEV JSON, FIRST.org EPSS CSV, NVD API v2 |
| Deploy | GitHub Pages via `actions/deploy-pages` |
| Refresh | GitHub Actions scheduled cron |

## Engineering highlights

- **Gzip CSV fetch in Node** — the EPSS daily feed is gzip-compressed CSV. The ETL decompresses it in memory with `pako` rather than writing to disk, keeping the build step clean and dependency-light. A fallback to the uncompressed FIRST.org paginated API is in place in case the gzip endpoint changes content-type or CORS headers.
- **Manual chunk splitting** — `vite.config.ts` splits `vendor-react` and `vendor-recharts` into separate chunks. The main app bundle is ~46 KB; vendor chunks cache separately across deploys.
- **Filter-aware chart updates** — every chart reads from the same `useFilteredData` hook. A single filter change re-derives all charts, KPIs, and the table from the same filtered dataset, guaranteeing internal consistency.
- **Lazy NVD enrichment** — CVSS scores and NVD descriptions are only fetched when the user opens the drill-down panel, not at build time. This keeps the bundle small and avoids hammering the NVD API at every rebuild. Results are cached per CVE ID within the session.
- **URL state preservation** — filter state serialises to a compact query string and deserialises on mount. Direct links to specific filtered views are the primary sharing mechanism — no backend session storage.

## Design decisions

**Dark theme.** The data is security-adjacent and the target reader is a defender. A dark slate background with a single accent colour (`#22c55e`) for highlights reads as "operations tool" rather than "marketing site". It also plays well with the red-for-overdue, green-for-upcoming convention used in the due-date histogram.

**Static JSON over API.** The entire enriched dataset is ~1.2 MB gzipped. Shipping it as a static file means no server, no rate limits, no API keys, no cold-start lag. The nightly Action gives the user a daily-fresh dataset for free.

**No authentication anywhere.** Everything is public data. Everything is shareable. The URL query string is the only state.

## Open source

Repository at [github.com/TamasCzaban/kev-explorer](https://github.com/TamasCzaban/kev-explorer). Built as a portfolio project demonstrating applied vulnerability management domain knowledge paired with a modern React/TypeScript frontend.
