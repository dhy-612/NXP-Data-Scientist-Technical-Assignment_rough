# NXP Data Scientist Assignment

## Overview

This repository contains my solution for the NXP Data Scientist assignment using World Bank data on life expectancy, mortality, and fertility.

The analysis covers:

- Statistical analysis across 1960–2023
- Three required visualizations
- An interactive Streamlit dashboard
- Reusable Python modules under `src/`

Unless gender is explicitly specified, **total life expectancy at birth** is used as the primary measure.

## Assignment Dataset

The analysis uses the following five World Bank indicators:

| Indicator | Code | Dataset Column |
|---|---|---|
| Life expectancy at birth, total | `SP.DYN.LE00.IN` | `LifeExp_Total` |
| Life expectancy at birth, male | `SP.DYN.LE00.MA.IN` | `LifeExp_Male` |
| Life expectancy at birth, female | `SP.DYN.LE00.FE.IN` | `LifeExp_Female` |
| Death rate, crude | `SP.DYN.CDRT.IN` | `Death_Rate` |
| Fertility rate, total | `SP.DYN.TFRT.IN` | `Fertility_Rate` |

The analysis uses annual observations from **1960 through 2023**, giving 64 years of data.

## Project Structure

```text
NXP_Data_Scientist_Assignment/
│
├── README.md
├── requirements.txt
├── .gitignore
├── app.py
│
├── data/
│   ├── raw/
│   │   ├── API_SP.DYN.CDRT.IN_DS2_en_csv_v2_33278/
│   │   ├── API_SP.DYN.LE00.FE.IN_DS2_en_csv_v2_34008/
│   │   ├── API_SP.DYN.LE00.IN_DS2_en_csv_v2_408/
│   │   ├── API_SP.DYN.LE00.MA.IN_DS2_en_csv_v2_189/
│   │   └── API_SP.DYN.TFRT.IN_DS2_EN_csv_v2_33381/
│   │
│   └── processed/
│       ├── master_dataset.csv
│       └── classified_dataset.csv
│
├── notebooks/
│   └── analysis.ipynb
│
├── src/
│   ├── __init__.py
│   ├── data_preprocessing.py
│   ├── statistical_analysis.py
│   └── visualizations.py
│
└── visualizations/
    ├── 01_life_expectancy_by_income_group.html
    ├── 02_life_expectancy_world_map_2023.html
    └── 03_life_expectancy_sankey.html
```

# 1. Data Preparation

## Raw Data

The World Bank CSV files are provided in wide format, with country/entity information in the first columns and yearly values from 1960 onward.

The preprocessing pipeline:

1. Reads the five indicator datasets.
2. Skips the World Bank metadata/header rows.
3. Keeps the years **1960–2023**.
4. Converts the datasets from wide format to long format.
5. Converts numeric observations to numeric values, coercing unavailable values to `NaN`.
6. Adds `Region` and `IncomeGroup` using country metadata.
7. Merges all five indicators using `Country Code` and `Year`.
8. Creates `Gender_Gap = LifeExp_Female - LifeExp_Male`.
9. Validates that there is only one observation per `Country Code + Year`.

The merge uses `Country Code` rather than country name because country names can differ between indicator files.

## Master Dataset

`master_dataset.csv` is the complete cleaned dataset.

It contains:

- **265 source entities**
- **64 years**
- **16,960 rows** (`265 × 64`)
- Classified countries/entities plus aggregate or unclassified entities

The master dataset is the main cleaned source of truth produced by preprocessing.

## Classified Dataset

`classified_dataset.csv` is created from the master dataset by keeping only observations where `IncomeGroup` is available:

```python
classified_df = master_df[master_df["IncomeGroup"].notna()].copy()
```

It contains:

- **217 classified entities**
- **64 years**
- **13,888 rows** (`217 × 64`)

Income-group distribution:

| Income Group | Entities |
|---|---:|
| High income | 86 |
| Upper middle income | 59 |
| Lower middle income | 47 |
| Low income | 25 |

The **classified dataset** is used for income-group analysis, country-level correlation analysis, and the country-based visualizations because aggregate/unclassified entities do not have an income-group classification.

# 2. Statistical Analysis

## Q1. Which income group has the largest change in the average male-female life expectancy difference?

The analysis uses all years from 1960–2023.

For each year and income group:

```text
Gender Gap = Average Female Life Expectancy
             - Average Male Life Expectancy
```

The annual change is then calculated as:

```text
Yearly Change = Current Year Gap - Previous Year Gap
```

To capture movement throughout the complete period rather than only comparing 1960 and 2023, the ranking metric is:

```text
Total Absolute Gap Change
= Σ |Yearly Change|
```

This is the cumulative year-to-year movement of the gender-gap series.

### Result

| Rank | Income Group | Total Absolute Gap Change |
|---:|---|---:|
| 1 | Low income | **8.797** |
| 2 | Lower middle income | 7.539 |
| 3 | Upper middle income | 5.214 |
| 4 | High income | 4.240 |

**Answer: Low income** has the largest cumulative movement in the average male-female life expectancy gap.

For the low-income group:

- Gap in 1960: **2.840 years**
- Gap in 2023: **4.399 years**
- Endpoint change: **+1.559 years**
- Cumulative absolute year-to-year movement: **8.797 years**

> Note: 8.797 is cumulative movement across yearly changes. It does **not** mean that the gender gap increased by 8.797 years.

## Q2. Which income group has the greatest variability in life expectancy at birth?

For every year, the average total life expectancy is calculated for each income group.

Variability is measured across the **64 annual income-group averages** using variance:

```text
Variance of annual average life expectancy, 1960–2023
```

### Result

| Rank | Income Group | Variance | Standard Deviation |
|---:|---|---:|---:|
| 1 | Low income | **51.517** | 7.178 |
| 2 | Lower middle income | 42.370 | 6.509 |
| 3 | Upper middle income | 29.605 | 5.441 |
| 4 | High income | 19.060 | 4.366 |

**Answer: Low income** has the greatest variability in life expectancy over the full 1960–2023 period.

For the low-income group:

- Mean annual average life expectancy: **50.425 years**
- Minimum annual average: **39.761 years**
- Maximum annual average: **64.202 years**
- Range: **24.441 years**

> Note: The assignment does not prescribe an exact mathematical definition for “variability”, so this analysis defines it as the variance of the income group's annual average life expectancy over 1960–2023.

## Q3. Which countries have the highest and lowest correlation between fertility rate and life expectancy?

For every classified entity, Pearson correlation is calculated between:

- `LifeExp_Total`
- `Fertility_Rate`

All available paired observations from 1960–2023 are used for each entity.

### Results

**Highest positive correlation**

- **Zimbabwe:** `+0.345`

**Most negative correlation**

- **India:** `-0.997`

**Strongest absolute correlation**

- **India:** `-0.997`

Correlation measures association, not causation.

# 3. Visualizations

## Visualization 1 — Life Expectancy by Income Group

A time-series line chart showing the average total life expectancy for each income group from 1960 to 2023.

Output:

```text
visualizations/01_life_expectancy_by_income_group.html
```

## Visualization 2 — World Map of Life Expectancy in 2023

A Plotly choropleth map showing country-level total life expectancy for 2023. The map uses the classified dataset so aggregate/unclassified entities are excluded.

Output:

```text
visualizations/02_life_expectancy_world_map_2023.html
```

## Visualization 3 — Sankey Diagram of Rank-Category Transitions

Countries are ranked separately in 1960 and 2023 according to total life expectancy and divided into five roughly equal buckets:

1. Very low
2. Low
3. Medium
4. High
5. Very high

The Sankey diagram shows how countries moved between these categories from 1960 to 2023.

There are **217 classified entities**.

1960 bucket sizes:

```text
Very low   = 43
Low        = 43
Medium     = 43
High       = 43
Very high  = 43
```

2023 bucket sizes:

```text
Very low   = 43
Low        = 43
Medium     = 44
High       = 43
Very high  = 44
```

Output:

```text
visualizations/03_life_expectancy_sankey.html
```

# 4. Interactive Dashboard

The project includes a Streamlit dashboard for monitoring life expectancy, mortality, and fertility indicators.

The dashboard provides:

- Total life expectancy
- Male life expectancy
- Female life expectancy
- Male-female life expectancy difference
- Death rate
- Fertility rate
- Country income group
- Region
- Historical trends

### Filters

Users can filter the dashboard by:

- Country
- Income group
- Region
- Year range

### Additional comparisons

The dashboard also supports comparison views by:

- Income group
- Region

Run the dashboard with:

```bash
python -m streamlit run app.py
```

# 5. Reusable Python Modules

The `src/` directory separates the project into reusable components.

## `src/data_preprocessing.py`

Handles:

- Raw World Bank CSV loading
- Wide-to-long transformation
- Country metadata merge
- Five-indicator merge
- Gender-gap creation
- Master dataset creation
- Classified dataset creation

Run directly with:

```bash
python src/data_preprocessing.py
```

## `src/statistical_analysis.py`

Contains reusable functions for:

- Q1 gender-gap movement
- Q2 life-expectancy variability
- Q3 fertility/life-expectancy correlation

Run directly with:

```bash
python src/statistical_analysis.py
```

## `src/visualizations.py`

Contains reusable functions for:

- Income-group life expectancy trend
- 2023 world map
- 1960 → 2023 Sankey diagram

It can also save all three visualizations as standalone HTML files.

Run directly with:

```bash
python src/visualizations.py
```

# 6. Notebook

`notebooks/analysis.ipynb` contains the complete analysis workflow in a readable, report-style format.

It includes:

- Dataset inspection
- Data preprocessing
- Data validation
- Master vs classified dataset explanation
- Q1 analysis
- Q2 analysis
- Q3 analysis
- Required visualizations
- Final findings

The notebook uses project-relative paths so it can be run from the repository rather than relying on machine-specific file paths.

# 7. Installation

It is recommended to use a project-local virtual environment.

### Create virtual environment

Windows:

```bash
py -3.9 -m venv .venv
```

Activate it:

```bash
.\.venv\Scripts\Activate.ps1
```

### Install dependencies

```bash
python -m pip install -r requirements.txt
```

# 8. Running the Project

From the project root:

### Rebuild processed datasets

```bash
python src/data_preprocessing.py
```

### Run statistical analysis

```bash
python src/statistical_analysis.py
```

### Generate visualizations

```bash
python src/visualizations.py
```

### Launch dashboard

```bash
python -m streamlit run app.py
```

### Run notebook

Open:

```text
notebooks/analysis.ipynb
```

and execute the cells sequentially.

# 9. Reproducibility

The project is designed so that the workflow can be reproduced from the raw World Bank datasets.

The code uses:

- Relative project paths
- Explicit year range: 1960–2023
- Explicit indicator-to-folder mapping
- Merge validation for expected relationships
- Consistent variable names
- Reusable modules in `src/`

The `.venv` environment itself should not be committed to GitHub.

# 10. Technology Stack

- Python
- Pandas
- NumPy
- Plotly
- Streamlit
- Jupyter Notebook

# 11. Data Source

**World Bank — World Development Indicators**

The five indicators listed above are sourced from the World Bank data collection used in the assignment.

# 12. Key Findings

### Statistical Analysis

- **Low-income** entities show the largest cumulative movement in the male-female life expectancy gap.
- **Low-income** entities also show the greatest variability in annual average life expectancy across 1960–2023.
- **Zimbabwe** has the highest positive fertility/life-expectancy correlation among classified entities.
- **India** has the most negative and strongest absolute correlation.

### Overall Observation

The analysis shows substantial long-term differences in life expectancy across income groups, with lower-income groups experiencing greater movement and variability over the historical period.
