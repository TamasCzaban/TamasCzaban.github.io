---
title: "Why I Build Internal Tools with Streamlit"
summary: "Streamlit lets you ship a real, interactive web app in a fraction of the time it takes with a traditional stack. Here's why I keep reaching for it."
date: "Feb 10 2025"
draft: false
tags:
- Python
- Streamlit
- Internal Tools
---

There's a pattern I keep seeing in corporate environments: a team has data, they need to make decisions from it, and the current solution is a shared Excel file that someone emails around on Fridays.

When I joined Citi's Vulnerability Threat Management team, that was roughly the situation. Thousands of vulnerability records across global assets, tracked in spreadsheets, summarised manually, sent up the chain. The data was there. The insight wasn't.

My answer was Streamlit.

## What Streamlit actually is

Streamlit is a Python library that turns a script into an interactive web app. You write Python — the same Pandas, the same logic you'd use in a notebook — and Streamlit handles the UI. Dropdowns, sliders, tables, charts: all declared in a few lines.

```python
import streamlit as st
import pandas as pd

df = pd.read_csv("vulnerabilities.csv")

severity = st.selectbox("Filter by severity", ["All", "Critical", "High", "Medium"])
if severity != "All":
    df = df[df["severity"] == severity]

st.dataframe(df)
st.bar_chart(df.groupby("business_unit")["vuln_count"].sum())
```

That's a working filter and chart. In a traditional stack that's a backend route, a frontend component, a state management layer, and a deployment pipeline.

## Why it works for internal tools

Internal tools have a different set of constraints to customer-facing products. The users are colleagues — they're forgiving of rough edges, they understand the domain, and they just need the information. What they can't tolerate is waiting six weeks for the IT backlog to clear so someone can run a report for them.

Streamlit hits the sweet spot:

- **Fast to build** — a useful prototype in an afternoon, a polished tool in a week
- **Python all the way down** — no context switch to JavaScript, no API layer to maintain
- **Runs where Python runs** — local, cloud, internal server
- **Easy to update** — when the data changes or the team wants a new filter, I change one file

## The BEMER CRM

The most complex Streamlit app I've built is a full CRM for BEMER product distributors. It manages customer contacts, tracks deals through a sales pipeline, processes payments via Stripe, and syncs everything in real time through Firebase Firestore.

This pushed Streamlit into territory it's not always associated with — persistent state, multi-user access, role-based auth, live data. It works because Streamlit's component model is flexible enough, and because Firebase handles the heavy lifting of real-time sync and authentication.

That said: Streamlit has limits. It's not the right tool for highly interactive UIs, complex client-side logic, or public-facing consumer products. For internal tooling though — dashboards, data apps, reporting interfaces — it's the fastest path from idea to working software I've found.

## The workflow I've settled on

1. Start with the data — load it, clean it, understand what questions people actually need answered
2. Build the minimal version — one filter, one chart, one table
3. Share it early — real users find the gaps faster than I do
4. Iterate — add filters, improve the layout, connect more data sources

The goal is never the tool. The goal is the decision the tool enables. Streamlit just gets out of the way fast enough that I can focus on that.
