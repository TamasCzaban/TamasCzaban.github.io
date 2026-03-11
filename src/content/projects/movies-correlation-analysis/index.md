---
title: "Movies Correlation Analysis"
summary: "Python-based statistical analysis of what drives box office revenue, using Pandas, Seaborn, and Matplotlib on a Kaggle movies dataset."
date: "Mar 01 2022"
draft: false
tags:
- Python
- Pandas
- Seaborn
- Matplotlib
- Data Analysis
repoUrl: https://github.com/TamasCzaban
---

A data analysis project investigating which factors most strongly predict a movie's gross box office earnings, using Python and the scientific stack in a Jupyter Notebook.

## Key Finding

**Budget has the strongest positive correlation with gross revenue.** Vote count came in as the second-highest predictor — initially surprising, but logical: the more people who see a film, the more votes it accumulates on review platforms.

## What it does

- Cleans and prepares a real-world movie dataset from Kaggle
- Computes a full correlation matrix across all numeric features
- Visualises relationships between variables with scatter plots and heatmaps
- Applies regression analysis to quantify budget-to-revenue relationships

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python |
| Environment | Jupyter Notebook |
| Data manipulation | Pandas, NumPy |
| Visualisation | Seaborn, Matplotlib |
| Data source | Kaggle — danielgrijalvas/movies |

## Methodology

1. **Import & inspect** — loaded CSV, checked shape, dtypes, and null counts
2. **Clean** — converted float columns to integers where appropriate, handled missing values
3. **Explore** — sorted by gross revenue to surface top performers
4. **Correlate** — generated numeric correlation matrix, then encoded categorical variables for inclusion
5. **Visualise** — scatter plots with trend lines, regression plots, correlation heatmap

## Visualisations

- Scatter plot: budget vs gross earnings with regression line
- Heatmap: full correlation matrix across all features
- Trend analysis incorporating categorised non-numeric variables (genre, rating, company)

## Key Skills Demonstrated

- End-to-end exploratory data analysis workflow in Python
- Statistical correlation analysis with proper handling of categorical variables
- Data cleaning and type normalisation with Pandas
- Clear visual communication of statistical findings with Seaborn/Matplotlib
