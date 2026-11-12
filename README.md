# Rossmann Store Sales: Promotion Days and Sales Forecasting

How much higher are daily sales on promotion days in Rossmann drugstores, and how well can
daily sales be forecast six weeks ahead?

This project analyses daily sales data from 1,115 Rossmann stores in Germany
(January 2013 – July 2015). It combines exploratory analysis, a panel regression for the
sales difference on promotion days, and a comparison of forecasting models on rolling
test windows.

![Sales difference on promotion days by weekday](figures/promo_effect_by_weekday.png)

## Key results

- **Promotion days:** After controlling for day of week, season, day of month, holidays and
  store differences, daily sales are about **37% higher on promotion days** than on
  non-promotion days (95% CI: 35% to 40%, standard errors clustered by store and week).
  This is an observational comparison, not a causal effect (see [Limitations](#limitations)).
- **The difference depends on the weekday:** about **+58% on Mondays** and **+22% on Fridays**
  (Wald test Monday vs. Friday: χ² = 147.3, p < 0.001).
- **Forecasting:** Across four 6-week test windows, LightGBM has an average RMSPE of 14.4%,
  compared with 15.2% for the best simple baseline. LightGBM is ahead in all four windows,
  but the gain is small (0.4 to 1.4 points), and the average error (MAE) improves by about 5%.
- **Pitfalls found in the data:** Several patterns that look like effects turned out to be
  caused by selection bias or confounding (see [Exploratory analysis](#exploratory-analysis)).

## Data

The data comes from the Kaggle competition
[Rossmann Store Sales](https://www.kaggle.com/competitions/rossmann-store-sales/data).
Two files are used:

- `train.csv`: daily sales, customers, promotions and holidays per store (1,017,209 rows)
- `store.csv`: store attributes such as store type, assortment and competition (1,115 stores)

The data is **not included** in this repository. To run the notebooks, download it from
Kaggle (you need to accept the competition rules first) and place both files in `data/`.

## Project structure

```
rossmann-sales-analysis/
├── data/                              # not included, see "Data"
├── figures/                           # plots used in this README
├── notebooks/
│   ├── 01_data_preparation.ipynb      # data quality checks, cleaning, merge
│   ├── 02_eda.ipynb                   # exploratory data analysis
│   ├── 03_promo_effect.ipynb          # panel regression for promotion days
│   └── 04_forecasting.ipynb           # baselines vs. LightGBM on rolling windows
├── requirements.txt
└── README.md
```

## Methods

### 1. Data preparation
- Integrity checks: no duplicate store-date rows, `DayOfWeek` matches `Date`, one row per
  store in `store.csv`.
- Removed closed days (`Open = 0`, 172,817 rows) and 54 open days with zero sales
  (52 of them also have zero customers, so these stores were effectively closed).
- 180 stores have no data for 184 days (1 July to 31 December 2014). For time trends,
  average sales per store are used instead of total sales.
- Missing `Promo2` dates are structural: they belong only to stores without Promo2.
- Saved the cleaned data (844,338 rows) as Parquet to keep data types.

### 2. Exploratory analysis
See [findings](#exploratory-analysis) below.

### 3. Sales on promotion days (panel regression)
- Outcome: `log(Sales)`, so coefficients can be read as percentage differences.
- Models with increasing controls: (1) promotion only, (2) + day of week, month, year and
  holidays, (3) + store fixed effects, (4) + day of month. Model 5 estimates a separate
  difference for each weekday.
- Promotion status is the same for all stores on a given date and constant within each
  week (135 weeks, 72 of them promotion weeks). Standard errors are therefore clustered
  by store and week.
- Sundays are excluded (only 33 special stores are open, and promotions never run on Sundays).
- Library: `linearmodels` (`PanelOLS`).

### 4. Forecasting
- Four rolling test windows of 6 weeks each (February to July 2015). For every window,
  the models only see data from before the window.
- Only features that are known in advance are used. `Customers` is excluded to avoid
  data leakage. Promo2 features respect the week in which each store joined Promo2.
- Three simple baselines, all based on the average per store, weekday and promotion status:
  (A) arithmetic mean over the full history, (B) geometric mean over the last 52 weeks,
  (C) geometric mean over the last 12 weeks.
- LightGBM trained on `log(1 + Sales)`. The number of trees is chosen by early stopping on
  the 6 weeks before each test window. The model is then refitted with three seeds.
- Metrics: RMSPE (the official Kaggle metric) and MAE.

## Results

### Sales on promotion days

| Model | Controls | Difference | 95% CI |
|---|---|---|---|
| 1 | none | +41.5% | +38.1% to +45.0% |
| 2 | day of week, month, year, holidays | +39.3% | +36.6% to +42.2% |
| 3 | + store fixed effects | +39.3% | +36.5% to +42.2% |
| 4 | + day of month | **+37.3%** | +34.6% to +40.0% |

- Time controls reduce the estimate, because promotions never run on Saturdays, which have
  slightly lower sales.
- Store fixed effects do not change it, because promotions are scheduled chain-wide.
- Day of month matters: 77% of weekdays on days 1–5 of a month are promotion days, but only
  33% on days 11–15. Without this control, start-of-month effects are partly attributed
  to promotions.

| Day | Difference | 95% CI |
|---|---|---|
| Mon | +57.7% | +52.1% to +63.4% |
| Tue | +43.8% | +38.9% to +49.0% |
| Wed | +35.3% | +31.1% to +39.7% |
| Thu | +30.0% | +26.6% to +33.4% |
| Fri | +22.2% | +19.3% to +25.2% |

### Forecasting

RMSPE (%) by test window. LightGBM values are the mean of three seeds.

| Test window | Baseline A | Baseline B | Baseline C | LightGBM |
|---|---|---|---|---|
| 14 Feb – 27 Mar 2015 | 19.4 | 17.7 | 22.9 | 16.8 |
| 28 Mar – 8 May 2015 | 17.1 | 15.8 | 17.1 | 15.4 |
| 9 May – 19 Jun 2015 | 15.3 | 14.0 | 14.4 | 12.6 |
| 20 Jun – 31 Jul 2015 | 14.7 | 13.5 | 15.1 | 12.9 |
| **Mean** | 16.6 | 15.2 | 17.4 | **14.4** |
| Mean MAE | 780 | 709 | 822 | 677 |

- Baseline B (last 52 weeks, geometric mean) is the best simple baseline. Using only the
  last 12 weeks (C) is worse, because each average is based on few observations.
- LightGBM beats Baseline B in all four windows, by 0.4 to 1.4 RMSPE points. Over all windows
  and seeds, its RMSPE ranges from 12.4% to 17.0%. The advantage is consistent but small.
- Forecast accuracy differs much more between test windows than between models.
- In the most recent window, R² is 0.888 for Baseline B and 0.914 for LightGBM. LightGBM is
  better in 744 of 1,115 stores.

**A few stores dominate the error.** In the most recent window, the worst 1% of rows produce
24% of the squared percentage error. Store 292 has an RMSPE above 100%. Without the five
worst stores, the RMSPE of LightGBM drops from 12.6% to 12.0% (seed 0).

**Which features matter?** Retraining LightGBM on the most recent window without groups
of features:

| Features | RMSPE | MAE |
|---|---|---|
| All features | 12.6% | 602 |
| Without store ID | 21.7% | 994 |
| Without store attributes (type, assortment, competition, Promo2) | 13.4% | 620 |

The store ID carries most of the information: the store attributes cannot replace it.
The attributes add a small improvement on top of the store ID.

![Forecast examples](figures/forecast_examples.png)

## Exploratory analysis

The most important lesson from the exploratory analysis: several simple comparisons are
misleading.

- **Selection bias on Sundays and holidays:** Average sales look high on Sundays and public
  holidays. But only a few special stores are open on those days. Within the 33 Sunday-open
  stores, Sunday sales are on average lower than on other days (this holds for 20 of the
  33 stores).
- **Confounding by promotions:** Saturdays seem to be weak sales days. But promotions never
  run on Saturdays. Without promotions, Saturday sales are close to the weekday level.
- **Location as a confounder:** Stores with a competitor nearby have higher sales. This is
  likely because competitors cluster in busy urban areas, not because competition increases
  sales.
- **Seasonality:** Strong peak in December, drop in January, slight upward trend over time.
- **Store structure:** Store types reach similar sales in different ways (more customers vs.
  larger baskets). All 17 type-b stores are open on Sundays, which suggests special
  high-traffic locations.

## Limitations

- **The promotion result is not a causal effect.** Promotions were not randomly assigned.
  Anything else happening chain-wide in promotion weeks (e.g. advertising) is part of
  the estimate.
- **Pull-forward cannot be ruled out.** Promotion and non-promotion weeks alternate: 60 of
  the 63 non-promotion weeks directly follow a promotion week. If customers move purchases
  into promotion weeks, sales in the comparison weeks are lower than normal, and the
  difference overstates the additional sales created by promotions.
- The forecasting models use no information about recent sales levels (lag or rolling
  features), and stores with unusual periods (e.g. closures) are forecast poorly.
- LightGBM hyperparameters were not tuned systematically. The number of trees chosen by
  early stopping varies strongly between windows (229 to 859).
- The error analysis by store and the feature ablation use only the most recent test window
  and one seed.

## How to run

**Locally**

1. Clone the repository and install the packages: `pip install -r requirements.txt`
2. Download `train.csv` and `store.csv` from Kaggle and place them in `data/`.
3. Run the notebooks in `notebooks/` in order (01 → 04). Notebook 01 creates
   `data/train_clean.parquet`, which the other notebooks use. Notebook 04 trains about
   20 models and takes a while.

**Google Colab**

Create the folder `rossmann-sales-analysis` in your Google Drive with the subfolders
`data/`, `figures/` and `notebooks/`, upload the data and the notebooks, and run them in
order. The notebooks detect Colab and mount Google Drive automatically.

Tested with Python 3.13.

## Tools

Python · pandas · NumPy · Matplotlib · linearmodels · LightGBM · Google Colab

## Author

Mina Motaghinasab, M.Sc. Data Science student at TU Dortmund
[LinkedIn](https://www.linkedin.com/in/m-motaghinasab) ·
[GitHub](https://github.com/mina-motaghinasab)
