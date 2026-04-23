---
title: "Vital Registry v1 — Streamlit Production"
summary: "The v1 of Vital Registry: a production full-stack CRM built with my brother for our Mum's BEMER medical device rental business. Streamlit frontend over Firebase Firestore, Stripe billing, Google/Facebook OAuth, client-side PDF contract generation, dual inventory (B2C + B2B cross-rentals), multi-entity billing, and 391 passing tests."
date: "Jan 01 2024"
draft: false
tags:
- Python
- Streamlit
- Firebase
- Stripe
- CRM
- Testing
repoUrl: https://github.com/TamasCzaban
---

A production CRM built with my brother for a real business: our Mum's BEMER medical device rental and sales operation. She is the sole non-technical user. Every architectural and UX decision was made against a single standard — **the Mum Test**: the UI must be forgiving, idiot-proof, strictly in Hungarian, and use exact local currency formatting everywhere.

> **Note**: Private production app — no public demo or source code available.

## The Problem

Managing a medical device rental business manually means tracking which devices are out on rental, which are coming back, which are borrowed from partner distributors for sub-rental, who owes what, and when contracts need generating — across spreadsheets and memory. The goal was to replace all of that with a single app that a non-technical user could operate confidently.

## Architecture

### Dual Inventory System

The app manages two distinct asset types with different rules:

- **Owned Assets** — devices purchased outright. Can be rented to clients, rented to other distributors (B2B), or sold. Full history tracked.
- **Partner Assets** — devices borrowed from partner distributors (B2B cross-rentals). Can only be sub-rented to clients within the active possession window. Sales are strictly blocked at the router level.

The **Possession Envelope Rule** governs Partner Assets: an outgoing rental to a client is only valid if its dates fall entirely within an active incoming B2B possession period. The availability checker enforces this before any form submission.

### Multi-Entity Billing & Contracts

Users can operate via multiple billing entities — registered companies or private persons — each with their own legal details. Every rental and sale transaction is tied to a specific entity, which drives contract generation. If the entity was deleted after a transaction was created, the contract generator throws a visible validation error rather than silently falling back to incorrect data.

### DAO Pattern with Enriched Memory Caching

All database access goes through a centralised `db_service.py` — no raw Firestore reads in UI views. Key patterns:

- **Enriched fetches**: `get_enriched_devices_df()` and similar functions compute statuses and merge histories entirely in RAM before the UI layer ever sees the data. Heavy payloads (HTML contract content) are excluded from analytical DataFrames to conserve memory.
- **Centralised mutations**: every write function in `db_service.py` clears the exact `@st.cache_data` blocks it affects, guaranteeing cache coherence after every change.
- **Optimistic UI overlay**: for high-frequency actions like checking off a calendar task, raw caches are not flushed. Instead, the write is committed and the change is saved to `st.session_state.task_overrides`. An uncached wrapper applies the overrides to the cached base data in milliseconds — avoiding a full cache flush for minor state updates.

### Dynamic Reminders (Calendar Feed)

Tasks are not stored as separate database entities. They are generated dynamically in RAM by scanning rental start and end dates. Task completion is tracked via sparse boolean flags on the rental record itself. Delays ("snooze") are tracked via sparse integer fields that shift the display date without mutating the contractual dates.

### Client-Side PDF Contract Generation

Contracts are generated client-side via an injected `html2pdf.js` component using Base64-encoded HTML payloads. This avoids server-side PDF libraries that block Streamlit threads and consume significant RAM — important for a shared-host deployment.

### Authentication

- Google and Facebook OAuth via `google-auth-oauthlib` with PKCE
- Manual email verification via Firebase REST API (`handle_code_in_app: True`) — the app intercepts the `?mode=verifyEmail&oobCode=...` redirect and calls `accounts:update` directly
- On first login, a default billing entity is automatically created from the user's registration details

### Stripe Integration

Plan-based feature gating (free vs pro) via Stripe Checkout and Customer Portal. All Stripe functions check `is_configured()` first and fail silently if not set up — allowing local development without a live Stripe connection. Subscription status is synced on session load.

## Testing

391 tests, 0 failures across 7 test files:

| Module | Tests |
|---|---|
| `utils.py` | 101 |
| `db_service.py` | 45 |
| `rental_form.py` | 38 |
| `partner_device_form.py` | 35 |
| `stripe_service.py` | 31 |
| `sale_form.py` | 21 |
| `email_service.py` | 15 |

Business logic — rental overlap checks, possession envelope validation, sale availability, atomic sale + buyer-status writes via Firestore batch — is fully covered. Test files for view-layer forms contain reference implementations of inlined logic that double as specifications for future extraction.

## Tech Stack

| Layer | Technology |
|---|---|
| UI / App | Streamlit |
| Database | Firebase Firestore (NoSQL) |
| Auth | Firebase Admin SDK, Google/Facebook OAuth |
| Payments | Stripe API (Checkout + Portal) |
| PDF generation | html2pdf.js (client-side, Base64 HTML) |
| Email | SMTP + Firebase REST API (verification) |
| Rich text | streamlit-jodit |
| Testing | pytest (391 tests) |

## Built With

Co-developed with my brother Zsombor, who led the project. I owned the navigation architecture and core data layer. Built for and actively used by our Mum to run her BEMER device rental and sales business.
