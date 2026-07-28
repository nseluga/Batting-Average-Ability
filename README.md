# Batting Average Ability (BAA)

I built BAA to answer one question: how much of batting average is skill, and how much is luck? It's a season-indexed metric from a player-random-intercept mixed-effects model that separates the two.

## Headline result

The same-season model explains **Test R² 0.457** (MAE 0.0215) of batting average from peripheral stats alone. The mixed-effects structure gives an ICC of **~24.7%**: about a quarter of the variance in batting average comes from persistent player skill. The rest is luck and context.

## Data & method

- FanGraphs season-level batting stats, 2015–2024, min 100 PA, pulled via [pybaseball](https://github.com/jldbc/pybaseball): 4,374 player-seasons, 1,173 players.
- I modeled BABIP with player random intercepts on a centered-log-ratio (CLR) transform of batted-ball composition (LD/GB/FB), weighted by plate appearances.
- AVG then comes from predicted BABIP plus plate-discipline features (K%, BB%, swing/contact rates).
- I tried a random forest too and rejected it on the evidence: Test R² 0.293 against 0.372 for the linear model, with a much larger overfit gap. The interpretable model won honestly, not by default.

## Concrete findings

- Sprint speed is the second-biggest BABIP driver after batted-ball trajectory: fast hitters beat out more ground balls.
- Face validity check: Luis Arraez 2023 scores BAA 133.2 (actual AVG .354); Joey Gallo 2023 scores 61.2 (actual AVG .177).
- BAA is indexed to mean 100 / SD 10 within each season, so scores compare directly across years.
- The BABIP model's test R² is 0.347 in the current notebook run but 0.372 in the saved `babip_final_model_info.txt`; the two runs were fit slightly differently and I haven't reconciled the gap. Don't quote one figure without knowing which run produced it.

This isolates skill; it doesn't forecast. It explains same-season AVG from same-season peripherals, not next season's average.

## Run it

```bash
pip install -r requirements.txt
jupyter notebook notebooks/batting_average_model.ipynb
```

`notebooks/BA_dict.ipynb` is a standalone warm-up (name to batting-average lookup), not part of the modeling pipeline. Trained models live in `models/`, output tables in `data/predictions/`.
