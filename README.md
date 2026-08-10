# DataAnalyzer — API Documentation

**Package:** `aaa-eda-analyzer` (PyPI/install name) · **Import as:** `eda_analyzer`
**Version:** 0.2.3

`DataAnalyzer` provides an automated, end-to-end Exploratory Data Analysis (EDA)
pipeline. It ingests raw datasets, standardizes and maps variables into semantic
roles, computes statistical hypothesis tests, calculates non-linear feature
associations, dynamically selects optimal charts, and compiles self-contained
reports in HTML or PDF format.

---

## 1. Installation

```bash
pip install aaa-eda-analyzer
```

For PDF report generation (via [weasyprint](https://weasyprint.org/)):

```bash
pip install "aaa-eda-analyzer[pdf]"
```

Core dependencies (installed automatically): `pandas`, `numpy`, `seaborn`,
`matplotlib`, `scipy`, `scikit-learn`, `statsmodels`.

> **Note on naming:** the distribution name on PyPI is `aaa-eda-analyzer`, but
> the Python import name is `eda_analyzer`:
> ```python
> from eda_analyzer import DataAnalyzer
> ```
> This split between install name and import name is normal (same pattern as
> `pip install beautifulsoup4` → `import bs4`).

---

## 2. Supported Data Structures and Semantic Type Mapping

### Supported input data structures

The library standardizes incoming data into a pandas `DataFrame` via an
internal `_standardize_data()` step. Accepted formats:

| Input type | Handling |
|---|---|
| `pd.DataFrame` | Processed directly via an explicit deep copy |
| `pd.Series` | Converted into a single-column DataFrame |
| `pl.DataFrame` (Polars) | Converted via `.to_pandas()` |
| `np.ndarray` (1D) | Converted to a single column named `feature_0` |
| `np.ndarray` (2D) | Converted to columns `feature_0, feature_1, ..., feature_n` |
| `dict` / `list` | Ingested directly via `pd.DataFrame(data)` |

### Semantic type detection engine

Rather than relying strictly on native pandas dtypes, `DataAnalyzer`
dynamically categorizes each feature into one of four semantic types via
`_detect_semantic_types()`:

| Semantic type | Detection criteria | Used for |
|---|---|---|
| **datetime** | `datetime64` dtype, or an object column whose values parse as dates | Monthly trend plots in `plot_time_series()` |
| **continuous** | Numeric dtype, with unique values ≥ 15 **and** unique-value ratio ≥ 0.05 | Pearson correlation, Z-Score/IQR outlier analysis, ANOVA, VIF, regression plots |
| **categorical** | Numeric dtype with unique values < 15 **or** unique ratio < 0.05; or non-numeric dtype with unique values < 20 **or** unique ratio < 0.5 | Cramér's V, Chi-Square tests, Correlation Ratio (η), violin plots, stacked bar charts |
| **text** | Non-numeric dtype with high cardinality (unique ratio ≥ 0.5 or ≥ 20 unique values) | Excluded from statistical/plotting steps to avoid rendering errors |

---

## 3. Constructor Reference

```python
DataAnalyzer(
    data: Any,
    target_col: Optional[str] = None,
    exclude_cols: Optional[List[str]] = None,
    random_state: int = 42,
    verbose: bool = True,
    show_plots: bool = True,
    save_dir: Optional[str] = None,
    download: bool = False,
    export_pdf: bool = False
)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `data` | `Any` | required | Dataset to analyze (DataFrame, Series, array, dict, or list) |
| `target_col` | `str` | `None` | Target variable for predictive analysis; unlocks target-centric tests and charts |
| `exclude_cols` | `List[str]` | `None` | Column names to exclude entirely (e.g. `['user_id', 'transaction_id']`) |
| `random_state` | `int` | `42` | Seed for randomized sampling (e.g. Shapiro-Wilk), for reproducibility |
| `verbose` | `bool` | `True` | Log progress and text tables to the console |
| `show_plots` | `bool` | `True` | Render visualizations inline |
| `save_dir` | `str` | `None` | Directory to save generated image assets and reports |
| `download` | `bool` | `False` | Headless mode — suppresses inline rendering, saves everything to an HTML report |
| `export_pdf` | `bool` | `False` | If `True` (with `download=True`), also generates a PDF report via weasyprint |

---

## 4. Operational Rules and Guards

**Target column**
- `target_col` must match an existing column name, or the library logs a
  warning and falls back to target-agnostic mode.
- Target-centric methods (`run_bivariate_stats`, `compute_feature_importance`,
  `plot_bivariate`, `plot_time_series`) return safely with no exception if no
  target is set.

**Cardinality**
- `MAX_CAT_CARDINALITY = 10` — categorical variables with more than 10 unique
  levels are flagged in `data_quality_report()` and excluded from bivariate
  plotting, Cramér's V matrices, and group tests.

**Quasi-constant features**
- `QUASI_CONSTANT_THRESHOLD = 0.95` — a column (not truly constant) where a
  single value covers ≥95% of non-null rows is flagged in
  `data_quality_report()` as low-variance/quasi-constant, separate from the
  truly-constant check.

**Sample size / completeness guards**
- Correlation Ratio (η): only evaluated with ≥ 20 complete cases across 2+
  categories; otherwise returns `NaN`.
- Normality check: Shapiro-Wilk samples a maximum of 5,000 rows for continuous
  columns with more than 5,000 non-null values.
- High-missing exclusion: features with > 50% missing values (per
  `data_quality_report()`) are excluded from `compute_feature_importance()` to
  avoid excessive row loss from `dropna()`.
- VIF: requires at least 2 continuous features and `n ≥ (number of continuous
  features + 2)` complete rows.

**Filenames**
- All plot export filenames are sanitized via `_sanitize_filename()` — any
  non-alphanumeric character is replaced with `_` to avoid path/OS issues.

---

## 5. Core Methods

Run everything at once with `run_all()`, or call methods individually for a
modular workflow.

**Summarization & data quality**
- `summarize()` — row/column counts, memory usage, global missingness.
- `data_quality_report()` — duplicate rows, constant columns, features with
  > 50% missing, high-cardinality categoricals (> 10 levels), skewed
  continuous features (`|skew| > 1`), and quasi-constant columns (a single
  value covering ≥95% of rows).
- `plot_missingness()` — missing-value bar chart plus a null-location matrix
  heatmap.
- `plot_univariate(top_n=None, columns=None)` — per-column univariate profile.
  Summary stats (`univariate_summary_`) are always computed for **every**
  eligible column; only a limited number get plotted (histogram+boxplot for
  continuous, bar chart for categorical, count-over-time for datetime):
  - `top_n`: how many columns to plot. Defaults to `MAX_UNIVARIATE_PLOTS` (10).
    Pass `-1` for no cap. If `target_col` is set, columns are ranked by
    relevance to the target (Mutual Information) and the top_n most related
    are plotted, with the target itself always included. If no `target_col`
    is set, there's nothing to rank by — the first `top_n` columns in dataset
    order are plotted instead, with a warning logged.
  - `columns`: explicit list of column names to plot, in the given order —
    overrides the ranking. Invalid names are ignored with a warning.

**Statistical analysis**
- `detect_outliers()` — Z-Score (`|Z| > 3`) for normal data (per
  Shapiro-Wilk), IQR (`1.5×`) otherwise.
- `run_bivariate_stats()` *(requires `target_col`)* — tests every feature
  against the target, **checking assumptions before choosing a test** rather
  than applying one blindly:
  - Continuous vs. continuous: Pearson if both variables pass a Shapiro-Wilk
    normality check, otherwise falls back to Spearman.
  - Continuous vs. categorical: ANOVA if per-group normality and variance
    homogeneity (Levene) both hold, otherwise falls back to Kruskal-Wallis.
  - Categorical vs. categorical: Chi-Square, flagged as unreliable if too
    many contingency cells have expected count < 5 (Cochran's rule).

  Results include `Assumptions_Met` (bool) and `Notes` (why a fallback test
  was used, or why a result may be unreliable) columns.
- `compute_feature_importance()` *(requires `target_col`)* — Mutual
  Information ranking (classification or regression, auto-detected).
- `detect_multicollinearity()` — VIF via OLS R²; VIF > 5.0 flags high
  multicollinearity.

**Visualization engine**
- `plot_bivariate()` *(requires `target_col`)* — auto-selects the chart type:
  - continuous vs. continuous, large N → hexbin density
  - continuous vs. continuous, small N → scatter + regression line
  - continuous vs. categorical → violin plot
  - categorical vs. categorical → 100% stacked bar
- `plot_multivariate()` — Pearson correlation heatmap (continuous) and
  Cramér's V heatmap (categorical).
- `plot_time_series()` — monthly trend of `target_col` over any detected
  datetime column.

**Advanced interactions**
- `analyze_feature_interactions(top_n=5, threshold=0.0)` — pairwise
  associations between independent features (Pearson |r| or Spearman
  fallback, Cramér's V, Correlation Ratio η), plotted for the strongest
  `top_n` pairs. Each row includes `Assumptions_Met` and `Notes`, same
  fallback logic as `run_bivariate_stats()`.

---

## 6. Accessing Programmatic Results

Result attributes are stored as class attributes and end with a trailing
underscore by convention:

```python
analyzer.run_all()

analyzer.quality_report_       # dict of data-quality flags
analyzer.outlier_report_       # DataFrame of outlier detection metrics
analyzer.bivariate_stats_      # DataFrame of bivariate test results
analyzer.feature_importance_   # DataFrame ranking feature importance
analyzer.vif_report_           # DataFrame of VIF scores
analyzer.top_feature_pairs_    # DataFrame of top feature-feature interactions
analyzer.univariate_summary_   # DataFrame of per-column stats (all cols) + Plotted flag
```

---

## 7. Generating Downloadable Reports

All analyses, tables, and plots can be bundled into a single self-contained
HTML file (images embedded as base64, no external assets needed).

```python
import pandas as pd
from eda_analyzer import DataAnalyzer

df = pd.read_csv('housing_data.csv')

analyzer = DataAnalyzer(
    data=df,
    target_col='price',
    download=True,
    export_pdf=True,          # also exports a PDF via weasyprint
    save_dir='monthly_reports'
)

analyzer.run_all()
print(f"Report generated at: {analyzer.report_path_}")
```

### Custom pipeline execution

```python
analyzer.run_all(steps=['summarize', 'data_quality', 'missingness_plot', 'feature_importance'])
```

Available step identifiers: `'summarize'`, `'data_quality'`, `'outliers'`,
`'bivariate_stats'`, `'feature_importance'`, `'multicollinearity'`,
`'feature_interactions'`, `'missingness_plot'`, `'univariate_plot'`,
`'bivariate_plot'`, `'multivariate_plot'`, `'time_series_plot'`.

---

## 8. Complete Example: Titanic Dataset

```python
import pandas as pd
from eda_analyzer import DataAnalyzer

df = pd.read_csv('titanic.csv')

analyzer = DataAnalyzer(
    data=df,
    target_col='Survived',
    exclude_cols=['PassengerId', 'Name', 'Ticket'],
    random_state=42,
    verbose=True,
    show_plots=True,      # set False for headless runs
    download=False        # set True to build an HTML report
)

analyzer.run_all()

print("\n=== PROGRAMMATIC RESULTS ===")
print("\n1. High Missing Columns (>50%):")
print(analyzer.quality_report_['high_missing_columns'])

print("\n2. Feature Importance (Mutual Information):")
print(analyzer.feature_importance_)

print("\n3. Outlier Summary Report:")
print(analyzer.outlier_report_)

print("\n4. Bivariate Statistical Significance:")
print(analyzer.bivariate_stats_)

print("\n5. Variance Inflation Factor (Multicollinearity):")
print(analyzer.vif_report_)
```
