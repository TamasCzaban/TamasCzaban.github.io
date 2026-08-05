---
title: "From Two Minute First Paint to Under Two Seconds in Streamlit"
summary: "A deployed internal dashboard was fast locally but took one to two minutes to render its heaviest page in OpenShift. Instrumentation exposed a cache invalidation path that walked an entire shared folder. Selected date checks, background reconciliation, and precomputed common states brought common first paint below two seconds."
date: "Aug 05 2026"
tags: ["Python", "Streamlit", "Performance", "Caching", "OpenShift"]
draft: false
---

The heaviest page in an internal Streamlit dashboard took more than a minute to show its first usable screen after deployment. On my machine, it felt fine. In OpenShift, some sessions waited close to two minutes.

I had already started planning a DuckDB and React rewrite. Streamlit has limits, and the dashboard had grown beyond the shape of a simple app. Before committing the team to a migration, I wanted to know where the deployed version spent its time.

The answer was a cache miss, not the charts.

This post covers the mechanisms and decisions from an internal Citi tool. It does not include source code, infrastructure details, data, or internal paths.

## Start with a measurement boundary

The dashboard rendered several independent visuals. I first looked at parallelism. Python's GIL limits parallel execution of Python bytecode, and Streamlit's cache makes concurrent work fragile. I introduced concurrency where the work allowed it, then measured the change. It helped, but only slightly.

That result stopped me from optimizing the wrong part of the app.

I then timed the important function calls from the first app function through the final UI render call. The deployed environment made that harder than expected. Console output did not survive in the place I expected it to, while stdout reached the remote terminal. I added focused application logging that wrote the timings where I could inspect them after a request.

The measurements were not a formal latency SLO or percentile study. They were elapsed timings inside the application, corroborated by what users saw in the browser. They were enough to isolate the slow path.

## A one minute TTL that triggered a full folder walk

The dashboard supports historical date lookback. New Parquet files can arrive after deployment, so I had replaced an old static file list with dynamic discovery and a one minute TTL. The original static list froze the available dates at deployment time. If data landed the next day, the app could show no data for today.

The new implementation fixed freshness and introduced a worse problem. Whenever the date selector missed its cache, the app walked the entire shared folder to rediscover every available date. That network operation took roughly a minute and a half. The user waited for it before the page could render.

The data lifecycle gave me a narrower option. Delivered files are immutable. We only generate data for the current day, so a historical date that was absent will not appear later.

I changed the request path to check the selected date directly and return as soon as the file existed. A separate background process reconciles the rest of the dates since the previous polling event. Someone returning after a week can get today's availability without waiting for the whole historical scan, while the background work restores the wider date list.

That change brought first paint from roughly one to two minutes to under ten seconds.

## Precompute the states people actually open

The next bottleneck appeared after the folder walk disappeared. Complex pages use five multiselect filters. Their combinations exceed 16,000 states, so precomputing every view would waste storage and build time.

I precomputed the default state and the common filtered states instead. For each of those states, the data pipeline now produces the layout inputs and the filter aware week over week deltas ahead of time. Previously, the app had to reaggregate seven daily datasets on demand to calculate those deltas.

The app still computes uncommon filter combinations on demand. The common path gets a prepared answer.

Common first paint now lands below two seconds. Less common filter combinations have a different profile and were not part of the measured common state result.

## What changed in my approach

I first assumed the architecture was the problem. A rewrite may still make sense, and the DuckDB and React migration remains under way. It was not the fastest way to fix a deployed performance incident.

Instrumentation made the next steps specific:

- remove the full folder walk from the request path
- separate freshness for the requested date from reconciliation of historical dates
- prepare the few filter states users open most often

I now add observability before assuming a framework is the bottleneck. Local performance tells you whether the app works. Deployed measurements tell you which dependency users are waiting on.

For the broader system design, see the [Firewall and Load Balancer Vulnerability Dashboard](/projects/citi-firewall-vulnerability-dashboard/) and the earlier [13 stage Tekton deployment story](/blog/10-openshift-tekton-streamlit/).
