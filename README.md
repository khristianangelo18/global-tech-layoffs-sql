# 📉 Global Tech Layoffs Analysis (2020–2023)

![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-Data_Cleaning_%26_EDA-E38C00?style=for-the-badge&logo=databricks&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

An end-to-end SQL project demonstrating real-world **Data Cleaning** and **Exploratory Data Analysis (EDA)** on over 2,300 tech layoff records across the globe from March 2020 to March 2023.

---

## 📖 Table of Contents
- [Project Overview](#-project-overview)
- [Dataset Summary](#-dataset-summary)
- [Repository Structure](#-repository-structure)
- [Phase 1: Data Cleaning Workflow](#-phase-1-data-cleaning-workflow)
- [Phase 2: Exploratory Data Analysis (EDA)](#-phase-2-exploratory-data-analysis-eda)
- [Key Business Insights](#-key-business-insights)
- [How to Reproduce](#-how-to-reproduce)
- [Author](#-author)

---

## 📌 Project Overview
Between 2020 and 2023, the global tech ecosystem underwent dramatic fluctuations—from rapid pandemic-era hiring to widespread workforce downsizings driven by macroeconomic pressures and rising interest rates.

This project uses **MySQL** to transform raw, messy layoff records into an analysis-ready database and extract actionable business intelligence. The workflow is divided into two phases:

1. **Data Cleaning & Standardization ([`01_data_cleaning.sql`](sql/01_data_cleaning.sql))**: Raw tabular data is staged, deduplicated using window functions, standardized (text trimming, date formatting, categorical normalization), and cleansed of irrecoverable nulls.
2. **Exploratory Data Analysis ([`02_exploratory_data_analysis.sql`](sql/02_exploratory_data_analysis.sql))**: Deep-dive querying utilizing Common Table Expressions (CTEs), window functions (`DENSE_RANK()`, `SUM() OVER()`), and multi-level aggregations to analyze layoffs across companies, industries, countries, funding stages, and timelines.

---

## 📊 Dataset Summary

| Metric | Details |
| :--- | :--- |
| **Total Records** | 2,361 rows |
| **Time Period** | March 11, 2020 – March 6, 2023 |
| **Total Layoffs Tracked** | 386,000+ employees worldwide |
| **Key Attributes** | `company`, `location`, `industry`, `total_laid_off`, `percentage_laid_off`, `date`, `stage`, `country`, `funds_raised_millions` |
| **Primary Source** | Tech layoff tracker dataset ([`data/layoffs.csv`](data/layoffs.csv)) |

---

## 📁 Repository Structure

```text
global-tech-layoffs-sql/
├── data/
│   └── layoffs.csv                       # Raw source dataset (2,361 records)
├── sql/
│   ├── 01_data_cleaning.sql              # Staging, deduplication, schema standardization & null handling
│   └── 02_exploratory_data_analysis.sql  # Time-series trends, rolling totals, rankings, and aggregations
├── .gitignore                            # Standard Git ignore rules
└── README.md                             # Comprehensive project documentation
```

---

## 🧹 Phase 1: Data Cleaning Workflow

All data cleaning operations are executed in [`sql/01_data_cleaning.sql`](sql/01_data_cleaning.sql).

### 1. Staging Architecture
To safeguard raw data integrity, modifications were never performed directly on the source table:
- Created a replica staging environment `layoffs_staging` replicating the raw schema.
- Populated it with raw data to perform non-destructive transformations.

### 2. Duplicate Detection & Removal
Real-world datasets often feature duplicate entries across records:
- Implemented `ROW_NUMBER()` partitioned across all unique business dimensions to detect duplicate entries:
  ```sql
  WITH duplicate_cte AS (
      SELECT *,
          ROW_NUMBER() OVER(
              PARTITION BY company, location, industry, total_laid_off,
                           percentage_laid_off, `date`, stage, country, funds_raised_millions
          ) AS row_num
      FROM layoffs_staging
  )
  SELECT *
  FROM duplicate_cte
  WHERE row_num > 1;
  ```
- Because MySQL does not support direct `DELETE` operations on Common Table Expressions (CTEs), a second staging table (`layoffs_staging2`) was created with an explicit `row_num` column, enabling safe deletion:
  ```sql
  DELETE 
  FROM layoffs_staging2
  WHERE row_num > 1;
  ```

### 3. Standardization & Schema Formatting
- **Text Trimming**: Stripped leading and trailing whitespace across company names using `TRIM()`.
- **Category Normalization**: Consolidated fragmented industry labels (e.g., standardizing `CryptoCurrency` and `Crypto Currency` to `Crypto`).
- **Country Cleanup**: Removed trailing syntax anomalies (such as `'United States.'` to `'United States'`) using:
  ```sql
  UPDATE layoffs_staging2
  SET country = TRIM(TRAILING '.' FROM country)
  WHERE country LIKE 'United States%';
  ```
- **Temporal Conversion**: Converted raw text date values (`%m/%d/%Y`) to native MySQL `DATE` datatypes via `STR_TO_DATE()` followed by schema alteration (`ALTER TABLE layoffs_staging2 MODIFY COLUMN date DATE;`), enabling time-series indexing and aggregations.

### 4. Null & Blank Handling
- Converted empty string values (`''`) to native SQL `NULL` across categorical columns.
- **Self-Join Imputation**: Where a company had missing `industry` data in one record but valid industry data in another (e.g., Airbnb, Bally's Interactive), self-joins were used to backfill missing classifications:
  ```sql
  UPDATE layoffs_staging2 t1
  JOIN layoffs_staging2 t2
      ON t1.company = t2.company
  SET t1.industry = t2.industry
  WHERE t1.industry IS NULL
    AND t2.industry IS NOT NULL;
  ```
- **Pruning Unusable Records**: Filtered out unrecoverable records where both `total_laid_off` and `percentage_laid_off` were `NULL`, followed by dropping the temporary `row_num` helper column.

---

## 📈 Phase 2: Exploratory Data Analysis (EDA)

All exploratory queries and aggregations are documented in [`sql/02_exploratory_data_analysis.sql`](sql/02_exploratory_data_analysis.sql).

### Highlight 1: Rolling Layoff Totals Over Time
Aggregated monthly layoffs and calculated a progressive cumulative total using window functions:

```sql
WITH Rolling_Total AS (
    SELECT 
        SUBSTRING(`date`, 1, 7) AS `MONTH`, 
        SUM(total_laid_off) AS total_off
    FROM layoffs_staging2
    WHERE SUBSTRING(`date`, 1, 7) IS NOT NULL
    GROUP BY `MONTH`
    ORDER BY 1 ASC
)
SELECT 
    `MONTH`, 
    total_off, 
    SUM(total_off) OVER(ORDER BY `MONTH`) AS rolling_total
FROM Rolling_Total;
```

### Highlight 2: Top 5 Companies with Highest Layoffs Per Year
Ranked enterprise impact by year using Common Table Expressions and partitioned dense ranking:

```sql
WITH Company_Year (company, years, total_laid_off) AS (
    SELECT 
        company, 
        YEAR(`date`), 
        SUM(total_laid_off)
    FROM layoffs_staging2
    GROUP BY company, YEAR(`date`)
), 
Company_Year_Rank AS (
    SELECT *, 
           DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS Ranking
    FROM Company_Year
    WHERE years IS NOT NULL
)
SELECT *
FROM Company_Year_Rank
WHERE Ranking <= 5;
```

---

## 💡 Key Business Insights

* 🇺🇸 **Geographic Concentration**: Over **66%** of tracked layoffs originated in the **United States** (~256K roles), followed by **India** (~36K roles) and the **Netherlands** (~17K roles).
* 🛒 **Most Impacted Sectors**: **Consumer** (~46.6K) and **Retail** (~43.6K) faced the highest workforce reductions, outpacing pure-play enterprise software and finance sectors.
* 🏢 **Enterprise Scaling & Impact**: Mega-cap tech enterprises account for the largest individual layoff events recorded, led by **Amazon** (18,150+), **Google** (12,000), **Meta** (11,000), and **Salesforce** (10,090).
* 📉 **Temporal Volatility**: Layoff volumes accelerated sharply throughout late 2022 and peaked in early 2023, reflecting macroeconomic interest rate adjustments and post-pandemic tech workforce realignments.
* 💥 **Complete Liquidations**: Multiple high-profile startups with significant funding (over $100M+ raised) experienced 100% workforce layoffs (`percentage_laid_off = 1`), highlighting vulnerability even among well-capitalized firms.

---

## 🚀 How to Reproduce

### Prerequisites
- MySQL Server (v8.0+ recommended)
- MySQL Workbench, DBeaver, or command-line client
- Git

### Step-by-Step Instructions

1. **Clone the repository:**
   ```bash
   git clone https://github.com/khristianangelo18/global-tech-layoffs-sql.git
   cd global-tech-layoffs-sql
   ```

2. **Create Database & Import Dataset:**
   - In MySQL Workbench or CLI, create a new database schema:
     ```sql
     CREATE DATABASE world_layoffs;
     USE world_layoffs;
     ```
   - Import [`data/layoffs.csv`](data/layoffs.csv) into table `layoffs` using the MySQL Table Data Import Wizard.

3. **Run Data Cleaning:**
   - Execute [`sql/01_data_cleaning.sql`](sql/01_data_cleaning.sql) to generate and populate `layoffs_staging2`.

4. **Run Exploratory Data Analysis:**
   - Execute [`sql/02_exploratory_data_analysis.sql`](sql/02_exploratory_data_analysis.sql) to run analytical and ranking queries.

---

## 👤 Author
**Khristian Angelo TIu**
- GitHub: [@khristianangelo18](https://github.com/khristianangelo18)
