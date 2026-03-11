---
title: "Citi GEM Dashboard"
summary: "Internal vulnerability tracking dashboard built with Python, Streamlit, and Pandas for Citi's global security team."
date: "Apr 01 2025"
draft: false
tags:
- Python
- Streamlit
- Pandas
- Security
- Data Engineering
---

An internal analytics dashboard developed at Citi to track global vulnerability exposure across enterprise assets and surface remediation insights for security stakeholders.

> **Note**: This is an internal tool — no public demo or source code is available due to confidentiality.

## What it does

- Ingests raw vulnerability scan data from multiple security tools
- Normalizes and aggregates data using Pandas for consistent reporting
- Provides filterable, drill-down views by asset, severity, and business unit
- Tracks remediation SLA compliance over time with trend charts
- Exports executive summary reports for leadership review

## Tech Stack

| Layer | Technology |
|---|---|
| Data processing | Python, Pandas |
| UI / Dashboard | Streamlit |
| Data sources | Internal security scan APIs |
| Reporting | Excel exports, PDF summaries |

## Impact

Replaced a manual, error-prone Excel-based process — reducing weekly reporting effort significantly and enabling the team to identify high-risk assets faster.

## Status

Internal / confidential — deployed for use by Citi's Vulnerability Threat Management team.
