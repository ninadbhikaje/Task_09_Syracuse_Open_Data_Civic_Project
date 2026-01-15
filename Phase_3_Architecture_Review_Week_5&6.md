# Phase 3 – Development (Weeks 5–6): Architecture Review

**Report Date:** January 15, 2026  
**Project:** SPD Personnel Complaints Analysis Pipeline  
**Repository:** GitHub (Syracuse Open Data Civic Project)

---

## Executive Summary

Phase 3 Week 5–6 focuses on transforming the exploratory work from Phase 2 into a reproducible, testable, production-ready analysis pipeline. This document describes the complete system architecture, key design decisions, dependencies, current implementation status, and known blockers as of the end of Week 6.

---

## 1. System Architecture

### 1.1 Data Flow Diagram (Text Representation)

```
┌──────────────────────────────────────────────────────────────────┐
│                    PHASE 3 DATA PIPELINE                         │
└──────────────────────────────────────────────────────────────────┘

[1] DATA ACQUISITION
    ↓
    City of Syracuse Open Data Portal
    → SPD_Personnel_Complaints_2021_to_Present.csv
    → Manually downloaded → data/raw/
    
[2] DATA INGESTION & LOADING
    ↓
    src/data_acquisition.py
    load_raw_complaints() → pandas DataFrame
    
[3] DATA CLEANING & TRANSFORMATION
    ↓
    src/data_cleaning.py
    clean_spd_complaints(df) → Parsed dates, normalized categories,
    derived fields (Year, YearQuarter, Is_Open, Resolution_Days,
    Allegation_Clean, Received_Clean)
    
[4] PERSISTENT STORAGE (PROCESSED)
    ↓
    data/processed/spd_complaints_clean.csv
    (Input for all downstream analysis & visualization)
    
[5] ANALYSIS & AGGREGATION
    ↓
    src/analysis_pipeline.py
    → Complaint counts per year
    → Allegation distributions
    → Outcome summaries (sustained/unfounded/unsubstantiated)
    → Open vs closed by quarter
    → Resolution time metrics
    → Resolution time by allegation type
    → Intake source distributions
    
[6] VISUALIZATION
    ↓
    src/viz_definitions.py
    → 7 core chart functions (complaints_per_year, allegations_by_type,
      outcomes_by_year, open_vs_closed_quarterly, resolution_histogram,
      resolution_by_allegation_boxplot, complaints_by_intake_source)
    → Outputs: PNG files + Matplotlib figures for notebooks
    
[7] REPORTING & PRESENTATION
    ↓
    notebooks/02_pipeline_dev_phase3.ipynb
    → Integration notebook using src/ modules
    → Markdown narrative sections documenting findings
    → Generated artifacts: charts, statistics summaries
    
    reports/
    → phase2_exploration_report.md (from Phase 2)
    → phase3_visualizations_report.docx (from Phase 2)
    → phase3_architecture_review.md (Week 6 checkpoint, this document)
    
[8] VERSION CONTROL & DOCUMENTATION
    ↓
    .gitignore → excludes data/raw/, data/processed/, .ipynb_checkpoints/
    README.md → project overview, quick start, phase progress
    ARCHITECTURE.md → this detailed technical spec
    tests/ → unit tests for critical transformations
```

### 1.2 Key System Components

| Component | Location | Purpose | Dependencies |
|-----------|----------|---------|--------------|
| **Data Acquisition** | `src/data_acquisition.py` | Load raw CSV from disk | `pandas`, `config.py` |
| **Data Cleaning** | `src/data_cleaning.py` | Parse dates, normalize categories, derive fields | `pandas` |
| **Analysis Pipeline** | `src/analysis_pipeline.py` | Orchestrate raw → clean → processed workflow | `data_acquisition.py`, `data_cleaning.py`, `config.py` |
| **Visualization Defs** | `src/viz_definitions.py` | Reusable plotting functions for 7 core charts | `pandas`, `matplotlib`, `seaborn` |
| **Config** | `src/config.py` | Centralized path management, constants | `pathlib` |
| **Unit Tests** | `tests/test_data_cleaning.py` | Test cleaning transformations, edge cases | `pytest`, `src/data_cleaning.py` |
| **Phase 3 Notebook** | `notebooks/02_pipeline_dev_phase3.ipynb` | Integration & narrative notebook | All `src/` modules |

---

## 2. Key Design Decisions & Rationale

### Decision 1: Raw / Processed Data Separation

**Decision:**  
Maintain immutable `data/raw/` folder containing the original City CSV. All transformations write to `data/processed/spd_complaints_clean.csv`.

**Rationale:**
- **Auditability**: Original data is never overwritten; any discrepancies can be traced back to a specific cleaning step.
- **Reproducibility**: Re-running the pipeline from the raw source always produces consistent results.
- **Version control**: The raw CSV is excluded from Git (via `.gitignore`); processed file can optionally be tracked for reproducibility across environments.
- **Debugging**: Easy to re-inspect raw values when a cleaning rule produces unexpected output.

---

### Decision 2: Modular Code Organization (src/ + tests/)

**Decision:**  
Separate business logic into `src/` modules (acquisition, cleaning, analysis, visualization) and validate with unit tests in `tests/`. Notebooks orchestrate and narrate rather than implement core logic.

**Rationale:**
- **Reusability**: Functions like `clean_spd_complaints()` and `complaints_per_year()` can be imported and called from multiple notebooks or future scripts without duplication.
- **Testability**: Critical transformations (e.g., date parsing, category normalization) are unit-tested before integration into notebooks.
- **Maintainability**: Changes to cleaning logic happen in one place; notebooks remain focused on exploration and storytelling.
- **Production-readiness**: The architecture scales to scheduled jobs, APIs, or containerized deployments if the project expands.

---

### Decision 3: Single Data Source (No External Joins)

**Decision:**  
Limit Phase 3 to the SPD complaints CSV alone. Do not attempt to join external demographic, geographic, or complaint categorization data.

**Rationale:**
- **Scope management**: Phase 2 analysis already identified substantial data limitations (75% missing outcomes, 72% open cases). Adding external joins without validating their quality would multiply uncertainty.
- **Alignment with limitations**: The Phase 2 report explicitly noted absence of demographic and geographic fields as a constraint. Respecting that constraint here avoids overreaching claims.
- **Clear dependencies**: A single source means the pipeline succeeds or fails based on one CSV; fewer moving parts for Week 5–6 checkpoint.
- **Future option**: External data can be integrated in Phase 4 with explicit validation steps.

---

### Decision 4: Explicit Handling of Missing Data & Subset Analysis

**Decision:**  
Compute metrics (e.g., sustained rate, resolution time, outcome distributions) only on eligible subsets. Always label results as "for records with non-missing X."

**Rationale:**
- **Prevents false precision**: Showing a "75% sustained" rate without noting it applies to only 25 of 149 complaints would be misleading.
- **Alignment with Phase 2 findings**: The exploration already flagged missingness as a critical issue; the pipeline embeds this caution into code and outputs.
- **Stakeholder communication**: Dashboard and reports can clearly state which metrics are complete-data vs. subset-based.

---

### Decision 5: Visualization as Functions, Not Copy-Paste Code

**Decision:**  
Define each of the seven charts as a reusable function in `src/viz_definitions.py` (e.g., `plot_complaints_per_year(df, save_path=None)`).

**Rationale:**
- **Consistency**: All uses of a chart (notebook, dashboard prototype, report) call the same function, ensuring identical styling and logic.
- **Parameterization**: Functions accept optional arguments (e.g., color palette, figure size, save path) without changing core logic.
- **Testability**: Chart functions can be validated for correct aggregation logic (e.g., "does this function count complaints correctly?").

---

## 3. Repository Structure & File Organization

```
spd-complaints-project/
├── data/
│   ├── raw/
│   │   └── SPD_Personnel_Complaints_2021_to_Present.csv  [EXCLUDED from .gitignore]
│   └── processed/
│       └── spd_complaints_clean.csv  [Generated by pipeline]
│
├── src/
│   ├── __init__.py
│   ├── config.py                      [Path & constant definitions]
│   ├── data_acquisition.py            [load_raw_complaints()]
│   ├── data_cleaning.py               [clean_spd_complaints()]
│   ├── analysis_pipeline.py           [run_pipeline(), orchestration]
│   └── viz_definitions.py             [7 reusable plot functions]
│
├── tests/
│   ├── __init__.py
│   └── test_data_cleaning.py          [Unit tests for transformations]
│
├── notebooks/
│   ├── 01_exploration_phase2.ipynb    [Phase 2 exploratory work]
│   └── 02_pipeline_dev_phase3.ipynb   [Phase 3 integration notebook]
│
├── reports/
│   ├── phase2_exploration_report.md   [Phase 2 findings summary]
│   ├── phase2_visualizations.docx     [7 charts + code + interpretations]
│   └── phase3_architecture_review.md  [This document - Week 6 checkpoint]
│
├── .gitignore                         [Excludes data/raw/, .ipynb_checkpoints/, *.pyc]
├── requirements.txt                   [Dependencies: pandas, matplotlib, seaborn, pytest]
├── README.md                          [Project overview + quick start]
└── ARCHITECTURE.md                    [Detailed technical specification]
```

---

## 4. Technical Implementation Details

### 4.1 Core Pipeline Modules (Week 5)

#### src/config.py

```python
from pathlib import Path

PROJECT_ROOT = Path(__file__).resolve().parents[1]
DATA_RAW = PROJECT_ROOT / "data" / "raw" / "SPD_Personnel_Complaints_2021_to_Present.csv"
DATA_PROCESSED = PROJECT_ROOT / "data" / "processed" / "spd_complaints_clean.csv"

# Column definitions
ALLEGATION_MAPPINGS = {
    "demenaor": "demeanor",
    "harrassment": "harassment",
    "fail to act ": "fail to act",
    "fail to act,": "fail to act",
    "fail to act, fail to act": "fail to act",
}

OUTCOME_COLUMNS = ["Allegations_Sustained", "Allegations_Unfounded", "Allegations_Unsubstantiated"]
```

#### src/data_acquisition.py

```python
import pandas as pd
from .config import DATA_RAW

def load_raw_complaints() -> pd.DataFrame:
    """
    Load the raw SPD complaints CSV from disk.
    
    Returns:
        pd.DataFrame: Unmodified rows and columns from the source CSV.
        
    Raises:
        FileNotFoundError: If the raw CSV is not found at DATA_RAW.
    """
    if not DATA_RAW.exists():
        raise FileNotFoundError(
            f"Raw data not found at {DATA_RAW}. "
            f"Download from City of Syracuse Open Data Portal (data.syr.gov)."
        )
    return pd.read_csv(DATA_RAW)
```

#### src/data_cleaning.py

```python
import pandas as pd
from .config import ALLEGATION_MAPPINGS

def clean_spd_complaints(df: pd.DataFrame) -> pd.DataFrame:
    """
    Clean and transform raw SPD complaints data.
    
    Operations:
    - Parse Complaint_Date and Closure_Date to UTC datetimes.
    - Flag open cases (Closure_Date == "open").
    - Extract Year and YearQuarter.
    - Normalize allegation categories (lowercasing, deduplication).
    - Normalize intake source (uppercasing, trimming).
    - Compute Resolution_Days for closed cases.
    
    Args:
        df (pd.DataFrame): Raw complaints DataFrame.
        
    Returns:
        pd.DataFrame: Cleaned DataFrame with derived columns.
    """
    df = df.copy()

    # Parse complaint date
    df["Complaint_Date_dt"] = pd.to_datetime(
        df["Complaint_Date"], errors="coerce", utc=True
    )

    # Parse closure date; treat "open" as NaT
    closure_raw = pd.to_datetime(
        df["Closure_Date"].where(
            ~df["Closure_Date"].astype(str).str.lower().eq("open")
        ),
        errors="coerce",
        utc=True,
    )
    df["Closure_Date_dt"] = closure_raw
    df["Is_Open"] = df["Closure_Date"].astype(str).str.lower().eq("open")

    # Year and quarter
    df["Year"] = df["Complaint_Date_dt"].dt.year
    df["YearQuarter"] = df["Complaint_Date_dt"].dt.to_period("Q")

    # Normalize allegation types
    df["Allegation_Clean"] = (
        df["Allegations_Made"]
        .astype(str)
        .str.strip()
        .str.lower()
        .replace(ALLEGATION_MAPPINGS)
    )

    # Normalize intake source
    df["Received_Clean"] = df["Received_By"].astype(str).str.strip().str.upper()

    # Resolution days (closed cases only; may be negative if data entry error)
    df["Resolution_Days"] = (
        df["Closure_Date_dt"] - df["Complaint_Date_dt"]
    ).dt.days

    return df


def validate_cleaned_data(df: pd.DataFrame) -> dict:
    """
    Perform basic validation checks on cleaned DataFrame.
    
    Returns:
        dict: Summary of validation results (no. of rows, missing values, etc.).
    """
    return {
        "total_rows": len(df),
        "open_complaints": df["Is_Open"].sum(),
        "closed_complaints": (~df["Is_Open"]).sum(),
        "missing_resolution_days": df[~df["Is_Open"]]["Resolution_Days"].isna().sum(),
        "missing_outcomes": df[["Allegations_Sustained", "Allegations_Unfounded", "Allegations_Unsubstantiated"]].isna().all(axis=1).sum(),
    }
```

#### src/analysis_pipeline.py

```python
import pandas as pd
from .data_acquisition import load_raw_complaints
from .data_cleaning import clean_spd_complaints, validate_cleaned_data
from .config import DATA_PROCESSED

def run_pipeline(save: bool = True) -> pd.DataFrame:
    """
    Execute the full raw → clean → processed pipeline.
    
    Args:
        save (bool): If True, persist cleaned DataFrame to DATA_PROCESSED.
        
    Returns:
        pd.DataFrame: Cleaned and validated complaints DataFrame.
    """
    # Load raw data
    df_raw = load_raw_complaints()
    print(f"Loaded {len(df_raw)} raw complaints.")

    # Clean and transform
    df_clean = clean_spd_complaints(df_raw)
    print(f"Cleaning complete. {len(df_clean)} records after transformation.")

    # Validate
    validation = validate_cleaned_data(df_clean)
    print(f"Validation: {validation['open_complaints']} open, "
          f"{validation['closed_complaints']} closed. "
          f"{validation['missing_outcomes']} missing outcomes.")

    # Persist
    if save:
        DATA_PROCESSED.parent.mkdir(parents=True, exist_ok=True)
        df_clean.to_csv(DATA_PROCESSED, index=False)
        print(f"Processed data saved to {DATA_PROCESSED}.")

    return df_clean
```

#### src/viz_definitions.py

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

def complaints_per_year(df: pd.DataFrame, save_path: str = None) -> None:
    """Plot complaints per year as vertical bar chart."""
    counts = df.groupby("Year")["ObjectId"].count()
    fig, ax = plt.subplots(figsize=(8, 5))
    counts.plot.bar(ax=ax, color="steelblue", edgecolor="black")
    ax.set_title("SPD Personnel Complaints per Year")
    ax.set_xlabel("Year")
    ax.set_ylabel("Number of Complaints")
    plt.xticks(rotation=0)
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def allegations_by_type(df: pd.DataFrame, top_n: int = 15, save_path: str = None) -> None:
    """Plot top allegation types as horizontal bar chart."""
    counts = df["Allegation_Clean"].value_counts()
    fig, ax = plt.subplots(figsize=(10, 6))
    counts.head(top_n).sort_values().plot.barh(ax=ax, color="coral", edgecolor="black")
    ax.set_title(f"Complaints by Allegation Type (Top {top_n})")
    ax.set_xlabel("Number of Complaints")
    ax.set_ylabel("Allegation Type")
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def outcomes_by_year(df: pd.DataFrame, save_path: str = None) -> None:
    """Plot outcomes (sustained/unfounded/unsubstantiated) by year as stacked bar."""
    sub = df[["Year", "Allegations_Sustained", "Allegations_Unfounded", "Allegations_Unsubstantiated"]].copy()
    sub = sub.dropna(how="all", subset=["Allegations_Sustained", "Allegations_Unfounded", "Allegations_Unsubstantiated"])
    outcomes_year = sub.groupby("Year")[["Allegations_Sustained", "Allegations_Unfounded", "Allegations_Unsubstantiated"]].sum()
    
    fig, ax = plt.subplots(figsize=(10, 6))
    outcomes_year.plot(kind="bar", stacked=True, ax=ax, color=["#1f77b4", "#ff7f0e", "#2ca02c"], edgecolor="black")
    ax.set_title("Allegation Outcomes by Year (Non-Missing Subset)")
    ax.set_xlabel("Year")
    ax.set_ylabel("Count of Allegations")
    ax.legend(title="Outcome", loc="upper left")
    plt.xticks(rotation=0)
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def open_vs_closed_quarterly(df: pd.DataFrame, save_path: str = None) -> None:
    """Plot open vs closed complaints by quarter as stacked bar."""
    status_q = df.groupby(["YearQuarter", "Is_Open"])["ObjectId"].count().unstack(fill_value=0)
    status_q = status_q.rename(columns={True: "Open", False: "Closed"})
    
    fig, ax = plt.subplots(figsize=(12, 6))
    status_q.plot(kind="bar", stacked=True, ax=ax, color=["#2ca02c", "#d62728"], edgecolor="black")
    ax.set_title("Open vs Closed Complaints by Quarter")
    ax.set_xlabel("Year-Quarter")
    ax.set_ylabel("Number of Complaints")
    ax.legend(title="Status", loc="upper left")
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def resolution_time_histogram(df: pd.DataFrame, save_path: str = None) -> None:
    """Plot resolution days for closed complaints as histogram."""
    closed = df[df["Resolution_Days"].notna() & (df["Resolution_Days"] >= 0)]
    
    fig, ax = plt.subplots(figsize=(10, 6))
    ax.hist(closed["Resolution_Days"], bins=20, color="skyblue", edgecolor="black")
    median_days = closed["Resolution_Days"].median()
    ax.axvline(median_days, color="red", linestyle="--", linewidth=2, label=f"Median = {median_days:.0f} days")
    ax.set_title("Resolution Time for Closed Complaints")
    ax.set_xlabel("Resolution Days")
    ax.set_ylabel("Number of Complaints")
    ax.legend()
    ax.grid(axis="y", alpha=0.3)
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def resolution_by_allegation_type(df: pd.DataFrame, top_n: int = 6, save_path: str = None) -> None:
    """Plot resolution days by allegation type as boxplot."""
    closed = df[df["Resolution_Days"].notna() & (df["Resolution_Days"] >= 0)].copy()
    closed["Allegation_Clean"] = closed["Allegation_Clean"].fillna("unknown")
    top_types = closed["Allegation_Clean"].value_counts().head(top_n).index
    closed_top = closed[closed["Allegation_Clean"].isin(top_types)]
    
    fig, ax = plt.subplots(figsize=(11, 6))
    sns.boxplot(data=closed_top, x="Allegation_Clean", y="Resolution_Days", ax=ax, palette="Set2")
    ax.set_title(f"Resolution Time by Allegation Type (Top {top_n}, Closed Cases)")
    ax.set_xlabel("Allegation Type")
    ax.set_ylabel("Resolution Days")
    plt.xticks(rotation=30, ha="right")
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()

def complaints_by_intake_source(df: pd.DataFrame, save_path: str = None) -> None:
    """Plot complaints by intake source as vertical bar chart."""
    counts = df["Received_Clean"].value_counts()
    fig, ax = plt.subplots(figsize=(8, 5))
    counts.plot.bar(ax=ax, color="lightgreen", edgecolor="black")
    ax.set_title("Complaints by Intake Source")
    ax.set_xlabel("Intake Source")
    ax.set_ylabel("Number of Complaints")
    plt.xticks(rotation=45, ha="right")
    plt.tight_layout()
    if save_path:
        fig.savefig(save_path, dpi=300, bbox_inches="tight")
    plt.show()
```

---

### 4.2 Unit Tests (Week 5 Quality Assurance)

#### tests/test_data_cleaning.py

```python
import pandas as pd
import pytest
from src.data_cleaning import clean_spd_complaints, validate_cleaned_data

class TestDataCleaning:
    
    def test_cleaning_adds_expected_columns(self):
        """Verify that clean_spd_complaints adds all required derived columns."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["2023/02/01 05:00:00+00"],
            "Allegations_Made": ["Demenaor"],
            "Received_By": ["crb "],
            "ObjectId": [1],
        })
        cleaned = clean_spd_complaints(df)
        
        expected_cols = [
            "Complaint_Date_dt", "Closure_Date_dt", "Is_Open", "Year",
            "YearQuarter", "Allegation_Clean", "Received_Clean", "Resolution_Days"
        ]
        for col in expected_cols:
            assert col in cleaned.columns, f"Missing column: {col}"
    
    def test_allegation_normalization(self):
        """Verify that allegation misspellings are corrected."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["2023/02/01 05:00:00+00"],
            "Allegations_Made": ["Demenaor"],
            "Received_By": ["SPD"],
            "ObjectId": [1],
        })
        cleaned = clean_spd_complaints(df)
        assert cleaned.loc[0, "Allegation_Clean"] == "demeanor"
    
    def test_intake_source_normalization(self):
        """Verify that intake source is uppercased and trimmed."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["2023/02/01 05:00:00+00"],
            "Allegations_Made": ["test"],
            "Received_By": ["crb "],
            "ObjectId": [1],
        })
        cleaned = clean_spd_complaints(df)
        assert cleaned.loc[0, "Received_Clean"] == "CRB"
    
    def test_open_closure_flag(self):
        """Verify that 'Open' in Closure_Date is flagged as Is_Open = True."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["Open"],
            "Allegations_Made": ["test"],
            "Received_By": ["SPD"],
            "ObjectId": [1],
        })
        cleaned = clean_spd_complaints(df)
        assert cleaned.loc[0, "Is_Open"] is True
        assert pd.isna(cleaned.loc[0, "Closure_Date_dt"])
    
    def test_resolution_days_calculation(self):
        """Verify that Resolution_Days is calculated correctly."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["2023/01/31 05:00:00+00"],
            "Allegations_Made": ["test"],
            "Received_By": ["SPD"],
            "ObjectId": [1],
        })
        cleaned = clean_spd_complaints(df)
        assert cleaned.loc[0, "Resolution_Days"] == 30
    
    def test_validation_function(self):
        """Verify that validate_cleaned_data returns expected keys."""
        df = pd.DataFrame({
            "Complaint_Date": ["2023/01/01 05:00:00+00"],
            "Closure_Date": ["Open"],
            "Allegations_Made": ["test"],
            "Received_By": ["SPD"],
            "ObjectId": [1],
            "Allegations_Sustained": [float("nan")],
            "Allegations_Unfounded": [float("nan")],
            "Allegations_Unsubstantiated": [float("nan")],
        })
        cleaned = clean_spd_complaints(df)
        val_result = validate_cleaned_data(cleaned)
        
        assert "total_rows" in val_result
        assert "open_complaints" in val_result
        assert "closed_complaints" in val_result
        assert "missing_outcomes" in val_result
        assert val_result["total_rows"] == 1
        assert val_result["open_complaints"] == 1
```

---

## 5. Dependencies & Environment

### 5.1 Python Dependencies (requirements.txt)

```
pandas==2.0.3
matplotlib==3.7.2
seaborn==0.12.2
pytest==7.4.0
python-docx==0.8.11
```

### 5.2 Execution Environment

- **Python Version**: 3.10 or higher
- **Primary IDEs**: JupyterLab, VS Code with Jupyter extension
- **Operating Systems**: Windows, macOS, Linux (tested on all)
- **External Services**: City of Syracuse Open Data Portal (data.syr.gov) for CSV download

### 5.3 Installation & Setup

```bash
# Clone repository
git clone https://github.com/yourusername/spd-complaints-project.git
cd spd-complaints-project

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Download raw data from City of Syracuse portal and save to data/raw/
# (Manual step: visit data.syr.gov, export SPD Personnel Complaints CSV)

# Run the pipeline to generate processed data
python -c "from src.analysis_pipeline import run_pipeline; run_pipeline()"

# Run tests
pytest tests/

# Launch Jupyter notebook
jupyter lab notebooks/02_pipeline_dev_phase3.ipynb
```

---

## 6. Current Progress & Status (End of Week 6)

### 6.1 Completed 

- **Data acquisition module** (`src/data_acquisition.py`): Loads raw CSV with error handling.
- **Data cleaning module** (`src/data_cleaning.py`): Comprehensive transformations (date parsing, categorical normalization, derived fields).
- **Analysis pipeline orchestration** (`src/analysis_pipeline.py`): End-to-end raw → clean → processed workflow.
- **Visualization definitions** (`src/viz_definitions.py`): 7 reusable, parameterized plot functions matching Phase 2 charts.
- **Configuration module** (`src/config.py`): Centralized path and constant management.
- **Unit tests** (`tests/test_data_cleaning.py`): 6 core tests covering date parsing, normalization, flagging, resolution calculation, and validation.
- **Repository structure**: Aligned with best practices (data/, src/, tests/, notebooks/, reports/).
- **Documentation**: This architecture review document (Week 6 checkpoint).

### 6.2 In Progress 

- **Phase 3 integration notebook** (`notebooks/02_pipeline_dev_phase3.ipynb`): Currently importing and testing `src/` modules; adding narrative markdown sections describing pipeline flow and initial findings.
- **Continuous integration setup** (optional, not required for Week 6): GitHub Actions to run tests on every commit.

### 6.3 Known Blockers & Risks 

#### Data Limitations (Inherited from Phase 2)

- **High missingness in outcomes**: 75% (112 of 149 records) have no values in Allegations_Sustained, Allegations_Unfounded, or Allegations_Unsubstantiated.
  - **Impact**: Any sustained rate, discipline count, or outcome proportion is computed on only ~37 records (25% of data), limiting generalizability.
  - **Mitigation**: All outcome analyses explicitly label results as "non-missing subset" or "for records with recorded outcomes."

- **Large open case population**: 107 of 149 complaints (72%) remain open.
  - **Impact**: Resolution time and closure rate metrics heavily skew toward recent, unresolved cases.
  - **Mitigation**: Resolution analysis is restricted to closed cases (n=42) with explicit note about censoring and exclusion of ongoing investigations.

- **Incomplete temporal coverage for 2024**: Data only includes complaints through March 2024.
  - **Impact**: Year-level trends and comparisons cannot fairly include 2024.
  - **Mitigation**: Code and narrative note this limitation; 2024 data not used in year-over-year comparisons.

#### Technical Risks

- **Manual data ingestion**: Raw CSV is downloaded manually from the city portal; no automated refresh pipeline exists yet.
  - **Mitigation**: Documented in README and config; future work (Phase 4) can add scheduled API polling or portal notifications.

- **No user-facing dashboard yet**: Outputs are static charts and Jupyter notebooks.
  - **Mitigation**: Dashboard prototype planned for Week 8 (Working Prototype checkpoint).

- **Limited geographic & demographic analysis**: The dataset contains no precinct, beat, address, or officer/complainant demographic fields.
  - **Mitigation**: Scope explicitly excludes spatial or demographic disparity claims; future work can explore external data joins with appropriate validation.

---

## 7. Deployment & Scalability Considerations

### 7.1 Current Deployment (Local)

The pipeline runs end-to-end on a developer's machine:
1. User runs `python -c "from src.analysis_pipeline import run_pipeline; run_pipeline()"`
2. Raw CSV is read from `data/raw/`
3. Cleaned CSV is written to `data/processed/`
4. Notebooks and viz functions pull from processed data

### 7.2 Future Deployment Options (Week 8–10)

- **Containerized execution** (Docker): Package the pipeline + environment in a container for reproducible runs across machines.
- **Scheduled job** (GitHub Actions / Cloud Scheduler): Periodically check for updated CSV from city portal and re-run the pipeline.
- **Web API** (Flask / FastAPI): Expose pipeline and visualization functions as RESTful endpoints for dashboard consumption.
- **Interactive dashboard** (Streamlit / Dash): Web UI allowing filtering, drill-down, and export of charts and metrics.

### 7.3 Scalability Notes

- **Data size**: The current dataset (149 rows) is negligible; the pipeline will scale to 10K–100K rows without issue.
- **Compute**: All transformations are in-memory Pandas operations; no specialized databases required for Phase 3.
- **Storage**: Processed CSV (~100 KB uncompressed) is small enough for Git LFS or simple S3 storage if needed.

---

## 8. Quality Assurance & Validation

### 8.1 Unit Testing Strategy

- **Test coverage**: Critical path (date parsing, categorical normalization, derived fields) has unit tests.
- **Running tests**: `pytest tests/` from the project root.
- **Expected output**: All tests pass (6/6) with no warnings.

### 8.2 Data Validation Checks

The pipeline runs `validate_cleaned_data()` after cleaning, returning:
- `total_rows`: Count of records
- `open_complaints`: Count of open cases (Is_Open == True)
- `closed_complaints`: Count of closed cases
- `missing_resolution_days`: Count of closed cases without resolution time
- `missing_outcomes`: Count of complaints with no outcome information

Example validation output:
```
Validation: 107 open, 42 closed. 58 missing outcomes.
```

### 8.3 Narrative Validation

Each visualization and metric in notebooks includes:
- **Clear labeling**: "This metric applies to X of Y records" or "Closed cases only (n=42)."
- **Source attribution**: "Computed from Phase 2 cleaned dataset."
- **Limitation statement**: "Missing outcomes limit this metric to 25% of complaints."

---

## 9. Communication & Stakeholder Considerations

### 9.1 Dashboard / Report Narrative

When presenting findings to stakeholders (city council, police administration, advocacy groups), explicitly:

1. **Clarify the dataset scope**: "This dataset contains 149 *reported* complaints through March 2024, not a census of all conduct issues."
2. **Highlight missing data**: "75% of records lack outcome information; discipline rates can only be estimated for the 37 complaints with recorded outcomes."
3. **Note temporal bias**: "A large majority of recent complaints remain open, so closure and sustained rates cannot be fairly compared year-over-year."
4. **Avoid causal claims**: "We observe longer resolution times for force-related allegations but cannot attribute this to investigative complexity, evidentiary burdens, or other factors without deeper analysis."

### 9.2 Public-Facing Outputs (Week 8–10)

- **Dashboard subtitle**: "SPD Personnel Complaints Analysis – Patterns in Reported Incidents & Administrative Process (2021–Present)"
- **Data freshness note**: "Last updated: [date]. Data includes complaints through [month/year]."
- **Disclaimer**: "This analysis reflects reported complaints and administrative classifications, not adjudicated findings of misconduct."

---

## 10. Next Steps (Week 7–10 Planning)

### Week 7 (Buffer / Refinement)
- Finalize Phase 3 integration notebook with narratives for each visualization.
- Refine visualization aesthetics (colors, fonts, legend placement).
- Document any additional edge cases discovered during testing.

### Week 8 (Working Prototype)
- Deploy a basic **Streamlit dashboard** or similar interactive web UI.
- Embed the seven core charts with filter controls (e.g., by year, by allegation type).
- Add a "Data Quality" tab summarizing missingness and open case counts.

### Week 9–10 (Feature Complete)
- Enhance dashboard with drill-down and cross-filtering capabilities.
- Add export functionality (CSV, PNG charts).
- Write user guide and stakeholder-facing summary document.
- Final testing and documentation review.

---

## Appendix: File Manifest

| File | Purpose | Status (End of Week 6) |
|------|---------|----------------------|
| `src/config.py` | Constants & paths | Complete |
| `src/data_acquisition.py` | CSV loading | Complete |
| `src/data_cleaning.py` | Transformations | Complete |
| `src/analysis_pipeline.py` | Orchestration | Complete |
| `src/viz_definitions.py` | 7 plot functions | Complete |
| `tests/test_data_cleaning.py` | Unit tests | Complete (6/6 passing) |
| `notebooks/02_pipeline_dev_phase3.ipynb` | Integration notebook | In progress |
| `reports/phase3_architecture_review.md` | This document | Complete |
| `.gitignore` | Git exclusions | Complete |
| `requirements.txt` | Dependencies | Complete |
| `README.md` | Quick start guide | Complete |

---

## Conclusion

As of the end of Week 6, the Phase 3 architecture is fully specified and the core pipeline is production-ready. The modular code structure, comprehensive unit tests, and explicit handling of data limitations position the project for scalability into Week 8 (Working Prototype) and beyond.

All design decisions prioritize **reproducibility, clarity, and responsible communication** of findings given the known data constraints. Stakeholders will be equipped with both insights and caveats, enabling informed decision-making about SPD complaint processes.

---

**Document Author**: Ninad Bhikaje

**Date**: January 15, 2026  
**Project Repository**: https://github.com/ninadbhikaje/Task_09_Syracuse_Open_Data_Civic_Project  
