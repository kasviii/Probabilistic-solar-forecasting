# Probabilistic Solar Forecasting with Quantile Regression Forests

Reproduction of: *"Short-term forecasting of solar irradiance using decision
tree-based models and non-parametric quantile regression"* (PLOS ONE, 2024).
[Paper link](https://journals.plos.org/plosone/article?id=10.1371/journal.plos.one.0312814) ·
[Reference implementation studied](https://github.com/energyCUEE/probforecast)

## What this is

The paper trains a Quantile Regression Forest (QRF) to forecast solar
irradiance (GHI) one hour ahead, producing not just a point forecast but a
full predictive distribution — useful for grid operators who need to know
*how confident* a forecast is, not just what it is. The original paper
validates this on a single location. This repo reproduces the core method
and then **extends it to three climatically distinct sites** to test whether
the method's uncertainty calibration holds up outside the conditions it was
built on.

**Short answer: point-forecast accuracy holds up fine across climates.
Uncertainty calibration does not.**

## Method

- **Baseline**: plain Random Forest regressor, point forecast only.
- **QRF (paper's method)**: `RandomForestQuantileRegressor` from the
  [`quantile-forest`](https://pypi.org/project/quantile-forest/) library,
  trained to predict the 10th/25th/50th/75th/90th percentiles of GHI.
  The 50th percentile is the point forecast; the 10th-90th range is an
  80% prediction interval.
- **Features**: hour of day, day of year, month, GHI at t-1/t-2/t-3/t-24
  (same hour previous day), 3-hour rolling mean of GHI, temperature, wind
  speed, and a clear-sky index (GHI / clear-sky GHI, a standard proxy for
  cloud cover derived from NSRDB's modeled clear-sky value).
- **Split**: chronological 80/20 train/test — no shuffling, since this is
  time series.
- **Metrics**:
  - MAE / RMSE / R² for point-forecast accuracy.
  - **PICP** (Prediction Interval Coverage Probability) — what fraction of
    true values actually fall inside the predicted 10-90% band. Should be
    ≈0.80 if well calibrated.
  - **PINAW** (Prediction Interval Normalized Average Width) — how wide the
    interval is, normalized by the target's range. Narrower is better,
    *conditional on* PICP being correct — a narrow but wrong interval is
    worse than a wide but honest one.
  - **Pinball loss** at each quantile — the proper scoring rule for
    quantile forecasts.
  - PICP/PINAW are reported **daytime-only** (GHI > 5 W/m²). Nighttime
    hours are trivially GHI≈0 for every quantile and inflate coverage
    numbers without saying anything about forecast quality — including
    them would make every site look better calibrated than it is.

## Data

Three sites, all from NREL's National Solar Radiation Database (NSRDB),
chosen to span distinct climate/cloud regimes:

| Site | Climate | Source | Years |
|---|---|---|---|
| Izmir, Turkey | Mediterranean | [Kaggle: NREL NSRDB solar radiation dataset](https://www.kaggle.com/datasets/ibrahimkiziloklu/solar-radiation-dataset) | 2017-2019 |
| Phoenix, Arizona | Desert | NREL/NLR API, GOES Aggregated v4.0.0 | 2019 |
| Mumbai, India | Tropical / monsoon | NREL/NLR API, Meteosat IODC (PSM v3) | 2019 |

Note: NREL's developer API domain migrated from `developer.nrel.gov` to
`developer.nlr.gov` in 2026 as part of the lab's rebrand to the National
Laboratory of the Rockies (NLR). All API calls in the notebook use the
current domain.

## Results

### Point-forecast accuracy (baseline Random Forest)

| Site | MAE (W/m²) | RMSE (W/m²) | R² |
|---|---|---|---|
| Izmir | 6.87 | 17.41 | 0.9968 |
| Phoenix | 19.49 | 41.90 | 0.9638 |
| Mumbai | 27.39 | 50.23 | 0.9604 |

R² stays high everywhere, but error magnitude roughly quadruples from
Izmir to Mumbai — consistent with Mumbai's monsoon-driven irradiance
being inherently less persistent hour-to-hour than Izmir's smoother
Mediterranean cloud patterns.

### QRF calibration (10-90% interval, daytime only)

| Site | PICP (target ≈0.80) | PINAW | Pinball @ 50% |
|---|---|---|---|
| Izmir | **0.919** (overcovers) | 0.055 | 3.38 |
| Phoenix | **0.831** (well-calibrated) | 0.194 | 8.64 |
| Mumbai | **0.723** (undercovers) | 0.191 | 13.51 |

### Feature importance

`GHI_lag_1` (the most recent hour's irradiance) dominates at ~0.82
importance across sites — a persistence effect well known in solar
forecasting literature. `GHI_lag_24` and `hour` pick up most of the rest.
Weather covariates (temperature, wind speed) and `month` contribute almost
nothing at this 1-hour-ahead horizon; they may matter more for longer
horizons, which is a natural next step.

### Reliability diagram

Empirical coverage on Izmir sits above the diagonal at every nominal
confidence level (not just at 80%), confirming the overcoverage is a
systematic, direction-consistent miscalibration rather than noise.

## What we found (the extension)

The original paper validates QRF on one location. Testing it on three
climatically distinct sites shows that **the failure mode is calibration,
not point accuracy**:

- Daytime PICP for the nominal 80% interval goes **0.919 → 0.831 → 0.723**
  across Izmir → Phoenix → Mumbai — the same model architecture, retrained
  per site on its own data, swings from meaningfully overconfident-safe to
  meaningfully overconfident-risky depending on climate.
- This matters practically: a QRF interval that is a trustworthy ~92%
  band in Izmir becomes an unreliable ~72% band in Mumbai — silently
  overclaiming confidence in exactly the climate where irradiance is
  hardest to predict.
- Mumbai's poor calibration isn't from too narrow an interval in absolute
  terms — its PINAW (0.191) is close to Phoenix's (0.194), which *is*
  well-calibrated. The honest reading is that Mumbai's true GHI volatility,
  likely driven by fast monsoon cloud transitions, exceeds what even a wide
  band can reliably bound with this feature set.
- **Practical takeaway**: don't assume a single-site calibration (or a
  paper's reported PICP) transfers to a new climate zone. Site-specific
  recalibration — e.g. conformal adjustment of the quantile levels
  post-hoc, or monsoon-aware features capturing cloud rate-of-change —
  looks necessary for tropical/monsoon deployments specifically.

## Repo contents

- `solar_qrf_reproduction.ipynb` — full pipeline, runs end to end: data
  loading → preprocessing → baseline RF → QRF → all visualizations →
  3-site comparison.
- `data/` — not included (see Data section above for sources; Izmir via
  Kaggle download, Phoenix/Mumbai via NREL/NLR API — both reproducible
  with a free API key).
- `results_*.png` / `results_comparison_table.csv` — saved figures and
  the final comparison table, generated by the notebook.

## Limitations / honest caveats

- Single year of data for Phoenix and Mumbai (2019) vs. three years for
  Izmir — the comparison isn't perfectly apples-to-apples, and a
  longer record for the newer sites would strengthen the calibration
  findings.
- Only one forecast horizon (1 hour ahead) was tested. Calibration
  behavior may differ at longer horizons where persistence features
  matter less.
- No hyperparameter tuning per site — the same `n_estimators` and
  `min_samples_leaf` were used everywhere, which likely contributes to
  the Izmir overcoverage (a smaller `min_samples_leaf` might sharpen the
  Izmir interval without necessarily fixing Mumbai's undercoverage).
