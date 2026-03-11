---
title: "BEMER CRM"
summary: "A full-stack CRM and e-commerce platform for BEMER distributors, built with Streamlit, Firebase, and Stripe."
date: "Jan 01 2024"
draft: false
tags:
- Python
- Streamlit
- Firebase
- Stripe
- CRM
repoUrl: https://github.com/TamasCzaban
---

A production CRM system built for BEMER product distributors to manage customer relationships, track sales pipelines, and process payments — all from a single web interface.

## What it does

- **Customer management**: Full CRUD interface for managing distributor contacts and leads
- **Sales pipeline**: Visual kanban-style pipeline to track deals through stages
- **Payment processing**: Stripe integration for subscription billing and one-off purchases
- **Real-time database**: Firebase Firestore for live data sync across users
- **Auth**: Firebase Authentication with role-based access (admin vs distributor)

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit |
| Backend | Python |
| Database | Firebase Firestore |
| Auth | Firebase Authentication |
| Payments | Stripe API |
| Hosting | Google Cloud / Firebase |

## Key Decisions

Chose Streamlit over a traditional frontend framework to maximize development speed while still shipping a polished, interactive UI. Firebase was selected for its real-time sync capabilities and free tier for small teams. Stripe handles PCI compliance so the app never touches raw card data.

## Status

Production — actively used by BEMER distributors.
