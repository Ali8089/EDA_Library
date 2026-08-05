# DataAnalyzer: Documentation

The **DataAnalyzer** library provides an automated, end-to-end Exploratory Data Analysis (EDA) pipeline. It is designed to ingest raw datasets, standardize and map variables into semantic roles, compute statistical hypothesis tests, calculate non-linear feature associations, dynamically select optimal charts, and compile self-contained reports in HTML or PDF format.

---

## 1. Prerequisites and Installation

### Core Dependencies

The library relies on standard data science and machine learning packages. Ensure the following dependencies are installed in your Python environment:

```bash
pip install pandas numpy seaborn matplotlib scipy scikit-learn

```

### Optional Dependencies

To enable PDF report generation alongside the default HTML format, install the `weasyprint` rendering library:

```bash
pip install weasyprint

```

---

## 2. Supported Data Structures and Semantic Type Mapping

### Supported Input Data Structures

The library standardizes incoming data into a Pandas DataFrame via an internal `_standardize_data()` wrapper. The following data formats are accepted:

* **Pandas DataFrame (`pd.DataFrame`)**: Processed directly via explicit deep copy.
* **Pandas Series (`pd.Series`)**: Automatically converted into a single-column DataFrame.
* **Polars DataFrame (`pl.DataFrame`)**: Automatically converted via `.to_pandas()`.
* **NumPy Array (`np.ndarray`)**: 1D arrays are converted to a single column named `feature_0`. 2D arrays are converted to columns named `feature_0`, `feature_1`, ..., `feature_n`.
* **Dictionaries and Lists (`dict`, `list`)**: Ingested directly using standard `pd.DataFrame(data)` constructors.

### Internal Semantic Type Detection Engine

Rather than relying strictly on native Python/Pandas dtypes, `DataAnalyzer` dynamically categorizes features into four semantic types via `_detect_semantic_types()`:

| Semantic Type | Detection Criteria / Rules | Internal Usage |
| --- | --- | --- |
| **`datetime`** | Columns with `datetime64` data types or parsable time strings. | Triggers monthly trend plots in `plot_time_series()`. |
| **`continuous`** | Numeric dtypes (`int`, `float`) where unique values $\ge 15$ AND the ratio of unique values to non-null entries $\ge 0.05$. | Used for Pearson correlation, Z-Score/IQR outlier analysis, ANOVA, VIF, and regression plots. |
| **`categorical`** | 1. Numeric dtypes where unique values $< 15$ OR unique ratio $< 0.05$.<br>

<br>2. Non-numeric dtypes (`object`, `string`, `category`) where unique values $< 20$ OR unique ratio $< 0.5$. | Used for Cramér's $V$, Chi-Square tests, Correlation Ratio ($\eta$), violin plots, and stacked bar charts. |
| **`text`** | 1. Non-numeric dtypes with high unique ratios ($\ge 0.5$) or cardinality $\ge 20$.<br>

<br>2. Columns containing $100\%$ null values. | Bypassed during statistical plot generation to prevent rendering errors. |

---

## 3. Configuration Parameters (Constructor Reference)

The behavior of `DataAnalyzer` is customized during instantiation via its constructor parameters:

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
| --- | --- | --- | --- |
| **`data`** | `Any` | *Required* | The dataset to be analyzed (DataFrame, Series, Array, Dict, List). |
| **`target_col`** | `str` | `None` | The target variable for predictive analysis. Unlocks target-centric statistical tests and charts. |
| **`exclude_cols`** | `List[str]` | `None` | A list of column names to be entirely excluded from analysis (e.g., `['user_id', 'transaction_id']`). |
| **`random_state`** | `int` | `42` | Seed value used for randomized sampling (e.g., Shapiro-Wilk tests) to ensure reproducible results. |
| **`verbose`** | `bool` | `True` | If `True`, logs standard output, progress updates, and text-based tables to the console. |
| **`show_plots`** | `bool` | `True` | If `True`, renders and displays visualizations inline. |
| **`save_dir`** | `str` | `None` | Directory path where generated image assets and reports will be saved. |
| **`download`** | `bool` | `False` | Enables headless execution mode (suppressing inline rendering), saving all outputs directly into an HTML report. |
| **`export_pdf`** | `bool` | `False` | If `True` (and `download=True`), generates a PDF version of the report alongside the HTML file using `weasyprint`. |

---

## 4. Operational Rules and Syntax Restrictions

To maintain computational efficiency and prevent runtime failures, `DataAnalyzer` enforces the following internal constraints:

### Target Column Constraints

* The `target_col` string must match an existing column name in the dataset.
* If `target_col` does not exist, the library logs a warning and falls back to target-agnostic EDA mode.
* Target-centric methods (`run_bivariate_stats`, `compute_feature_importance`, `plot_bivariate`, `plot_time_series`) return safely without raising exceptions if no target column is specified.

### Cardinality Thresholds

* **`MAX_CAT_CARDINALITY = 10`**: Categorical variables with more than 10 unique levels are flagged in `data_quality_report()` and excluded from bivariate plotting, Cramér's V matrices, and group tests to avoid overcrowded visualizations.

### Sample Size and Data Completeness Guards

* **Correlation Ratio ($\eta$) Guard**: Evaluates continuous vs. categorical pairings only if the sub-dataframe contains at least 20 complete cases ($n \ge 20$) across 2 or more unique categories. Small sample sizes return `np.nan`.
* **Normality Check Guard**: Shapiro-Wilk tests for outlier detection sample a maximum of 5,000 rows (using `random_state`) for continuous columns exceeding 5,000 non-null values.
* **High-Missing Exclusion**: Features with $>50\%$ missing values identified during `data_quality_report()` are automatically excluded from `compute_feature_importance()` calculations to prevent excessive row drops during complete-case drops (`dropna()`).
* **Multicollinearity (VIF) Limit**: Computing VIF requires a minimum of 2 continuous features and $n \ge (\text{number of continuous features} + 2)$ complete rows.

### Filename and Character Sanitization

* All plot export filenames undergo automatic Regex sanitization via `_sanitize_filename()`. Any non-alphanumeric character (spaces, parentheses, slashes, punctuation) is replaced with an underscore (`_`) to prevent path collisions and OS-level write errors.

---

## 5. Core Methods Reference

The pipeline methods can be executed all at once using `run_all()`, or called individually for modular analysis.

### Summarization & Data Quality

* **`summarize()`**: Outputs a high-level dataset overview, including row counts, column counts, memory usage, and global missing data percentages.
* **`data_quality_report()`**: Scans for structural anomalies. Flags duplicate records, constant columns (zero variance), features with $>50\%$ missing values, high-cardinality categorical variables ($>10$), and highly skewed continuous distributions ($\vert{}\text{skew}\vert{} > 1$).
* **`plot_missingness()`**: Generates a bar chart illustrating the percentage of missing values per feature, alongside a matrix heatmap detailing the exact location of null values across the dataset.

### Statistical Analysis

* **`detect_outliers()`**: Evaluates continuous variables for extreme values. Dynamically selects the statistical approach based on distribution: Z-Scores ($\vert{}Z\vert{} > 3$) for normally distributed data (verified via Shapiro-Wilk) and the Interquartile Range method ($\text{IQR} \times 1.5$) for skewed data.
* **`run_bivariate_stats()`**: *(Requires `target_col`)* Tests every feature against the target variable for statistical significance. Automatically applies Pearson Correlation ($r$), ANOVA ($F$), or Chi-Square ($\chi^2$) tests based on the data type pairing.
* **`compute_feature_importance()`**: *(Requires `target_col`)* Ranks the predictive utility of all features against the target using Mutual Information (compatible with both classification and regression tasks).
* **`detect_multicollinearity()`**: Computes the Variance Inflation Factor (VIF) for continuous features via Ordinary Least Squares (OLS) $R^2$. Variables with a VIF exceeding 5.0 are flagged as exhibiting high multicollinearity.

### Intelligent Visualization Engine

* **`plot_bivariate()`**: *(Requires `target_col`)* Acts as a routing engine that analyzes data types and volume to select the optimal visualization for feature-target relationships:
* *Continuous vs. Continuous (Large N):* Hexbin density plot.
* *Continuous vs. Continuous (Small N):* Scatterplot with linear regression overlay.
* *Continuous vs. Categorical:* Violin plots or KDE density distributions.
* *Categorical vs. Categorical:* 100% Stacked Bar charts.


* **`plot_multivariate()`**: Computes and plots two distinct association matrices: a Pearson correlation heatmap for continuous variables, and a Cramér's V heatmap for categorical interactions.
* **`plot_time_series()`**: If a datetime variable is detected, this method automatically aggregates and plots the monthly trend of the specified `target_col`.

### Advanced Interactions

* **`analyze_feature_interactions(top_n=5, threshold=0.0)`**: Analyzes interactions exclusively between independent features (excluding the target). Evaluates all possible feature pairings using Pearson $\vert{}r\vert{}$, Cramér's $V$, and Correlation Ratio ($\eta$), plotting detailed visualizations for the strongest `top_n` associations.

---

## 6. Accessing Programmatic Results

The library retains all computed metrics as class attributes, allowing for seamless integration into downstream scripts. By convention, populated result attributes end with an underscore (`_`).

```python
# Execute the full analysis pipeline
analyzer.run_all()

# Retrieve the DataFrame containing outlier detection metrics
outliers_df = analyzer.outlier_report_

# Retrieve a dictionary of data quality flags
quality_metrics = analyzer.quality_report_
print(f"Duplicate Rows Count: {quality_metrics['duplicate_rows']}")

# Retrieve the DataFrame containing bivariate statistical test results
stats_df = analyzer.bivariate_stats_

# Retrieve the DataFrame ranking feature importance
importance_df = analyzer.feature_importance_

# Retrieve the Variance Inflation Factor report
vif_df = analyzer.vif_report_

# Retrieve top feature-feature interactions
interactions_df = analyzer.top_feature_pairs_

```

---

## 7. Generating Downloadable Reports

For automated reporting, `DataAnalyzer` bundles all analyses, tables, and visualizations into a single, self-contained HTML file. Images are embedded using base64 encoding, ensuring the HTML file requires no external dependencies or image folders to render properly.

### Headless Reporting Example

```python
import pandas as pd

df = pd.read_csv('housing_data.csv')

# Initialize with download=True to activate headless reporting mode
analyzer = DataAnalyzer(
    data=df,
    target_col='price',
    download=True,
    export_pdf=True,         # Also exports a PDF report via WeasyPrint
    save_dir='monthly_reports'
)

# Execute the pipeline silently without popping up charts
analyzer.run_all()

# Retrieve the absolute file path of the generated report
print(f"Report generated at: {analyzer.report_path_}")

```

### Custom Pipeline Execution

To execute specific analytical steps or generate a lightweight report, pass a list of step identifiers to `run_all()`:

```python
analyzer.run_all(steps=['summarize', 'data_quality', 'missingness_plot', 'feature_importance'])

```

**Available Step Identifiers:**
`'summarize'`, `'data_quality'`, `'outliers'`, `'bivariate_stats'`, `'feature_importance'`, `'multicollinearity'`, `'feature_interactions'`, `'missingness_plot'`, `'bivariate_plot'`, `'multivariate_plot'`, `'time_series_plot'`.

---

## 8. Complete Implementation Example: Titanic Dataset

The following complete script demonstrates loading the Titanic dataset, excluding non-predictive identifiers, setting `Survived` as the target column, running the automated pipeline, and accessing the resulting metrics programmatically.

```python
import pandas as pd

# 1. Load Titanic dataset
df = pd.read_csv('/content/titanic.csv')

# 2. Instantiate DataAnalyzer
analyzer = DataAnalyzer(
    data=df,
    target_col='Survived',
    exclude_cols=['PassengerId', 'Name', 'Ticket'],
    random_state=42,
    verbose=True,
    show_plots=True,      # Set to False if running headless
    download=False        # Set to True to build HTML report
)

# 3. Run all analytical steps
analyzer.run_all()

# 4. Access programmatic outputs
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
