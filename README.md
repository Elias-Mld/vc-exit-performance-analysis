# VC analysis: investor pedigree and exit outcomes (CrunchBase 2003–2013)

## Introduction

In venture capital, the core question is often the **exit**: turning years of risk into liquidity for founders, and a portfolio into distributions for investors. A widespread belief is that **VC pedigree** is associated with better exit outcomes in terms of **probability**, **time to exit**, and **price**.

This study **does not claim strict causality**. It compares, on **CrunchBase** from **2003 to 2013**, startups funded by the **ten most active investors** in the dataset (*Top 10 VCs*, a proxy for **deal volume**, not financial performance) with startups funded by the **rest of the market**. The window spans a **full macroeconomic cycle** (expansion, 2008 crisis, recovery) and focuses on **documented, realized exits**, which limits the bias of still-**paper** valuations common in more recent vintages.

Detailed results, working hypotheses, charts, bias discussion, and the methodological appendix are in the **PDF report** below.

## CSV tables used (CrunchBase dump)

The extractions described in the **SQL appendix** rely in particular on **three files** from the dump:

| File | Role |
|------|------|
| `acquisitions.csv` | Acquisitions (exit dates, amounts when reported). |
| `objects.csv` | Entities (companies, funds) and identifiers to name VCs and link records. |
| `funding_rounds.csv` | Funding rounds and dates, including the **first round** used to compute **time to exit**. |

The **`investments.csv`** file is also used in the queries to **link investors to funded startups** and build the *Top VC* / *market* groups (see the SQL appendix).

**Public dataset (Kaggle):** [Startup Investments — CrunchBase](https://www.kaggle.com/datasets/justinas/startup-investments/data)

## Resources for the full analysis

To understand the **entire** analysis as delivered, these **three resources** are sufficient. Everything else in the repository is local production material and **not required** for understanding the methodology or the results.

| Resource | Contents |
|----------|----------|
|[Full PDF report](Analyse_de_donnee_pedigree.pdf) | Executive summary, hypotheses, six illustrated analyses, conclusion, biases, and integrated appendix. |
| **[Full SQL appendix](./appendix_sql_full.md)** | Extraction and data-structuring queries. |
| **[Full Python (Pandas) appendix](./appendix_pandas_full.md)** | Post-SQL processing: aggregations, quantiles, deduplication, metrics aligned with the report. |

## Author

**Elias Milandou** — [GitHub](https://github.com/Elias-Mld)
