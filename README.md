# Batting Average Ability (BAA)

Batting average mixes two things that matter to a front office: how well a
hitter actually hits the ball, and how many of those balls happen to find
grass. This project separates the two. It builds a metric called Batting
Average Ability (BAA) that estimates the skill portion of a hitter's batting
average from their plate discipline and quality of contact, and reports the
leftover as a luck component (actual batting average minus predicted).

Everything here models a player-season's own-season batting average and BABIP
from that same season's peripheral stats. It is a skill-isolation project, not a
forecasting one. There is no next-season target and no projection claim
anywhere in the repo. If forecasting is the goal, the natural next step is a
year-over-year BAA stability test, which is not yet built.

## What we did

We pulled ten seasons of FanGraphs batting stats (2015 to 2024, minimum 100
plate appearances), cleaned them into 4,374 player-seasons across 1,173
players, and split them 80/20 into 3,499 training and 875 test rows.

We then built models in two passes. The first pass predicts BABIP (batting
average on balls in play) from batted-ball composition and contact quality. The
second pass predicts batting average using the predicted BABIP from the first
pass plus plate-discipline stats. Both passes use mixed-effects models with a
random intercept per player, which lets the model separate stable player-level
skill from season-to-season noise. Finally, we combined the fitted coefficients
into the BAA index and validated it against actual batting average.

## What we found

About a quarter of batting-average variation is stable, player-level skill. The
player random intercept accounts for 24.7% of AVG variation and 18.4% of BABIP
variation (the intraclass correlation, or ICC). The batting-average model
explains 45.7% of test-set variance from peripherals alone, with a test mean
absolute error of 0.0215 and essentially no overfitting (train-test R-squared
gap of 0.0015).

The strongest single driver of BABIP after batted-ball mix is sprint speed. On
standardized coefficients it ranks second (+0.0087, p < 0.001), behind only the
line-drive-versus-flyball share. This matches baseball intuition: faster
hitters beat out grounders and leg out infield hits.

The BAA index passes a face-validity check. Indexed so that league average is
100 and one standard deviation is 10 points within each season, the top score
is Luis Arraez in 2023 at 133.2 (.354 batting average) and the bottom is Joey
Gallo in 2023 at 61.2 (.177).

| Model | Target | Test R² | Test MAE | Notes |
|-------|--------|---------|----------|-------|
| Linear regression | AVG | 0.983 | 0.0035 | Uses same-season BABIP, so it is close to an accounting identity. Not a skill result. |
| Linear regression | BABIP | 0.402 | 0.0253 | Baseline. |
| Random forest | AVG | n/a (RMSE 0.0312) | | BABIP excluded as a feature. |
| Mixed effects (CLR features) | BABIP | 0.347 to 0.372 | 0.026 | ICC 0.184. Range explained in Caveats. |
| Mixed effects (predicted BABIP) | AVG | 0.457 | 0.0215 | ICC 0.247. This is the model behind BAA. |

We tried more complex models for BABIP and rejected them on evidence. A random
forest scored a test R-squared of 0.293 against 0.372 for the linear baseline, a
21% drop, with an overfit gap of 0.166 versus 0.027. Interactions and an
ensemble were also worse. That is what justifies keeping the interpretable
mixed-effects model.

Validating BAA within each season by fitting actual batting average against
BAA, the R-squared ranges from 0.409 in 2023 to 0.527 in 2021, and is 0.425 in
2024 (MAE 0.0179).

## Repository layout

```
Batting-Average-Ability/
├── README.md
├── requirements.txt
├── notebooks/
│   ├── batting_average_model.ipynb   Main pipeline: clean, split, model, index, validate
│   └── BA_dict.ipynb                 Warm-up that builds a {player name: batting average} dictionary
├── data/
│   ├── raw/
│   │   └── 2023-2025_hitting_stats.csv     Baseball Savant, 214 players, used only by BA_dict.ipynb
│   ├── processed/
│   │   ├── cleaned_batting_data.csv         Master dataset, FanGraphs 2015 to 2024, 4,374 rows
│   │   ├── X_train.csv, X_test.csv          80/20 split, random_state 42
│   │   ├── Y_train.csv, Y_test.csv          Batting-average targets
│   │   ├── train_avg_features.csv, test_avg_features.csv
│   │   └── train_babip_clr_features.csv, test_babip_clr_features.csv
│   └── predictions/
│       ├── pred_avg_train.csv, pred_avg_test.csv       Actual vs predicted AVG and residuals
│       ├── pred_babip_train.csv, pred_babip_test.csv   Actual vs predicted BABIP
│       ├── baa_coefficients_output_no_predbabip.csv    Full dataset with the 100-indexed BAA and player names
│       ├── baa_season_table.csv     Per player-season skill estimate and luck component
│       └── baa_player_table.csv     Career-level raw batting average vs shrunk BAA
└── models/
    ├── linear_regression_avg.pkl, linear_regression_babip.pkl, rf_avg_model.pkl
    ├── linear_regression_coefficients.csv, linear_regression_babip_coefficients.csv
    ├── babip_final_model_info.txt          Final BABIP model spec and metrics
    └── babip_model_comparison.csv          Linear vs interactions vs random forest vs ensemble
```

The two forms of the metric live in different files. The 100-indexed BAA, with
player names and the plate and contact component scores, is in
`baa_coefficients_output_no_predbabip.csv`. The batting-average-scale skill
estimate and its per-season luck component are in `baa_season_table.csv`, and
career-level shrinkage is in `baa_player_table.csv`.

## Setup and running

```bash
pip install -r requirements.txt
jupyter notebook notebooks/batting_average_model.ipynb
```

File paths in the notebooks are relative to the `notebooks/` directory (for
example `../data/processed/X_train.csv`), so run them from that folder. Jupyter
does this automatically when you open a notebook there.

The committed CSV and model files let you inspect every result without running
anything. To regenerate from scratch, run `batting_average_model.ipynb` top to
bottom. Its first cell re-downloads FanGraphs data through `pybaseball`, so an
internet connection is required and the download takes a few minutes.

## Method in order

1. Clean. Pull FanGraphs 2015 to 2024 batting stats, drop columns that are more
   than half missing, keep one record per player-season (the one with the most
   plate appearances), one-hot encode teams, remove outliers with a 3x IQR
   rule, and median-fill the rest. Output: `cleaned_batting_data.csv`.
2. Split. A random 80/20 split with a fixed seed. Seasons are mixed together.
3. Baselines. Ordinary least squares for AVG and BABIP, plus a random forest
   for AVG. The random forest is rejected on the evidence in
   `babip_model_comparison.csv`.
4. BABIP model. A mixed-effects model with a random intercept per player. The
   batted-ball shares (line drive, ground ball, fly ball) are centered log-ratio
   transformed because they sum to one and cannot be treated as independent.
   Infield-fly rate is re-expressed on the balls-in-play scale, and rows are
   weighted by plate appearances through replication so full seasons count more.
5. AVG model. A second mixed-effects model that uses the predicted BABIP from
   step 4 plus plate-discipline stats such as strikeout rate, walk rate, chase
   and zone swing and contact rates, and swinging-strike rate. This model
   produces the 0.457 test R-squared and is the basis for BAA.
6. BAA index. Standardized coefficients are combined into a plate score and a
   contact score, summed, and then indexed to mean 100 and standard deviation 10
   within each season so that eras are comparable. Seasons below 200 plate
   appearances are excluded from the index.
7. Validation. Fit actual batting average against BAA within each season from
   2019 to 2024, and plot BAA against actual batting average colored by season.

`BA_dict.ipynb` is a small standalone warm-up that builds a lookup from player
name to batting average. It is not part of the modeling pipeline.

## Caveats

The models describe the same season, they do not forecast the next one.
Predicting a season's batting average from its own peripherals decomposes skill
and luck, it does not project future performance.

The 80/20 split mixes seasons and there is no cross-validation, so the reported
R-squared is not an out-of-time estimate. This is acceptable for a descriptive
metric but worth knowing.

The linear AVG R-squared of 0.983 is not a skill result. It includes same-season
BABIP as a feature, and batting average is largely an accounting identity of
BABIP, strikeout rate, and home-run-per-flyball rate. The honest headline number
is the mixed model's 0.457.

The BABIP model's test R-squared is 0.347 in the current notebook run but 0.372
in the saved `babip_final_model_info.txt` and `babip_model_comparison.csv`. The
two runs were fit slightly differently and the gap has not been reconciled. Do
not quote one figure without knowing which run produced it.

The BABIP prediction intervals are poorly calibrated. Coverage is 54.5% against
a nominal target, so the intervals are too narrow.

## Data source

Season-level batting statistics come from FanGraphs through the
[`pybaseball`](https://github.com/jldbc/pybaseball) library, calling
`batting_stats(year, qual=100)` for each year from 2015 to 2024. The Baseball
Savant export in `data/raw/` is used only by the warm-up notebook.
