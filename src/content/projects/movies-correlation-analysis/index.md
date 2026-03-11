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

---

## Dataset

![Initial Dataset](/images/movies/image.png)

![Data Types Check](/images/movies/dataset-dtypes.png)

![Cleaned Dataset](/images/movies/dataset-cleaned.png)

## Sorted by Gross Revenue

![Sorted by Gross](/images/movies/image-1.png)

## Correlation Analysis

![Budget vs Gross Scatter](/images/movies/image-2.png)

![Budget vs Gross with Trend Line](/images/movies/image-3.png)

![Score vs Gross Correlation](/images/movies/image-4.png)

## Correlation Matrix

![Correlation Matrix Table](/images/movies/image-5.png)

![Correlation Heatmap](/images/movies/image-6.png)

![Numerized Correlation Heatmap](/images/movies/image-7.png)

---

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

## Key Skills Demonstrated

- End-to-end exploratory data analysis workflow in Python
- Statistical correlation analysis with proper handling of categorical variables
- Data cleaning and type normalisation with Pandas
- Clear visual communication of statistical findings with Seaborn/Matplotlib
