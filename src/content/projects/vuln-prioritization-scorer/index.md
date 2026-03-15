---
title: "Vulnerability Prioritization Scorer"
summary: "Streamlit app that enriches CVE lists with live NVD (CVSS v3) and FIRST EPSS (exploit probability) data, scores via a configurable 4-factor composite formula (CVSS + EPSS + exposure tier + age decay), and exports ranked HTML/CSV reports."
date: "Mar 15 2026"
draft: false
tags:
- Python
- Streamlit
- Security
- Pandas
- Plotly
- NVD
- EPSS
demoUrl: "https://vuln-prioritization-scorer-tams-czaban.streamlit.app/"
repoUrl: "https://github.com/TamasCzaban/vuln-prioritization-scorer"
---

A Streamlit-based tool for security teams to enrich raw CVE lists with live NVD and EPSS data, then rank vulnerabilities using a configurable multi-factor composite scoring formula. Designed to cut through alert fatigue and surface the vulnerabilities that genuinely need patching first.

## The Problem

Raw vulnerability feeds from scanners hand you a list sorted by CVSS. But CVSS measures theoretical severity, not operational risk. A CVSS 9.8 with a 0.05% exploit probability sitting on an isolated internal host is not the same priority as a CVSS 7.2 with a 60% EPSS score on an internet-facing API gateway. This tool adds the missing dimensions — exploit probability, asset exposure, and vulnerability age — and lets the security team tune the formula to match their environment.

## Architecture

The app is organised into discrete layers that can be used and tested independently:

- **`api/`** — `NVDClient` (rate-limited, paginated) and `EPSSClient` (batch fetch, 100 IDs per request), with a session-state cache so weight changes don't trigger re-fetches
- **`core/`** — scoring engine (`scorer.py`), per-factor normalizers (`normalizer.py`), and age decay logic (`age_decay.py`)
- **`ui/`** — CSV uploader/validator, sidebar weight sliders, and results table with tabs
- **`export/`** — CSV exporter and Jinja2-templated self-contained HTML report with embedded Plotly chart

## Scoring Formula

```
composite_score = w_cvss * N_cvss + w_epss * N_epss + w_exposure * N_exposure + w_age * N_age
```

| Factor | Default Weight | Normalization |
|---|---|---|
| CVSS v3 | 0.35 | `cvss / 10.0` |
| EPSS | 0.30 | `sqrt(epss_probability)` — square-root stretch |
| Exposure tier | 0.20 | Ordinal map (internet-facing=1.0, critical-infra=0.9, unknown=0.6, internal=0.5) |
| Age decay | 0.15 | `log1p(age_days) / log1p(365*3)`, capped at 1.0 |

All weights are auto-normalized to sum to 1.0 as sliders are adjusted. Results are banded into Critical (≥ 0.75), High (≥ 0.55), Medium (≥ 0.35), and Low (< 0.35).

### Why square-root EPSS?

EPSS scores are heavily right-skewed — most CVEs cluster near zero with a long tail of high-probability entries. A linear mapping compresses that tail and makes top-of-list differentiation disappear. Square-root stretches low scores upward and keeps the formula sensitive to the difference between 1% and 60% exploit probability.

### Why log age decay?

Vulnerability age matters as a proxy for patch availability and attacker awareness, but the marginal urgency of a 3-year-old CVE vs a 4-year-old one is negligible. Log scaling captures the meaningful early decay (new vs 6-months-old) while preventing ancient CVEs from dominating the score.

## API Integration

- **NVD API v2** — fetches CVSS v3 base score and publication date per CVE ID. Without an API key, NVD enforces 5 requests per 30 seconds; with a free key (sidebar input or `.env`), the limit rises to 50/30s. The client handles rate limiting and 404s (CVE not found) gracefully.
- **FIRST EPSS API** — CVE IDs are chunked into batches of 100 and sent as comma-separated query parameters, reducing round trips from N to N/100. Missing CVEs default to EPSS=0.

## Tech Stack

| Layer | Technology |
|---|---|
| UI / app server | Streamlit |
| Data processing | Python, Pandas |
| Visualisation | Plotly |
| HTML export | Jinja2 |
| CVE data | NVD API v2 (NIST) |
| Exploit prediction | EPSS API (FIRST.org) |
| Cache | Streamlit `st.session_state` |
| Testing | pytest |
| Deployment | Streamlit Community Cloud |

## Testing

Unit tests cover the scoring engine, priority band assignment, age decay normalization, and `NVDClient` with mocked HTTP responses. Tests are structured to run without live API access.

```bash
pytest tests/ -v
```

## Open Source

Public repository — built as a portfolio project demonstrating applied vulnerability management domain knowledge, multi-source API integration, and a composable Python architecture with a Streamlit frontend.
