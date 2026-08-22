# CLAUDE.md

Guidance for AI when working in this repository.

## Project overview

Object-oriented toolkit for analyzing how weather affects F1 lap performance during the 2022–2025 ground-effect regulatory era. Two studies share a single OOP backbone:

- **wind** — head/cross wind effect on lap times (Azerbaijan in-domain; Saudi Arabia cross-domain)
- **temp** — air/track temperature effect on tyre degradation (Singapore in-domain; Las Vegas cross-domain)

## Project layout

```
F1_LapTime_Prediction/
├── f1lab/                 # OOP package
│   ├── __init__.py        # Facade: single import surface (re-exports the public classes)
│   ├── preprocessing.py   # BaseLapPreprocessor (ABC + Template Method + self-registry)
│   ├── experiments.py     # WindPreprocessor, TempPreprocessor (auto-register via experiment_name)
│   ├── models.py          # Per-model estimator factories + GridSearchCV grids (dt/rf/xgb)
│   ├── modeling.py        # ModelTrainer (model-agnostic), ModelEvaluator, Metrics, EvaluationResult
│   ├── visualization.py   # Visualizer (static methods: diagnostics + PDP/ICE)
│   ├── strategy.py        # UndercutScenario, ScenarioResult (PDP correction term + in-support/OOD guard)
│   └── data.py            # FastF1Downloader, RaceDataMerger
├── scripts/               # CLI entry points (use _bootstrap to add project root to sys.path)
│   ├── _bootstrap.py      # adds project root to sys.path so `f1lab` imports resolve
│   ├── download.py        # --start --end
│   ├── merge.py           # gp_name --years [--driver]
│   ├── train.py           # --experiment {wind,temp} --model {dt,rf,xgb,all} [--save | --quick] [--no-plots | --save-plots-dir DIR]
│   ├── evaluate.py        # --experiment {wind,wind-cross-domain,temp,temp-cross-domain} --model {dt,rf,xgb,all} [--no-plots | --save-plots-dir DIR]
│   ├── summarize.py       # parse logs/*.log → summary/metrics.csv + summary/best_params.csv
│   ├── mechanism.py       # PDP/ICE figures + symmetric undercut scenarios
│   ├── export_pdp_cache.py # RF PDP curves → demo/pdp_cache.json (lets the demo run model-free)
│   └── diagrams.py        # English architecture + research-flow diagrams (matplotlib, → docs/img/)
├── tests/                 # pytest suite (synthetic fixtures — no race data or network needed)
├── notebooks/             # 01_eda.ipynb (data-only EDA) + 02_walkthrough.ipynb (pretrained models; no training/network)
├── demo/                  # pdp_cache.json (exported PDP curves) + demo notes; the app is ./app.py
├── docs/img/              # committed English diagrams (exempt from the global *.png ignore)
├── data/                  # gitignored: raw/ + merged/
├── models/                # gitignored: *.joblib (full run produces 6 files)
├── logs/                  # gitignored: training/evaluation *.log
├── summary/               # gitignored: metrics.csv + best_params.csv (from summarize.py)
├── plots/                 # gitignored: diagnostic + PDP/ICE PNGs
├── legacy/                # READ-ONLY frozen snapshot: original .joblib + standalone scripts
├── app.py                 # Streamlit undercut demo (OOD-refusal UX; reads demo/pdp_cache.json)
├── main.py                 # one-command full pipeline (python main.py; --dry-run previews)
├── pyproject.toml         # packaging (pip install -e .) + ruff config
├── requirements.txt       # exact pinned dependency lock
├── .gitignore
├── README.md              # human-facing project description
└── CLAUDE.md              # this file
```

## Setup & run

Activate venv (Windows):

```powershell
.\.venv\Scripts\Activate.ps1   # PowerShell
.venv\Scripts\activate.bat     # CMD
source .venv/Scripts/activate  # bash
```

Required merged files (in `data/merged/`):

```
2022-2024_Azerbaijan_Grand_Prix.xlsx    2022-2024_Singapore_Grand_Prix.xlsx
2025_Azerbaijan_Grand_Prix.xlsx         2025_Singapore_Grand_Prix.xlsx
2025_Saudi_Arabian_Grand_Prix.xlsx      2025_Las_Vegas_Grand_Prix.xlsx
```

To regenerate from scratch: `download.py --start 2022 --end 2025` then `merge.py <GP_Name> --years <years>`.

Install: `pip install -r requirements.txt` (exact locked versions) or `pip install -e .[dev]` (editable package + pytest/ruff). `scripts/_bootstrap.py` keeps the scripts runnable without installing.

### Full symmetric run (6 trainings + 24 evaluations + 1 summary + 1 mechanism)

`python main.py` executes the full 32-command sequence (`--dry-run` prints it without running). Structure:

- **6 training runs** (3 models × 2 studies): DT, RF, XGB on Azerbaijan and Singapore training data
- **24 evaluation runs** (3 models × 4 conditions × 2 studies): each model evaluated on in-domain + cross-domain × raw + bias-corrected
- **1 summary run** (`scripts/summarize.py`): parses all 30 log files and writes `summary/metrics.csv` (24 rows) + `summary/best_params.csv` (6 rows)
- **1 mechanism run** (`scripts/mechanism.py`): generates the 8 PDP/ICE figures + prints the two symmetric undercut-scenario tables (depends only on the 6 trained models, not on the summary step)

Estimated wall-clock on 8-core CPU:

| Model | Grid combos | Time / study | Total (both studies) |
|---|---|---|---|
| DT | 648 | ~30 min | ~1 h |
| RF | 1,875 | ~26 h | ~52 h |
| XGB | 2,160 | ~10-13 h | ~20-26 h |
| **Sum** | | | **~75-80 h** |

DT is intentionally first in `main.py`'s `MODELS` list to act as a fast pipeline sanity check before committing 16+ hours to RF.

Both `train.py` and `evaluate.py` support `--save-plots-dir DIR` (mutually exclusive with `--no-plots`), and `main.py` uses it for all 30 train/evaluate commands. After a full run, `plots/` contains:

- **18 PNG from training** (6 trained models × 3 plot types — feature_importance, prediction_vs_actual, residual_distribution on the validation set), filename `train_{study}_{model}_<type>.png`
- **216 PNG from evaluation** (24 evals × 9 plot types — prediction_vs_actual, residual_distribution, residual_vs_predicted, residual_vs_feature×6 on the test set), filename `eval_{study}_{domain}_{mode}_{model}_<type>.png`

The 6 residual-vs-feature features per study are: 2 causal axes (`HeadWind`/`CrossWind` or `AirTemp`/`TrackTemp`) + 4 race/stint progression features (`LapNumber`, `LapInStint`, `TyreLife`, `TyreLifeNorm`).

**Total: 234 PNG, disk footprint ~40-60 MB**, all filenames unique (zero overlap between `train_` and `eval_` prefixes). `plots/` is gitignored.

Test suite: `python -m pytest tests/ -q` — 16 deterministic tests on synthetic fixtures (no network, no `data/` dependency). Linter: `ruff check .` (config in pyproject.toml). `train.py --quick` smoke-runs the real training pipeline on tiny 2-combo grids (~15 s for `--model all`); it is mutually exclusive with `--save` so a smoke run can never overwrite the real models. Use `--no-plots` for headless verification when plots are not needed.

## OOP architecture

Four GoF patterns are used:

| Pattern | Where | Purpose |
|---|---|---|
| Template Method | `BaseLapPreprocessor.run()` | Locks pipeline order; subclasses fill in hooks |
| Registry / Open-Closed | `BaseLapPreprocessor._registry` + `__init_subclass__` | New experiments self-register without touching base |
| Strategy | `ModelTrainer(preprocessor)` / `ModelEvaluator(model, preprocessor_cls)` | Inject preprocessor at runtime |
| Facade | `f1lab/__init__.py` | Single import surface for the package |

### `BaseLapPreprocessor` (ABC, template method, self-registry)

`run()` orchestrates a fixed pipeline:

1. `_sort()` — by `RaceYear → GPName → Driver → LapNumber`
2. `_engineer_common_features()` — `LapInStint`, `TyreLifeNorm`, `FuelLoad` (computed on **unfiltered** data so stint positions reflect each lap's true place in the original stint)
3. `_filter_invalid_laps()` — drop pit laps, non-green-flag (`TrackStatus != '1'`), and the lap immediately following either
4. `_engineer_specific_features()` — subclass hook (no-op by default)
5. `_compute_target()` — `LapTimeDelta = LapTime − min(LapTime per driver-stint)` in seconds
6. `_build_feature_matrix()` — select `feature_columns`, one-hot encode `Compound`, align to `features=` list if provided

Subclasses must override `feature_columns` (required) and may override `_engineer_specific_features` (optional). They self-register by setting `experiment_name`:

```python
class FooPreprocessor(BaseLapPreprocessor):
    experiment_name: ClassVar[str] = "foo"   # auto-registers
    @property
    def feature_columns(self) -> list[str]: ...
```

Lookup: `BaseLapPreprocessor.get("wind")` or the back-compat shim `get_preprocessor("wind")`.

Feature design is **causal axis × baseline controls**:

- `BaseLapPreprocessor.BASE_FEATURES` (ClassVar, 9 features) — shared controls: `LapNumber`, `LapInStint`, `Compound`, `TyreLife`, `TyreLifeNorm`, `FreshTyre`, `FuelLoad`, `Humidity`, `Rainfall`
- `WindPreprocessor` (`experiment_name="wind"`) — extends with `HeadWind`, `CrossWind` (wind is the explanatory variable under study)
- `TempPreprocessor` (`experiment_name="temp"`) — extends with `AirTemp`, `TrackTemp` (temperature is the explanatory variable under study)

Both subclasses end up with **11 features** — fully symmetric. Each study's "specific" features are the variables it claims as cause; the baseline is everything else.

The tyre-use axis carries three complementary features (`TyreLife`, `TyreLifeNorm`, `LapInStint`): raw FastF1 `TyreLife` gives absolute tyre age (carries across stints for inherited tyres), `TyreLifeNorm` gives stint-relative position (0~1), and `LapInStint` gives the lap's absolute position in the original stint. An earlier `TyreLifeTemp = TyreLifeNorm × TrackTemp` interaction was removed because (a) it back-doored `TrackTemp` into the wind model, violating the temp-only principle, and (b) tree-based models (DT/RF/XGB) can learn the interaction implicitly from main effects.

All shared constants (`GROUP_KEYS`, `SORT_KEYS`, `FUEL_START_KG`, `BASE_FEATURES`, etc.) are typed `ClassVar` to make immutable class-level intent explicit.

### `ModelTrainer`

Model-agnostic trainer. Selects estimator + grid by name (`"dt"`, `"rf"`, `"xgb"`) via `f1lab.models.get_model_spec()`. State checked via `is_fitted` property + `_require_fitted()` guard. After `fit()`, `cv_results_` is exposed so boundary-pinning diagnostics (e.g. R² gap between `n_estimators=2000` and `1500`) can be queried.

Steps in `fit()`:

1. `preprocessor.run()` to get X, y
2. `train_test_split(shuffle=False, test_size=0.2)` — held-out validation set
3. `GridSearchCV(TimeSeriesSplit(5), scoring='r2', n_jobs=-1)` on x_train, wrapping the estimator returned by the model factory

Public surface: `fit()`, `predict_valid()`, `report()`, `save(path)`, `is_fitted`, `cv_results_`, `best_params`, `model_name`.

`Metrics` dataclass (returned by `report()` and stored in `EvaluationResult`) carries 4 fields: `mae`, `mse`, `rmse` (= `sqrt(mse)`, in seconds — same unit as the target), and `r2`. `Metrics.compute(y_true, y_pred)` is the single construction point; `report()` prints all four at `.3f` precision. `DEFAULT_MODEL` (ClassVar on `ModelTrainer`) is `"xgb"` — applies only when `model_name=` is omitted from the `ModelTrainer` constructor; both `train.py` and `evaluate.py` CLI default to `"all"`, and `main.py` always specifies the model explicitly. `ModelTrainer(..., quick=True)` swaps the tiny smoke grids in via `get_model_spec(name, quick=True)`.

### Model specifications (`f1lab/models.py`)

Three models share the same training pipeline. Each grid is designed with boundary safety margins; see `f1lab/models.py` inline comments for the per-axis rationale. `get_model_spec(name, quick=False)` returns the full grid by default; `quick=True` returns a 2-combo smoke grid per model (used by `train.py --quick`).

**Decision Tree (`dt`)** — 648 combinations:

```python
"max_depth":          [None, 3, 5, 10, 20, 30]   # None and 30 at upper margin
"min_samples_leaf":   [1, 5, 10, 30, 90, 150]    # 1 = sklearn default
"min_samples_split":  [2, 10, 30, 50, 100, 200]  # 2 = sklearn default
"max_features":       [None, "sqrt", 0.5]        # None = all features
```

**Random Forest (`rf`)** — 1,875 combinations:

```python
"n_estimators":      [200, 500, 1000, 1500, 2000]
"max_depth":         [None, 3, 5, 10, 20]
"min_samples_leaf":  [1, 5, 10, 30, 90]
"min_samples_split": [2, 10, 30, 50, 100]
"max_features":      ["sqrt", 0.5, 1.0]
```

Design notes: `max_features` is RF's primary decorrelation knob; `min_samples_split` reaches the sklearn default `2`; `n_estimators` extends to `2000` so the upper bound has plenty of margin above the practical 1000-tree plateau — if `2000` is selected, the `cv_results_` gap to `1500` quantifies whether the upper bound is binding.

**XGBoost (`xgb`)** — 2,160 combinations:

```python
"n_estimators":     [200, 500, 1000, 1500, 2000]   # matches RF
"max_depth":        [3, 6, 10, 15]                 # 6 = XGBoost default
"learning_rate":    [0.05, 0.1, 0.2, 0.3]          # 0.3 = XGBoost default
"subsample":        [0.7, 0.85, 1.0]               # 1.0 = default
"colsample_bytree": [0.7, 0.85, 1.0]               # 1.0 = default
"min_child_weight": [1, 5, 10]                     # 1 = default
```

Design notes: XGBoost has 3 further commonly-tuned axes (`reg_alpha`, `reg_lambda`, `gamma`) not in the grid — they are typically fine-tuning knobs, and including them would push combinations to ~20,000+. The defaults are reachable on every axis so a "default-pin" outcome is defensible as a finding rather than a grid limitation.

Estimator construction: `tree_method="hist"` + `n_jobs=-1` for speed + parallelism; `random_state=42` for reproducibility.

All three models share `random_state=42` and `n_jobs=-1` (where applicable) on both the estimator and the outer `GridSearchCV`.

### `ModelEvaluator`

Loads a saved `.joblib`, exposes `feature_names` property, runs preprocessor with `features=self.feature_names` for column alignment, predicts, and returns an `EvaluationResult`. With `bias_correct=True` it adds a median-residual offset and reports both raw and corrected metrics.

### `Visualizer`

Utility class with static methods (no instance state). Every method accepts an optional `save_path=`; when provided the figure is written to disk at 150 DPI and closed (interactive `plt.show()` is skipped). When omitted the figure is shown interactively.

| Method | Purpose |
|---|---|
| `feature_importance(model, columns)` | Bar chart of `model.feature_importances_` (RF/DT/XGB compatible) |
| `prediction_vs_actual(y_true, y_pred)` | Sequential line plot of actual vs predicted |
| `residual_distribution(y_true, y_pred)` | Histogram + KDE of residuals (marginal view) |
| `residual_vs_predicted(y_true, y_pred)` | **Scatter** of residuals against predicted values + LOWESS overlay — surfaces burst values, heteroscedasticity, systematic bias |
| `residual_vs_feature(y_true, y_pred, feature_values, feature_name)` | **Scatter** of residuals against a single feature + LOWESS — locates *where in feature space* the model fails |
| `partial_dependence(model, x, feature)` | PDP (average marginal effect) curve — extracts the response function f(feature) → LapTimeDelta that the strategy correction term reads off |
| `ice(model, x, feature, subsample=200)` | ICE lines + PDP overlay — checks the average reaction isn't pulled by a few samples (per-stint curves should share the shape) |

The two residual-scatter methods require `statsmodels` for the LOWESS overlay; without it, plots still render but no smoothed line is drawn. `partial_dependence`/`ice` wrap `sklearn.inspection.PartialDependenceDisplay`; the `ice` call emits a benign "No artists with labels" legend warning (pre-existing, cosmetic — the PDP overlay still draws).

### `UndercutScenario` / `ScenarioResult` (`f1lab/strategy.py`)

The strategy-application layer. A study's RF model + training feature matrix define a `UndercutScenario`; `evaluate(current_value)` returns a `ScenarioResult`.

Per-lap correction term: `Δ = f(current_value) − f(training_mean)`, read off the precomputed PDP grid by linear interpolation. The correction is applied to a pit-stop undercut decision **iff** `current_value` lies inside the **training support** (`[feature.min(), feature.max()]` — the range the RF actually saw and split on). Outside it the input is OOD: the PDP is undefined, so the correction is *withheld* (`delta_per_lap`/`gap_corrected`/`decision_corrected` = None, `applicable` = False) rather than silently extrapolated.

`net`, `gap`, and the *uncorrected* decision are always computed (pure arithmetic / strategy params — not model-dependent); only the **correction term and the corrected gap** depend on in-support. This is the precise meaning of "OOD → withheld": gap is never refused, the correction is.

The two studies instantiate the SAME class with IDENTICAL fixed params (`pit_loss=20.0, gap=19.50, out=95.2, in=95.5, N=10`); they differ only in the in-support (wind/Saudi) vs OOD (temp/Las Vegas) verdict — see "PDP/ICE figures + strategy scenarios" below.

The model is touched **exactly once**, in `__init__`, to build the PDP grid; `evaluate()` afterwards reads only that grid, `support`, and `training_mean` (both stored at construction). `to_cache()` exports those model-derived values as plain JSON types and `from_cache()` rebuilds an equivalent scenario without a model — verified field-for-field identical (all 20 `ScenarioResult` fields) against the model path on both studies. `scripts/export_pdp_cache.py` writes `demo/pdp_cache.json` (6.7 KB: 100 HeadWind grid points, 49 TrackTemp), which is what the Streamlit app loads — so the deployed demo ships no `.joblib` and no race data, and boots in seconds instead of recomputing a ~1 min RF PDP. **Re-run the export after any retrain**, or the demo keeps showing the previous model's PDP.

## Feature engineering

| Feature | Definition |
|---|---|
| `LapInStint` | Lap's original position within driver-stint (computed pre-filter; surviving laps may show "gaps" where SC/pit laps were removed) |
| `TyreLife` | FastF1 raw absolute tyre age (laps used; may exceed `LapInStint` for inherited tyres) |
| `TyreLifeNorm` | `TyreLife / max(TyreLife within driver-stint)`, capped > 0; stint-relative position |
| `FuelLoad` | `110 − LapNumber × 1.7` kg (Cappello, 2025) |
| `HeadWind` (wind-only) | `WindSpeed × cos(WindDirection°)` |
| `CrossWind` (wind-only) | `WindSpeed × sin(WindDirection°)` |
| `AirTemp`, `TrackTemp` (temp-only) | raw FastF1 weather columns |
| `Compound_*` | One-hot |
| **target** | `LapTime − min(LapTime within driver-stint)` in seconds |

## Methodology — DO NOT VIOLATE

- **Train**: first 80% of 2022–2024 (time-ordered)
- **Validation**: last 20% of 2022–2024 (within-era held-out)
- **Test**: full 2025 race (truly unseen)
- `TimeSeriesSplit(n_splits=5)` runs *inside* the 80% training portion

The 2022–2025 ground-effect regulations are one regulatory era. 2025 must remain held out so within-era generalization can be measured. Year-on-year drift (median residual ~−0.17 s for wind, ~−0.37 s for temp) is a finding, not a bug — handle via post-hoc median-residual bias correction (`ModelEvaluator.evaluate(..., bias_correct=True)`), never by mixing 2025 into training.

## Saved models — `legacy/` is READ-ONLY

`legacy/` (gitignored) holds the original pre-refactor models and standalone scripts as a frozen reference snapshot. **Never write, edit, or delete anything inside `legacy/`.**

Active training writes to `models/`. Each study saves three model files (one per model class): `{circuit}_{model}.joblib` for circuit ∈ {azerbaijan, singapore} and model ∈ {dt, rf, xgb}, i.e. six files total after a full run.

## Verified metrics (full-run results — the numbers of record)

`summary/metrics.csv` = 24 rows; `summary/best_params.csv` = 6 rows. RF is the headline model in both studies.

**Hold-out model selection (2022–2024 last 20%):**

| Study | Model | hold-out R² | MAE | RMSE | best_params |
|---|---|---|---|---|---|
| wind | DT | 0.663 | 0.614 | 0.982 | max_depth=10, max_features=sqrt, min_samples_leaf=1, min_samples_split=50 |
| wind | RF | 0.734 | 0.562 | 0.871 | max_depth=None, max_features=0.5, min_samples_leaf=5, min_samples_split=2, n_estimators=1500 |
| wind | XGB | **0.799** | 0.554 | 0.759 | colsample_bytree=1.0, learning_rate=0.05, max_depth=3, min_child_weight=5, n_estimators=200, subsample=0.7 |
| temp | DT | 0.648 | 0.575 | 0.838 | max_depth=3, max_features=0.5, min_samples_leaf=1, min_samples_split=2 |
| temp | RF | **0.685** | 0.522 | 0.793 | max_depth=10, max_features=sqrt, min_samples_leaf=1, min_samples_split=2, n_estimators=500 |
| temp | XGB | 0.668 | 0.542 | 0.813 | colsample_bytree=0.7, learning_rate=0.05, max_depth=3, min_child_weight=1, n_estimators=500, subsample=0.85 |

RF sits within 0.07 of the best hold-out R² in both studies (wind: RF 0.734 vs XGB 0.799, gap 0.065; temp: RF is the winner) → no decisive winner → **RF kept as common main model** (PDP stability + variance reduction). Wind XGB edges RF on hold-out, but RF wins on the 2025-test generalization that matters (temp RF test R² 0.420 vs XGB 0.254; cross −6.080 vs −17.420).

**2025 test metrics — MAE / MSE / RMSE / R²:**

*Wind (Azerbaijan→Saudi Arabia/Jeddah):*

| Setting | DT | RF | XGB |
|---|---|---|---|
| Azerbaijan in-domain raw | 0.526/0.505/0.711/−0.042 | 0.457/0.383/0.619/0.209 | 0.463/0.377/0.614/0.221 |
| Azerbaijan in-domain bias | 0.516/0.506/0.711/−0.044 | 0.435/0.395/0.628/0.186 | 0.423/0.381/0.617/0.213 |
| Saudi cross-domain raw | 0.516/0.522/0.723/−0.017 | 0.460/0.430/0.656/0.163 | 0.465/0.437/0.661/0.149 |
| Saudi cross-domain bias | 0.496/0.520/0.721/−0.013 | 0.435/0.432/0.657/0.159 | 0.440/0.436/0.660/0.151 |

*Temp (Singapore→Las Vegas):*

| Setting | DT | RF | XGB |
|---|---|---|---|
| Singapore in-domain raw | 0.850/2.813/1.677/0.378 | 0.885/2.621/1.619/0.420 | 1.281/3.375/1.837/0.254 |
| Singapore in-domain bias | 0.835/2.912/1.706/0.356 | 0.821/2.718/1.649/0.399 | 0.932/2.967/1.722/0.344 |
| Las Vegas cross-domain raw | 1.821/4.192/2.047/−8.244 | 1.516/3.211/1.792/**−6.080** | 2.346/8.353/2.890/−17.420 |
| Las Vegas cross-domain bias | 0.781/1.015/1.008/−1.239 | 0.817/1.123/1.060/−1.477 | 1.590/3.437/1.854/−6.580 |

bias_offset (median residual subtracted) for RF: wind in-domain −0.17, cross −0.18; temp in-domain −0.37, cross −1.32.

**Headline findings:** wind RF generalizes across tracks (Saudi cross-domain bias R²=0.159 ≈ Azerbaijan in-domain 0.186 — physics-compatible regime); temp RF fails catastrophically cross-domain (raw R²=−6.080) because Las Vegas ~17°C is OOD vs Singapore's 27.6–37.4°C training range. This contrast is the core result — the single-mechanism isolation framework gives usable extrapolation when physics-compatible and honest refusal when OOD.

## PDP/ICE figures + strategy scenarios (`scripts/mechanism.py` output)

`scripts/mechanism.py` generates the 8 PDP/ICE figures and runs the two **symmetric** undercut scenarios. Both studies use the **RF** model on 2022–2024 training data; both scenarios share identical fixed params and differ only in in-support (wind/Saudi) vs OOD (temp/Las Vegas).

**Figures (8, written to `plots/`; filenames keep a historical `figure_6_*` numbering scheme):**
- Wind (Baku 2022–2024, `X=(2248, 13)`) → `figure_6_1_pdp_HeadWind`, `figure_6_2_pdp_CrossWind`, `figure_6_3_ice_HeadWind`, `figure_6_4_ice_CrossWind`
- Temp (Singapore 2022–2024, `X=(2366, 14)`) → `figure_6_5_pdp_AirTemp`, `figure_6_6_pdp_TrackTemp`, `figure_6_7_ice_AirTemp`, `figure_6_8_ice_TrackTemp`

**Feature ranges (training data):**

| Study | Feature | Range | Mean | Span |
|---|---|---|---|---|
| Wind | HeadWind | [−2.24, +3.60] m/s | +0.18 | 5.84 |
| Wind | CrossWind | [−2.74, +3.79] m/s | −0.08 | 6.53 |
| Temp | AirTemp | [26.80, 31.20] °C | 29.74 | 4.40 |
| Temp | TrackTemp | [27.60, 37.40] °C | 34.36 | 9.80 |

**Two symmetric undercut scenarios — computed by `scripts/mechanism.py`:**

- **Shared fixed params (identical for both)**: `pit_loss = 20.0 s` (illustrative, typical street-circuit magnitude); `gap (uncorrected) = 19.50 s`; our new-tyre out-lap = `95.20 s`; rival old-tyre in-lap = `95.50 s`; rival remaining laps = `10` → `net = 20.0 + 95.20 − 95.50 = 19.70 s`; margin = gap − net = `−0.20` → **CLOSE** (both, uncorrected).
- **(1) Saudi Arabia 2025, wind (cross-domain, in-support)**: model baseline = **Azerbaijan** training mean HeadWind `+0.18`; "current" = **Saudi 2025 measured MAX headwind `+1.38`** (an undercut is a single-lap decision → use the wind at that moment; the Saudi race **mean is −0.31**, which sits in the flat PDP region and would give Δ≈0). `+1.38` is IN the Baku training support `[−2.24, +3.60]`. PDP at `+0.18` = `+1.108 s`, at `+1.38` = `+1.147 s` (the PDP plateaus above +1.2); Δ/lap `+0.039 s`; ×10 = `+0.389 s`; corrected gap `19.89 s` → net < gap → **OPEN**. → **Decision flips CLOSE → OPEN** on real measured wind.
- **(2) Las Vegas 2025, temperature (cross-domain, OOD)**: model baseline = Singapore training mean; "current" = **Vegas 2025 measured mean TrackTemp `17.27 °C`** (real). Training support `[27.60, 37.40]` → `17.27` is OUT of support → PDP undefined → correction **withheld**, decision **cannot be evaluated** (corroborated by the cross-domain R²=−6.080).
- The asymmetry is *only* the in-support vs OOD verdict; pit_loss/net/gap are identical, both scenarios computed, and both environmental conditions are read from REAL 2025 data (no assumed values). `scripts/mechanism.py` loads `2025_Saudi_Arabian_Grand_Prix.xlsx` / `2025_Las_Vegas_Grand_Prix.xlsx` for the current conditions. (Strategy-desk params pit_loss/gap/out-lap/in-lap remain illustrative — they are not telemetry-measurable.)

**Reproduce:** `python scripts/mechanism.py` — deterministic (`RANDOM_STATE=42`, models loaded from `models/*.joblib`), regenerates the 8 figures and prints the feature-range table + both scenario tables. Re-running after a model retrain re-syncs figures and numbers. The console output is the source of record for the scenario numbers.

**Why OOD → withhold the correction (3 layers, weak→strong):**

First, the precise claim: `net` (19.70) and `gap` (19.50) and the uncorrected CLOSE decision are *always computed* — they are strategy params + arithmetic, not model outputs. What is withheld for the temp scenario is the **correction term Δ** (hence the *corrected* gap). gap is never refused; the correction is.

1. **PDP is only valid inside the training range.** A PDP is the model's average reaction over the *observed* feature distribution. Las Vegas TrackTemp ≈ 17°C is below the training minimum 27.6°C — the RF's decision trees never split there, so asking for a value at 17°C forces the input into the nearest leaf and returns a boundary-region average, not a genuine low-temperature response. Δ has no statistical support.
2. **The evaluation study already proved it empirically.** Temp model cross-domain (Las Vegas) raw R² = **−6.080** — an order of magnitude worse than guessing the mean. So even if a Δ were force-computed, it is known-wrong.
3. **"Knowing when not to predict" is the design contribution.** A black-box predictor silently emits a wrong number at OOD and the user can't tell it failed; this design explicitly detects the out-of-support condition and refuses, preserving human judgement. Refusing a correction it can't support is the responsible behaviour.

Contrast (why wind *does* apply): current HeadWind +1.38 m/s (Saudi 2025 measured max) lies inside the training support [−2.24, +3.60] → PDP has statistical support → Δ=+0.039 is trustworthy and gets applied. The applicability test is simply: is the current value within the observed training range?

## Feature-design rationale

The 11-feature symmetric design (9 baseline + 2 study-specific) is intentional:

- **`LapNumber`** is in baseline (not study-specific) because race progression is a confounder for any lap-time prediction — affects both tyre wear accumulation and fuel-load dynamics, regardless of whether wind or temperature is the variable of interest
- **`Humidity`, `Rainfall`** are in baseline because weather state affects air density (→ aero drag, → lap time) and surface grip (→ lap time) for both studies
- **`AirTemp`, `TrackTemp`** are temp-only because they ARE the variable under investigation in that study. Including them in the wind study would muddy the wind→performance causal claim (would let the temp study's signal leak into the wind model's coefficients)
- **`HeadWind`, `CrossWind`** are wind-only for the symmetric reason

The principle is: **anything in `BASE_FEATURES` is a control variable; anything study-specific is the cause being claimed**.

## Improvements that DO NOT apply

- **Sector time features** (Anandatama, 2025): their F1-score gain is for *compound classification*, where SectorTimes are valid covariates. For our regression target `LapTime − min(LapTime)`, sector times sum to ≈ LapTime → data leakage. Do not add.

## Dependencies

Runtime deps (exact lock in `requirements.txt`, Python 3.12): fastf1 3.8.1, pandas 2.3.3, numpy 2.4.3, scikit-learn 1.8.0, xgboost 3.2.0, joblib 1.5.3, matplotlib 3.10.8, seaborn 0.13.2, statsmodels 0.14.6, openpyxl 3.1.5 — all in `.venv/`. Install floors live in `pyproject.toml`; dev extras (`pip install -e .[dev]`) add pytest + ruff + ipykernel (for re-running the notebook); the `demo` extra adds streamlit. (`statsmodels` powers the LOWESS overlay on residual scatter plots; if missing, plots still render without the smoothed line.)
