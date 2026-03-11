---
title: "From Data Analyst to Security Analyst: What Transfers"
summary: "Moving from BI and data analysis into vulnerability management wasn't a pivot — it was the same skills in a different domain. Here's what I learned."
date: "Mar 01 2025"
draft: false
tags:
- Career
- Security
- Data Analysis
---

When I moved from a Data Analyst role at Ericsson into Vulnerability Threat Management at Citi, a few people asked if it was a big shift. I'd spent two years building Power BI dashboards and writing SQL queries. Now I was working in cybersecurity. It sounded like a change.

It wasn't, really.

## The job is still data

Vulnerability management is fundamentally a data problem. You have a large, messy dataset — thousands of vulnerabilities across hundreds of systems, sourced from multiple scanning tools in inconsistent formats. You need to make sense of it, track it over time, and communicate what matters to people who need to make decisions.

That's exactly what a data analyst does.

The domain knowledge is different. Learning what CVSS scores mean, understanding the difference between a vulnerability and an exposure, knowing which CVEs are actually being exploited in the wild — that took time. But the underlying skillset transferred directly:

- **SQL** for querying vulnerability databases and asset inventories
- **Python + Pandas** for normalising data across tools and automating reporting
- **Dashboarding** for giving stakeholders visibility without burying them in raw data
- **Knowing what question is actually being asked** — often the most valuable skill of all

## The BI skills I use every week

At Ericsson I spent a lot of time thinking about data models. What's a fact, what's a dimension, how do you structure tables so that filtering and aggregation are fast and reliable. That mental model applies directly to vulnerability data.

A vulnerability record is a fact. The asset it lives on is a dimension. The business unit that owns the asset is another dimension. The remediation deadline is a calculated field derived from severity and discovery date. Once you think of it that way, the reporting structure becomes obvious.

The dashboards I build at Citi are more complex than any Power BI report I built at Ericsson — more data sources, more stakeholders, higher stakes. But they're the same kind of thinking.

## What I had to learn

The domain itself. Security has its own vocabulary, its own mental models, its own threat landscape. I read a lot. I asked a lot of questions. I made a point of understanding *why* certain vulnerabilities get prioritised over others — not just reporting the numbers, but understanding what they mean.

I also had to get comfortable with ambiguity. Security data is messier than most. Scanners disagree with each other. Asset inventories are out of date. Systems get decommissioned and nobody updates the records. Part of the job is deciding what to trust and when to flag data quality issues rather than silently propagate them.

## The thing nobody tells you

The most valuable skill in both data analysis and security isn't technical. It's the ability to understand what decision your audience needs to make and give them exactly the information that serves it — no more, no less.

A vulnerability report that lists 10,000 open CVEs is useless. A report that says "these 12 Critical vulnerabilities are past SLA and owned by these three teams" is actionable. The difference isn't the data. It's understanding what the data is for.

That's what data analysis teaches you. Turns out it's also what security analysis needs.

If you're a data analyst wondering whether your skills would transfer to adjacent technical domains — security, engineering, product — the answer is usually yes. The domain knowledge is acquirable. The ability to work with data, ask the right questions, and communicate findings clearly is the harder thing to learn, and you probably already have it.
