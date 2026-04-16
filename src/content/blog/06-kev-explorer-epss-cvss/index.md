---
title: "Joining KEV and EPSS at Build Time: a Static-Site Vulnerability Dashboard"
summary: "CVSS tells you how severe a CVE theoretically is. KEV tells you whether it's been exploited. EPSS tells you how likely it is to be exploited. Here's how I joined all three into a single React dashboard that deploys as a static site and refreshes itself nightly via GitHub Actions."
date: "Apr 12 2026"
draft: false
tags:
- React
- TypeScript
- Vite
- Security
- KEV
- EPSS
- NVD
- GitHub Actions
- Recharts
---

A CVSS score on its own is a bad triage signal. Every vulnerability management team learns this the hard way: you sort by CVSS, work top-down, and eventually realise you're spending a week patching a CVSS 9.8 theoretical buffer overflow nobody has ever exploited while the CVSS 7.5 deserialisation flaw with a working Metasploit module sits in the queue.

The industry has produced two complementary datasets to fix this, both public, both free:

- **CISA KEV** — the Known Exploited Vulnerabilities catalogue. A CVE is on this list if and only if CISA has evidence of active exploitation. It's a retrospective, authoritative signal. If it's on KEV, someone has been exploited by it.
- **EPSS** — the Exploit Prediction Scoring System from FIRST.org. A daily-updated probability (0 to 1) that each published CVE will be exploited in the next 30 days. It's forward-looking and probabilistic, built on machine learning over public exploit signals.

Together they give you a much sharper triage view than CVSS alone. KEV says "this one is real". EPSS says "these are the ones likely to become real". CVSS says "if it goes wrong, here's how bad it is". All three are useful. None is sufficient on its own.

The problem is nobody ships them joined. CISA publishes KEV as a JSON feed. FIRST publishes EPSS as a gzip-compressed CSV. They have no shared UI. If you want to see them side by side, you join them yourself.

So I did. Here's how KEV Explorer works, and why every interesting decision fell on the side of "move work out of the browser".

## The shape of the problem

Two datasets, two refresh cadences, two formats.

**KEV** is a ~1.5 MB JSON file at `cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json`. It updates whenever CISA adds an entry, typically a few times per week. Currently 1,559 entries. Each entry has the CVE ID, vendor, product, dates, ransomware association flag, and a required-action field.

**EPSS** is a ~3 MB gzip-compressed CSV at `epss.cyentia.com/epss_scores-current.csv.gz`. It updates daily with fresh model scores for every published CVE — around 250,000 rows. Each row is `cve, epss, percentile`.

The join is trivial: match on `cveID`. The complication is where to do it. Options:

1. **Client-side at runtime.** Ship the KEV JSON and the EPSS CSV as static assets, join them in the browser on load. Problems: EPSS CSV is gzip-compressed, browsers will decompress automatically if the server sets `Content-Encoding: gzip`, but the FIRST.org CDN sets `Content-Type` to `application/x-gzip` without the right `Content-Encoding` header. So the browser downloads the gzip bytes and you decompress with `pako`. Feasible but adds ~3 MB of data to the initial page load.
2. **API-side at request time.** Spin up a backend that fetches both and serves the joined result. Problems: it's a backend. Rate limits, hosting, cold starts, CORS. For a portfolio project this is infrastructure you do not want.
3. **Build-time ETL.** Do the join once, at deploy time, and ship a static JSON file with everything pre-joined. No runtime fetch, no backend, no decompression in the browser.

Option 3 is the one that makes the browser's job tiny. The dashboard loads a single JSON file and renders. Everything else is build infrastructure.

## The ETL script

`scripts/fetch-data.ts` runs in Node as part of the build. It's about 80 lines.

```typescript
import { writeFileSync, mkdirSync } from "node:fs";
import { inflate } from "pako";

const KEV_URL = "https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json";
const EPSS_GZ_URL = "https://epss.cyentia.com/epss_scores-current.csv.gz";

async function fetchEpssScores(): Promise<Map<string, { score: number; percentile: number }>> {
  const res = await fetch(EPSS_GZ_URL);
  const buf = new Uint8Array(await res.arrayBuffer());
  const csvText = new TextDecoder().decode(inflate(buf));
  const map = new Map<string, { score: number; percentile: number }>();
  const lines = csvText.split("\n");
  for (const line of lines) {
    if (!line.startsWith("CVE-")) continue;
    const [cve, epss, percentile] = line.split(",");
    map.set(cve.trim(), {
      score: parseFloat(epss),
      percentile: parseFloat(percentile),
    });
  }
  return map;
}

async function fetchKev(): Promise<KevEntry[]> {
  const res = await fetch(KEV_URL);
  const data = await res.json();
  return data.vulnerabilities;
}

async function main() {
  const [kev, epss] = await Promise.all([fetchKev(), fetchEpssScores()]);
  const today = new Date();
  const enriched = kev.map((entry) => {
    const epssMatch = epss.get(entry.cveID);
    const dueDate = new Date(entry.dueDate);
    return {
      ...entry,
      epss: epssMatch?.score ?? null,
      epssPercentile: epssMatch?.percentile ?? null,
      isOverdue: dueDate < today,
      addedMonth: entry.dateAdded.slice(0, 7),
      addedYear: parseInt(entry.dateAdded.slice(0, 4)),
    };
  });

  mkdirSync("public/data", { recursive: true });
  writeFileSync("public/data/kev.json", JSON.stringify({
    fetchedAt: today.toISOString(),
    catalogVersion: data.catalogVersion,
    entries: enriched,
  }));
}
```

`pako.inflate` handles the gzip decompression in Node. The CSV parse is deliberately naive — a CSV library would be overkill for a file whose format has been stable for years. The output gets stuffed into `public/data/kev.json` as a single file with a `fetchedAt` header so the UI can render "data as of" next to the KPIs.

On the latest run, 1,559 out of 1,559 KEV entries found an EPSS match. 100%. EPSS publishes scores for every NVD-assigned CVE, so unless a KEV entry references a reserved-but-not-published CVE, you get full coverage.

## Running the ETL nightly

The whole thing is deployed to GitHub Pages via a standard Actions workflow, and the schedule trigger does the heavy lifting:

```yaml
on:
  schedule:
    - cron: '0 6 * * *'
  push:
    branches: [master]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run fetch-data
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist
      - id: deployment
        uses: actions/deploy-pages@v4
```

06:00 UTC daily, GitHub runs the ETL, rebuilds the Vite bundle, and redeploys. The live dashboard is never more than 24 hours stale, and there is zero server infrastructure anywhere. GitHub Actions gives you 2000 free minutes a month on public repos; a nightly build takes about 45 seconds; the budget is not a concern.

## The signature chart: EPSS × days-overdue

The join makes a specific visualisation possible that neither dataset supports on its own: the EPSS × days-overdue scatter plot.

Every KEV entry has a `dueDate` — the date by which federal agencies are expected to have patched. Subtract today from that date and you get a "days overdue" number (negative means "due in the future", positive means "already past due"). Pair that with the EPSS score on the other axis and you get a view that says, for every active KEV, "how likely is this to be exploited *and* how long have we been sitting on it?"

The top-right quadrant — high EPSS, high days-overdue — is the panic list. These are vulnerabilities known to be exploited, predicted by the model to continue being exploited, and past the deadline for remediation. That quadrant is highlighted with a soft coloured background region so the eye goes straight to it.

Points are coloured by ransomware association — entries CISA has flagged as tied to ransomware campaigns are rendered differently, because a ransomware-associated KEV is a different threat model than a "generic" exploited CVE.

Recharts makes this straightforward. A `<ScatterChart>` with two `<ZAxis>` controls (one for size by `epssPercentile`, one for colour by ransomware) and a `<ReferenceArea>` for the high-risk quadrant. The data preparation is a single `.map()` off the filtered dataset:

```typescript
const scatterData = filteredEntries.map(entry => ({
  epss: entry.epss ?? 0,
  daysOverdue: daysBetween(entry.dueDate, new Date()),
  ransomware: entry.knownRansomwareCampaignUse === "Known",
  cveID: entry.cveID,
}));
```

Because every chart reads from the same filtered-entries hook, adjusting the vendor multiselect or the EPSS threshold slider updates the scatter, the KPIs, the vendor leaderboard, and the data table all at once, all consistent with each other. One filter state, many derivations.

## URL state as the sharing mechanism

The dashboard has no accounts, no saved views, no bookmarks UI. Instead, every filter writes to the URL query string:

```
?vendors=Microsoft,Apple&overdue=1&epssMin=0.5&search=spring
```

On mount, the filter state initialises from the query string. On every change, the query string is updated via `history.replaceState` (not `pushState` — we don't want every slider tick to add a browser history entry).

Sharing a filtered view is copy-paste the URL. The person who opens it sees exactly the same filters applied, and their dashboard is also filter-aware and shareable. It's the primitive version of saved views, and it does about 95% of the job without a backend or a user model.

The other thing this gets you is reproducibility in bug reports. "Look at the scatter with EPSS > 0.7 and vendor = Ivanti" is a link.

## Things I would do differently

**A filter-aware delta KPI.** I deliberately left this out because the KEV dataset doesn't have a natural "prior week" comparison built in (the data is cumulative, not snapshot-based). But it would be possible to add a second dataset in the ETL — yesterday's KEV snapshot archived on each run — and compute a filter-aware week-over-week change in the "Additions over time" chart. The rolling-8-day Parquet trick from [my previous post](/blog/05-two-tier-parquet-caching/) applies directly here.

**NVD enrichment at build time rather than lazily.** Right now CVSS scores come from the NVD API when the user opens the drill-down panel. Doing that in the ETL would let the scatter plot use real CVSS on the x-axis instead of days-overdue, and the table could sort by CVSS without a separate fetch. The tradeoff is NVD rate limits — even with an API key, enriching 1,500 CVEs takes a few minutes, and doing it nightly adds Actions runtime. A weekly full-enrichment with a daily incremental pass would be the right compromise.

**A DuckDB-wasm layer for client-side SQL.** At 1,559 rows the dataset is small enough that `.filter()` on the array is instant. If the dataset grew (e.g. joining with the full 250,000-row EPSS table for historical lookback) DuckDB-wasm would be the interesting option — ship a Parquet file, query it with real SQL in the browser. That's a portfolio project of its own.

## What this demonstrates

This is about as simple as a "real" project gets: two public datasets, one join, one static site, one GitHub Action. The interesting content is the domain knowledge — understanding *why* EPSS and KEV belong together, *what* quadrant of the scatter matters, *how* to frame the filter state so people can share specific views.

The technical stack is deliberately unflashy. React, TypeScript, Vite, Recharts, Tailwind. One Node script. One GitHub Actions workflow. No backend, no database, no auth. The production system fits on a single screen of code for every piece.

That's the point. The data model is the product. The engineering is the scaffolding.

The live dashboard is at [tamasczaban.github.io/kev-explorer](https://tamasczaban.github.io/kev-explorer/). The source is [github.com/TamasCzaban/kev-explorer](https://github.com/TamasCzaban/kev-explorer).
