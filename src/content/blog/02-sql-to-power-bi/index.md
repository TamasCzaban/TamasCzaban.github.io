---
title: "The Right Way to Connect SQL to Power BI"
summary: "Pulling data straight from a database into Power BI works — until it doesn't. Here's the pattern I use to keep reports fast, clean, and maintainable."
date: "Oct 15 2024"
draft: false
tags:
- SQL
- Power BI
- Data Modelling
---

The first Power BI report I built connected directly to a raw database table with a "Get Data" click. It worked. The report loaded, the numbers looked right, and I shipped it.

Six months later it was a mess. Calculated columns everywhere, relationships between tables that shouldn't have relationships, a query that took 40 seconds to refresh. The lesson: the quality of a Power BI report is mostly determined before you open Power BI.

The pattern I use now — after building dashboards at Ericsson and on personal projects — separates the work cleanly into SQL and Power BI, each doing what it's good at.

## Do the transformation in SQL

Power BI has Power Query, and Power Query can do almost everything SQL can. But SQL is faster, more readable, version-controllable, and shareable across tools. Any transformation you can express in SQL should be expressed in SQL.

What this means in practice:

**Filter early.** If your report only needs data from 2019 onwards, put `WHERE year >= 2019` in the query. Don't pull five years of records into Power BI and filter them there.

**Join in SQL, not Power BI.** Multi-table joins in Power Query are slow and hard to debug. Write the join in SQL, import the result as a single clean table.

**Name columns like a human.** `CustomerFullName` not `CUST_NM_FULL_01`. You'll type these names in DAX constantly.

**Handle nulls before import.** `ISNULL(column, 'Unknown')` in SQL is one line. Dealing with nulls after the fact in Power BI is a cascade of conditional columns.

For my Sales Management Dashboard project, each dimension table got its own SQL view before it touched Power BI:

```sql
SELECT
    c.CustomerKey,
    c.FirstName + ' ' + c.LastName AS CustomerFullName,
    c.EmailAddress,
    g.City,
    g.EnglishCountryRegionName AS Country
FROM DimCustomer c
LEFT JOIN DimGeography g ON c.GeographyKey = g.GeographyKey
ORDER BY CustomerKey ASC
```

Clean, readable, no surprises in Power BI.

## Build a real data model

Once the data is in Power BI, build a proper star schema before writing a single measure. This means:

- **Fact tables** contain numeric measures and foreign keys (sales amounts, quantities, dates as keys)
- **Dimension tables** contain descriptive attributes (customer names, product categories, calendar fields)
- **One-to-many relationships** flowing from dimensions to facts

The temptation is to import one big flat table and get straight to the visuals. This works for simple reports but collapses as soon as you need to filter by multiple dimensions simultaneously or compare against a budget.

A proper model also makes DAX dramatically simpler. Most of your measures end up being `CALCULATE(SUM(...), filter)` rather than deeply nested conditional expressions.

## DAX belongs in measures, not calculated columns

New Power BI users reach for calculated columns first — they're familiar, they show up in the table view, they feel like spreadsheet formulas.

Calculated columns are evaluated row by row and stored in memory. For large tables this is expensive. Measures are evaluated at query time against the current filter context, which is almost always what you actually want.

My rule: if the calculation depends on the visual's filter context (which most do), it's a measure. If it's a fixed property of a row that doesn't change based on filters — like categorising ages into buckets — it can be a calculated column or, better, a SQL column.

## The payoff

Reports built this way refresh faster, are easier to update when requirements change, and are readable by anyone who picks them up later. The SQL files live in version control. The Power BI file contains a clean model and clear measures, not a tangle of transformations.

It's more upfront work. It's always worth it.
