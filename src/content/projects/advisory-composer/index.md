---
title: "Advisory Composer"
summary: "Browser tool that takes any lockfile — npm, pip, Go, Cargo, CycloneDX SBOM — queries OSV.dev and EPSS, and generates formatted security advisories for four channels in one pass: email drafts, Slack Block Kit JSON, GitHub PR comment markdown, and CSV exports. Pure client-side, zero backend, EPSS-weighted prioritisation."
date: "Apr 12 2026"
draft: false
tags:
- React
- TypeScript
- Automation
- Security
- OSV
- EPSS
- SBOM
demoUrl: "https://tamasczaban.github.io/advisory-composer/"
repoUrl: "https://github.com/TamasCzaban/advisory-composer"
---

A browser-based automation tool for converting a dependency manifest into formatted security advisories across every channel a defender needs to reach: email to an app owner, Slack notification for a team channel, GitHub PR comment for a dependency bump, CSV for the record. Paste a `package.json` or drag in a `go.mod`, click through a four-step wizard, and walk away with downloadable artefacts for every channel in one pass.

## The problem

Vulnerability disclosure within an engineering organisation is a multi-channel affair. The same advisory needs to reach:

- **App owners** — by email, with the subset of findings that applies to their service, usually as an attachment
- **Team channels** — as a Slack message with severity breakdown and inline details
- **Pull requests** — as a markdown comment explaining why a dependency is being bumped
- **The audit trail** — as a CSV appended to the incident record

Doing this manually for every scan result is slow and error-prone. The formatting differs between channels, the data is the same, and the mapping from "OSV JSON response" to "Slack Block Kit JSON" is pure transformation logic. Advisory Composer is that transformation, wrapped in a UI.

## The flow

### Step 1 — Upload

Drag-and-drop any supported manifest, or paste its contents into a textarea. Format auto-detected by filename and content sniffing:

- `package.json`, `package-lock.json` (v1/v2/v3)
- `requirements.txt`
- `go.mod`
- `Cargo.toml`
- CycloneDX SBOM JSON

A "sample data" button loads a fixture with a deliberately vulnerable set of packages (lodash 4.17.11, a few old Python packages) for quick demos.

### Step 2 — Scan + enrich

The parsed dependency list is sent to the OSV.dev batch API (`POST /v1/querybatch`) in chunks of 1000. OSV returns the vulnerability set for every package-version pair.

For every vulnerability with a CVE alias, a second call to the FIRST.org EPSS API fetches the exploit probability. Calls are cached in-memory to avoid duplicate lookups, and run 10-at-a-time concurrently to keep the scan responsive.

The results table shows each advisory colour-coded by severity, with package, version, ecosystem, advisory ID, severity, EPSS score, and summary. Sortable by any column. A KPI row at the top shows total advisories, critical count, packages affected, and highest EPSS observed.

### Step 3 — Configure outputs

Fields that get templated into the final outputs:

- Owner name or team name — used in the email greeting and Slack header
- Sender name — defaults to "Security Team"
- CC email — optional
- Severity threshold — filter down to CRITICAL, HIGH, MEDIUM, or ALL
- High-importance flag — adds `X-Priority: 1` to the `.eml`
- Draft-mode toggle — prepends `[DRAFT - DO NOT SEND]` to the subject line

Channels to generate are checkboxes; the user selects any subset. A live preview tab renders the current output for each selected channel as the configuration changes.

### Step 4 — Download

One button per format:

- **`.eml` file** — RFC 2822 compliant, HTML body with colour-coded severity table, CSV attachment with the filtered rows encoded as base64 MIME part
- **Slack Block Kit JSON** — header, divider-per-severity grouping, section block per advisory with fields for package/version/advisory/EPSS, footer with generated-by line; copy-paste straight into Slack's Block Kit Builder
- **GitHub PR comment markdown** — severity-grouped H3 headers, sortable table, collapsible `<details>` block per advisory with the full summary
- **CSV** — flat export, properly quoted, one advisory per row

A "copy to clipboard" button sits next to each download for quick paste workflows.

## Tech stack

| Layer | Technology |
|---|---|
| UI | React 18 + TypeScript (strict mode) |
| Build | Vite 5 |
| Styling | Tailwind CSS |
| Parsers | Pure TypeScript (zero runtime deps) |
| APIs | OSV.dev batch, FIRST.org EPSS |
| Deploy | GitHub Pages via `actions/deploy-pages` |
| State | React `useReducer` at App level |

## Engineering highlights

- **Six pure-TypeScript lockfile parsers** — `package.json`, `package-lock.json` (handles v1/v2/v3 schema differences), `requirements.txt` (comments, version operators, URL requirements), `go.mod` (require blocks), `Cargo.toml` (string and inline-table dependency formats), and CycloneDX SBOM (ecosystem derived from `purl` prefix). No runtime dependencies — the formats are simple enough to parse by hand, and avoiding `@yarnpkg/lockfile` and friends keeps the bundle small.
- **Ecosystem mapping** — OSV expects specific ecosystem names (`npm`, `PyPI`, `Go`, `crates.io`). The parser output is normalised at the boundary so downstream code is ecosystem-agnostic.
- **Batched + cached EPSS enrichment** — EPSS is looked up per CVE ID, which would be hundreds of round trips for a large scan. A module-scoped `Map` caches results across the session, and `Promise.all` batches run 10 concurrently to balance speed against being a good citizen of the FIRST.org API.
- **CVSS severity normalisation** — OSV reports severity in two places: a `severity` array with CVSS vectors, and a `db_specific` object with a normalised string. The scanner tries the array first and falls back to `db_specific`, handling cases where one source is missing.
- **RFC 2822 `.eml` generation** — no email library. The generator constructs the MIME multipart structure by hand: headers, HTML body, CSV attachment with `Content-Disposition: attachment`, boundary delimiters. Opens correctly in Outlook, Apple Mail, and Thunderbird.
- **Client-side download via Blob + object URL** — every output format is generated in the browser and offered as a `URL.createObjectURL()` download. No backend, no temporary file storage, no tokens.

## Design decisions

**Four-step wizard over a single page.** The flow has natural phase boundaries: you upload, you scan, you configure, you download. A wizard prevents premature configuration (you can't set severity thresholds before you know what severities are present) and gives clear progress feedback during the scan phase.

**Draft mode on by default.** The original inspiration — an internal tool that could send vulnerability emails directly through Outlook COM — learned the hard way that a "send" button is a sharp edge. This tool never sends anything; every output is a file or a clipboard payload. But the `[DRAFT]` prefix convention carries over as a safety reminder that the output is a template, not a finished message.

**No server, no state, no accounts.** The entire tool runs in the browser. Refresh the page and everything resets. Nothing is stored anywhere. For a tool that handles potentially sensitive dependency lists, this is a feature.

## Open source

Repository at [github.com/TamasCzaban/advisory-composer](https://github.com/TamasCzaban/advisory-composer). A companion to KEV Explorer, demonstrating the "automation" half of the security tooling portfolio alongside the "dashboard" half.
