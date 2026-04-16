---
title: "CZ Dev — Software Agency (Co-Founder)"
summary: "Co-founded with my brother Zsombor. Custom software for founders who've outgrown spreadsheets and no-code. Four shipped case studies across production CRM, security tooling, and vulnerability automation. Design-led frontend, Python backend, full-stack delivery."
date: "Mar 01 2026"
draft: false
tags:
- Agency
- Full-Stack
- React
- TypeScript
- Python
- Design-Led
demoUrl: "https://www.czaban.dev"
repoUrl: "https://github.com/TamasCzaban"
---

**CZ Dev** ([czaban.dev](https://www.czaban.dev)) is the software agency I co-founded with my brother Zsombor. The positioning: custom engineering for founders who've hit the ceiling of spreadsheets, Airtable, and no-code — and need something deliberate, typed, and maintained.

## Division of labour

| Area | Owner |
|------|------|
| Python backend, data layer, Firebase, security | Tamas (me) |
| React frontend, UX, TypeScript, visual design | Zsombor |
| Architecture, content, case studies | Both |

Zsombor leads the design language — minimalist, architectural, craft-oriented. I own the Python side: data modelling, business rules, backend workflows, automation, security, testing. On projects where I also contributed frontend (BEMER v2, KEV Explorer, Advisory Composer), Zsombor set the design system and component vocabulary; I built components inside that grammar.

## Public case studies

Listed at [czaban.dev/portfolio](https://www.czaban.dev/portfolio), with longer writeups on this portfolio site:

| Project | What | Stack | Role |
|---|---|---|---|
| [MedKölcsön v1](/projects/bemer-crm/) | Production rental/sales CRM for a medical device business | Python, Streamlit, Firebase | Auth, navigation, 391-test suite |
| [MedKölcsön v2](/projects/bemer-crm-v2/) | React rewrite of v1, live at vital-registry.com | React, TypeScript, Firebase | Data-layer port, multi-env deployment |
| [KEV Explorer](/projects/kev-explorer/) | Interactive dashboard over CISA's 1,500+ Known Exploited Vulnerabilities | React, TypeScript, Recharts, GitHub Actions | Security data enrichment, EPSS scoring |
| [Advisory Composer](/projects/advisory-composer/) | Browser tool converting lockfiles into multi-channel security advisories | React, TypeScript, OSV.dev, EPSS | OSV query logic, formatter, Slack Block Kit |

## Why "design-led"

The category descriptor on czaban.dev is deliberate. Most small custom-software shops ship functional but visually unremarkable work — the pattern is to treat UI as an afterthought to the backend. CZ Dev takes the opposite approach: Zsombor's design work comes first, and the backend is built to honour it. Every case study ships with considered typography, whitespace, and interaction design, not just a working feature set.

For me personally, working inside that design grammar was where I first learned how much a principled frontend system changes the engineering experience. On v1 of BEMER the UX was what Streamlit allowed. On v2 it's what Zsombor's design system dictates — and the code is better for it.

## Tech choices across the portfolio

- **Frontend:** React + TypeScript on every recent project. Component libraries we trust rather than building from scratch.
- **Backend:** Firebase for realtime-state apps (BEMER). Static + serverless where possible (KEV Explorer, Advisory Composer).
- **Data sources:** OSV.dev, CISA KEV feed, EPSS, NVD API v2 — public security data surfaced in interfaces defenders will actually use.
- **Delivery:** Multi-environment pipelines (dev/UAT/prod) on the production apps. GitHub Actions for nightly data refreshes on the public tools.

## Status

Actively running. BEMER v2 is the first client in sustained production. KEV Explorer and Advisory Composer are public tools used as proof of what CZ Dev ships. I keep CZ Dev strictly out-of-hours alongside my full-time role.
