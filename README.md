# Predicting Greenhouse Gas Emissions from Energy Consumption Data

A regression analysis project on global energy and emissions data (1900–2021),
built to demonstrate a complete, honest data science workflow including a
real methodological pitfall (data leakage) that was found, diagnosed, and
fixed rather than hidden.

## Table of Contents
- [Project Overview](#project-overview)
- [Dataset](#dataset)
- [Methodology](#methodology)
- [Key Finding: Catching Data Leakage](#key-finding-catching-data-leakage)
- [Results](#results)
- [Visualizations](#visualizations)
- [Limitations](#limitations)
- [How to Run](#how-to-run)
- [Future Work](#future-work)

## Project Overview

This project asks a simple question: **can national greenhouse gas (GHG)
emissions be predicted from energy-consumption and economic indicators?**

The answer turned out to be more interesting than expected not because of
a high accuracy score, but because of *why* the first model's accuracy score
was misleadingly high, and what a corrected, honest model reveals instead.

## Dataset

- **File:** `merged_energy_data.csv(1)` (included in this repository)
- **Raw size:** 17,239 rows × 129 columns (country–year panel data, 1900–2021)
- **After cleaning:** 1,785 rows × 21 columns
- Columns cover fuel-specific consumption (coal, gas, oil, nuclear,
  renewables), aggregate energy metrics, population, GDP, and GHG emissions.
- Originally sourced from a public energy dataset compiled by country and
  year; included here directly so the notebook is fully reproducible without
  external downloads.
  
## Methodology

1. **Data cleaning** — dropped rows missing the target variable, dropped rows
   with less than 80% complete data, median-imputed remaining gaps.
2. **Feature engineering** — derived `fossil_dependency_ratio`,
   `renewable_ratio`, and `emissions_per_capita` to capture structural energy
   mix rather than raw volume.
3. **Exploratory analysis** — distribution checks, a full correlation matrix,
   and time-series trends across decades.
4. **Modeling** — linear regression, evaluated with R², RMSE, and MAE.
5. **Validation** — critically, two different modeling approaches are
   compared side by side (see below), rather than reporting a single number.

## Key Finding: Catching Data Leakage

The first model built (**Model A**) used fuel-specific consumption features
(coal, gas, oil, renewables, nuclear) as predictors and achieved:

> **Test R² = 0.9956, RMSE = 90.5**

At first glance this looks like an excellent model. On inspection, it isn't
one — it's a **tautology**. `fossil_fuel_consumption` and related features
are near-identities of `greenhouse_gas_emissions`, because GHG emissions
*are* mechanically derived from fossil fuel combustion volumes. Feeding a
model these features is close to predicting a quantity from a rescaled
version of itself a classic case of **data leakage**.

A second issue compounded this: the data is **country × year panel data**,
but the original train/test split was a random row split. That lets the same
country appear in both the training and test sets, artificially inflating
the apparent generalization performance.

**Model B** was built to correct both issues:
- Removed all directly fuel-derived features, keeping only genuinely
  independent drivers: `population`, `gdp`, `energy_per_capita`,
  `fossil_dependency_ratio`, `renewable_ratio`.
- Used `GroupShuffleSplit` keyed on `country`, so entire countries are held
  out for testing a fair test of generalization to *unseen* countries.

> **Test R² = 0.2558, RMSE = 328.2**

The honest score is much lower and that is the correct, defensible result.
It shows that once the accounting shortcut is removed, cross-country
economic and demographic variables alone only modestly explain absolute
national emissions.

## Results

| Model | Features | Split | Test R² | Test RMSE |
|---|---|---|---|---|
| A (baseline) | Fuel-specific consumption + population/GDP | Random 80/20 | 0.9956 | 90.5 |
| B (honest) | Population, GDP, energy/capita, fuel/renewable ratios | Country-held-out | 0.2558 | 328.2 |

Top drivers in Model B (standardized coefficients): `energy_per_capita` and
`population` dominate; the fossil/renewable ratio features and GDP have much
smaller standalone effect once collinear fuel features are removed.

## Visualizations

The notebook produces the following comparison figures (all in `/images`):

- `02_time_series.png` — global emissions, fossil consumption, renewables,
  and population trends over time
- `03_target_distribution.png` — raw vs. log-transformed emissions
- `04_correlation_heatmap.png` — full correlation matrix
- `05_correlation_ranked_bar.png` — ranked correlation with the target
- `06_actual_vs_predicted.png` — Model A vs Model B, side by side
- `07_residuals.png` — residual plots for both models
- `08_residual_distribution.png` — overlaid residual density comparison
- `09_metric_comparison_bars.png` — R² / RMSE bar chart, train vs test, both models
- `10_feature_coefficients.png` — standardized coefficients, Model B

## Limitations

- **Data reduction:** cleaning dropped the dataset by 89.6% (17,239 → 1,785
  rows); results skew toward larger, better-reported economies and recent
  years.
- **Linear model only:** no comparison against non-linear baselines (e.g.
  gradient boosting) is included yet.
- **No formal inference:** coefficients are reported as point estimates,
  without confidence intervals or significance tests.
- **Model B's lower R² is expected, not a flaw** — it reflects the true,
  weaker signal once the accounting identity is removed.

## How to Run

```bash
git clone https://github.com/ZahidulAlam277/energy-emissions-prediction.git
cd energy-emissions-prediction
pip install -r requirements.txt
jupyter notebook Energy_Emissions_Analysis.ipynb

