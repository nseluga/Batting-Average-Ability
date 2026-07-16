# Batting Average Ability (BAA)

**Separating a hitter's batting-average *skill* from *luck* using mixed-effects
modeling.**

Batting average (AVG) mixes two things: how well a hitter actually hits the ball
and how many of those balls happen to find grass. This project builds **BAA
(Batting Average Ability)** — a season-normalized index (league average = 100,
one standard deviation = 10 points, in the spirit of `wRC+` / `OPS+`) that
estimates the skill component of AVG from a player's peripheral stats (plate
discipline + quality of contact), and reports the leftover as a per-season
**luck component** (actual BA − predicted BA).

> **Scope — read this first.** This is a **skill-isolation / descriptive**
> project, *not* a forecasting one. Every model predicts a player-season's
> *same-season* AVG or BABIP from that same season's peripherals. There is no
> next-season target and no forecasting claim anywhere in the repo.

---

## Headline results

All numbers are from the committed notebook outputs and saved artifacts
(`data/predictions/`, `models/`). Corpus: **4,374 player-seasons, 1,173 unique
players, 2015–2024** (FanGraphs season-level stats, min 100 PA).

| Model | Target | Test R² | Test MAE | Notes |
|---|---|---|---|---|
| Mixed-effects (player random intercepts) | **AVG** | **0.457** | **0.0215** | uses predicted BABIP + plate discipline; train/test R² gap +0.0015 (no overfitting) |
| Mixed-effects + CLR + PA weighting | BABIP | 0.35–0.37¹ | ~0.026 | ICC 0.184 |
| OLS linear | BABIP | 0.402 | 0.0253 | baseline |
| Random forest (no BABIP) | AVG | — (RMSE 0.0312) | — | rejected: Test R² 0.293 vs linear 0.372 |
| OLS linear (BABIP as feature) | AVG | 0.983 | 0.0035 | **identity/leakage — not a skill result** (AVG is an accounting identity of BABIP, K%, HR/FB) |

**Skill share of variance (the headline finding):** the player random intercept
explains **ICC ≈ 24.7%** of AVG variance and **18.4%** of BABIP variance — i.e.
roughly a quarter of batting average is stable, player-level skill.

**Face validity** (top / bottom BAA seasons): Luis Arraez 2023 = **133.2**
(BA .354); Joey Gallo 2023 = **61.2** (BA .177).

**Concrete baseball insight:** *sprint speed* is the **#2 driver of BABIP**
(standardized coef +0.0087, p<0.001) — fast hitters beat out grounders and
infield hits.

¹ The BABIP mixed model's Test R² is recorded as **0.347** in the current
notebook run but **0.372** in the saved `models/babip_final_model_info.txt` /
`models/babip_model_comparison.csv`. This inconsistency is unresolved — see
[Caveats](#caveats--known-limitations).

---

## Repository layout

```
Batting-Average-Ability/
├── README.md
├── requirements.txt
├── notebooks/
│   ├── batting_average_model.ipynb   # main pipeline: clean → split → models → BAA → validation
│   └── BA_dict.ipynb                 # warm-up: builds a {player name -> BA} dictionary
├── data/
│   ├── raw/          # source data as downloaded
│   │   └── 2023-2025_hitting_stats.csv     # Baseball Savant, 214 players (BA_dict.ipynb only)
│   ├── processed/    # cleaned data + train/test splits + standardized feature matrices
│   │   ├── cleaned_batting_data.csv         # master dataset (FanGraphs 2015-2024, 4,374 rows)
│   │   ├── X_train.csv / X_test.csv         # 80/20 random split (random_state=42)
│   │   ├── Y_train.csv / Y_test.csv         # AVG targets
│   │   ├── train_avg_features.csv / test_avg_features.csv
│   │   └── train_babip_clr_features.csv / test_babip_clr_features.csv
│   └── predictions/  # model outputs and the BAA deliverables
│       ├── pred_avg_train.csv / pred_avg_test.csv       # actual vs predicted AVG + residuals
│       ├── pred_babip_train.csv / pred_babip_test.csv   # actual vs predicted BABIP
│       ├── baa_player_table.csv     # 1,173 career rows: RAWBA vs shrunk BAA, PA, seasons
│       ├── baa_season_table.csv     # per player-season: avg_actual, avg_pred, luck_component
│       └── baa_coefficients_output*.csv
└── models/           # trained model artifacts + coefficients + model specs
    ├── linear_regression_avg.pkl / linear_regression_babip.pkl
    ├── rf_avg_model.pkl
    ├── linear_regression_*_coefficients.csv
    ├── babip_final_model_info.txt          # final BABIP model spec + metrics
    └── babip_model_comparison.csv          # linear vs interactions vs RF vs ensemble
```

The two headline deliverables are **`data/predictions/baa_player_table.csv`**
(career-level BAA per hitter) and **`data/predictions/baa_season_table.csv`**
(per player-season BAA and its `luck_component`).

---

## Getting started

```bash
# 1. Install dependencies (Python 3.9+ recommended)
pip install -r requirements.txt

# 2. Launch Jupyter and open the main notebook
jupyter notebook notebooks/batting_average_model.ipynb
```

All file paths in the notebooks are **relative to the `notebooks/` directory**
(e.g. `../data/processed/X_train.csv`), so run the notebooks from that folder
(Jupyter does this automatically when you open them there).

The committed CSVs and `.pkl` files let you inspect every result **without
re-running anything**. To regenerate from scratch, run
`batting_average_model.ipynb` top to bottom — the first cell re-downloads
FanGraphs data via `pybaseball`, so an internet connection is required and the
download takes a few minutes.

---

## Method, in order

The pipeline in `batting_average_model.ipynb` runs in these stages:

1. **Clean** (`cleaned_batting_data.csv`) — pull FanGraphs 2015–2024 batting
   stats via `pybaseball` (min 100 PA), drop >50%-missing columns, de-duplicate
   player-seasons (keep the max-PA record), one-hot encode teams, remove 3×IQR
   outliers, median-fill the rest.
2. **Split** — plain random 80/20 `train_test_split(random_state=42)`
   (3,499 train / 875 test; seasons mixed).
3. **Baselines** — OLS linear regression for AVG and for BABIP, plus a random
   forest for AVG (BABIP excluded). RF is **rejected on evidence**
   (`babip_model_comparison.csv`: RF Test R² 0.293 vs linear 0.372, overfit gap
   0.166 vs 0.027), which is what justifies the interpretable linear/mixed
   models.
4. **BABIP mixed-effects model** — a `statsmodels` `MixedLM` with random
   intercepts by player. Batted-ball composition (LD/GB/FB) is
   **CLR-transformed** (centered log-ratio) because the shares sum to one and
   can't be treated as independent; infield-fly rate is re-expressed on the
   balls-in-play scale; rows are **PA-weighted via replication** so high-PA
   seasons carry more weight.
5. **AVG mixed-effects model** — a second `MixedLM` that uses the *predicted*
   BABIP from step 4 plus plate-discipline features (K%, BB%, chase/zone swing &
   contact, SwStr%). This is the model behind BAA (Test R² 0.457).
6. **BAA index** — standardized coefficients are combined into a `plate_score`
   and a `contact_score`, summed to `baa_raw`, then indexed to **mean 100 / SD
   10 within each season** (min 200 PA) so eras are comparable.
7. **Validation** — within-season fit of `BA ~ BAA` for 2019–2024 (R² ranges
   0.409 in 2023 to 0.527 in 2021; 2024 R² 0.425), plus an interactive Plotly
   scatter of BAA vs actual BA colored by season.

`BA_dict.ipynb` is a small standalone warm-up (builds a `{player -> BA}`
dictionary from `data/raw/2023-2025_hitting_stats.csv`); it is **not** part of
the modeling pipeline.

---

## Caveats & known limitations

Honest framing for anyone building on this:

- **Same-season, not predictive.** Nothing here forecasts future performance.
  Predicting a season's AVG from that season's own peripherals is a
  skill-*decomposition*, not a projection. A year-over-year BAA stability test
  would be the natural next step to make a forecasting claim.
- **Random split, not temporal.** The 80/20 split mixes seasons, and there is no
  cross-validation. For a descriptive metric this is defensible, but it means
  the reported R² is not an out-of-time estimate.
- **The linear AVG R² of 0.983 is not a skill result.** It includes same-season
  BABIP as a feature, and AVG is largely an accounting identity of BABIP, K%,
  and HR/FB. Treat that row as a sanity check, not a finding. The honest AVG
  number is the mixed model's **0.457**.
- **BABIP metric discrepancy.** The saved artifacts report Test R² 0.372 while
  the current notebook run reports 0.347. The source of the gap has not been
  reconciled; don't quote a single figure without checking which run produced
  it.
- **BABIP prediction intervals are poorly calibrated** — coverage is 54.5%
  (`babip_final_model_info.txt`), well below nominal.

---

## Data source

Season-level batting statistics are pulled from **FanGraphs** via the
[`pybaseball`](https://github.com/jldbc/pybaseball) library
(`batting_stats(year, qual=100)`) for 2015–2024. The separate Baseball Savant
export in `data/raw/` is used only by the warm-up notebook.
