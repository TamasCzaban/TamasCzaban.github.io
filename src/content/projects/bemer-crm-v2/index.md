---
title: "Vital Registry v2 — React Production"
summary: "Full React migration of Vital Registry, the production CRM built with my brother for our Mum's BEMER medical device rental business. Same Firebase + Stripe backbone, redesigned UI, dev/UAT/prod deployment pipeline, and live at vital-registry.com/contracts."
date: "Apr 01 2026"
draft: false
tags:
- React
- TypeScript
- Firebase
- Stripe
- CRM
- Multi-Environment
demoUrl: "https://vital-registry.com/contracts"
repoUrl: "https://github.com/TamasCzaban"
---

The React rewrite of Vital Registry. Same business logic as v1 ([Streamlit version](/projects/bemer-crm/)), redesigned from scratch in React + TypeScript, and running in production at [vital-registry.com/contracts](https://vital-registry.com/contracts).

> **Live production app** — non-technical user (our Mum) runs her BEMER rental business on it daily.

## Why a v2

v1 shipped fast on Streamlit because rendering a working UI in Python was the shortest path to a production app for a non-technical user. It worked — but Streamlit's page-rerun model and limited component ergonomics made mobile responsiveness and UX polish a constant fight. v2 is the same domain, same rules, done right in React.

## What changed

- **Frontend:** React + TypeScript replaces Streamlit's server-rendered UI. Component boundaries are explicit, state is local, and the mobile view is first-class.
- **Backend:** Firebase Firestore stays. The v1 data model, including the dual-inventory Possession Envelope Rule, ports directly — core business logic was already centralised in the DAO layer.
- **Auth:** Firebase Auth with Google/Facebook OAuth, migrated from v1.
- **Billing:** Stripe Checkout + Customer Portal, plan-based feature gating preserved.
- **Multi-environment pipeline:** dev / UAT / prod Firebase projects, deploy gates, separate Stripe keys per env. v1 ran on a single Streamlit Cloud instance; v2 is structured for real release management.

## Shared with v1

Every business rule that was explicitly covered by the 391-test suite in v1 maps to v2:

- **Dual inventory** — owned vs partner assets, with the Possession Envelope Rule enforced before form submission.
- **Multi-entity billing** — rental and sale transactions tied to specific legal entities, with validation against deleted entities.
- **Optimistic UI pattern** — ported from Streamlit's `st.session_state.task_overrides` idea to React state, so task completions feel instant without round-trip to Firestore.
- **Client-side PDF contracts** — same html2pdf approach, now using React's component tree to build the HTML before passing to the PDF renderer.

## Tech Stack

| Layer | v1 (Streamlit) | v2 (React) |
|---|---|---|
| UI | Streamlit | React + TypeScript |
| State | `@st.cache_data` + `st.session_state` | Local component state + React Query |
| Database | Firebase Firestore | Firebase Firestore (same schema) |
| Auth | `google-auth-oauthlib` | Firebase Auth SDK |
| Payments | Stripe API via Python | Stripe API via TypeScript SDK |
| PDF | html2pdf.js injected | html2pdf.js native |
| Deployment | Streamlit Cloud (single env) | Firebase Hosting (dev/UAT/prod) |

## Built With

Co-developed with my brother Zsombor, who set the visual system and component language. I did almost all of the frontend redesign on top of it, plus the data-layer port from v1, multi-environment deployment setup, and the test harness that carries over from v1.

Case study: [czaban.dev/portfolio](https://www.czaban.dev/portfolio) (MedKölcsön v2).
