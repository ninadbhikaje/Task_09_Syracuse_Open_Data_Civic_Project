# Technical Documentation — SPD Personnel Complaints Analysis Pipeline

## 1. Architecture Overview

The system is organized as a reproducible pipeline with a separate presentation layer (dashboard).

1. **Data Acquisition (`src/data_acquisition.py`)**  
   - Loads raw complaint CSVs from `data/raw/`.  
   - Performs basic schema checks (required columns, non-empty files).  

2. **Cleaning & Feature Engineering (`src/data_cleaning.py`)**  
   - Parses `Complaint_Date` and `Closure_Date` with timezone awareness.  
   - Derives:  
     - `Year` (calendar year of complaint).  
     - `YearQuarter` (quarterly period of complaint).  
     - `Is_Open` (flag based on closure status or `"Open"` text).  
     - `Resolution_Days` (closure minus complaint date for closed cases, excluding negative values).  
   - Normalizes categorical text such as allegation type and intake source (trimming, case normalization, fixing common typos, and collapsing near-duplicates).  

3. **Analysis & Aggregations (`src/analysis.py`)**  
   - Builds the grouped tables used by the seven core charts:  
     - Annual complaint counts.  
     - Top-15 allegation types by frequency.  
     - Yearly counts of sustained, unfounded, and unsubstantiated allegations for the recorded-outcome subset.  
     - Open vs. closed counts by quarter.  
     - Resolution-time distributions and medians for closed cases.  
     - Resolution-time statistics by allegation type.  
     - Complaint counts by intake source.  

4. **Visualization Helpers (`src/visualizations.py`)**  
   - Provides reusable plotting functions that accept aggregated data and produce consistent chart objects or figure JSON.  

5. **Pipeline Orchestration (`src/pipeline.py`)**  
   - Coordinates the full run from raw input path to processed dataset and precomputed aggregates.  

6. **Dashboard (`dashboard/app.py`)**  
   - Exposes the seven visualizations via Plotly Dash with filters, tooltips, and explanatory text.  

---

## 2. Repository Layout

```text
spd-complaints-analysis/
├── data/
│   ├── raw/                      # Original CSVs
│   └── processed/                # Cleaned / derived outputs
├── src/
│   ├── data_acquisition.py
│   ├── data_cleaning.py
│   ├── analysis.py
│   ├── visualizations.py
│   └── pipeline.py
├── dashboard/
│   └── app.py
├── tests/
│   ├── test_cleaning.py
│   ├── test_analysis.py
│   └── test_pipeline.py
├── notebooks/
├── reports/
├── docs/
│   ├── TECHNICAL.md
│   └── METHODOLOGY.md
└── README.md
```

This reflects the structure established during Phases 3 and 4, with notebooks and reports preserved for exploratory and narrative work and `docs/` used for long-form documentation.

---

## 3. Development Setup

1. Clone the repository and create a virtual environment:

```bash
git clone https://github.com/<your-username>/spd-complaints-analysis.git
cd spd-complaints-analysis

python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

2. Optional: install development extras (linting, formatting) if defined in `requirements-dev.txt`.

Core technologies: Python 3.9+, Pandas, NumPy, Plotly, Dash, and Pytest.

---

## 4. Testing Instructions

Run the full test suite with:

```bash
pytest -q
```

Recommended coverage:

- **Cleaning logic:**  
  - Correct parsing of valid dates and handling of malformed entries (coercion to missing).  
  - Correct derivation of `Year`, `YearQuarter`, `Is_Open`, and `Resolution_Days`.  

- **Categorical normalization:**  
  - Allegation type mapping (typo correction, trimming, case normalization).  
  - Intake source normalization (CRB / SPD / AG / Other).  

- **Aggregations:**  
  - Counts and top-N logic used for allegation and intake charts.  
  - Correct stacking of sustained / unfounded / unsubstantiated counts by year.  
  - Correct open vs. closed counts by quarter.  
  - Resolution-time distributions and medians consistent with raw closed-case data.  

- **Pipeline:**  
  - End-to-end run from raw CSV to processed file, including error handling when required columns are missing or all rows are filtered out.

---

## 5. Deployment Guide

### Local Development

1. Ensure processed data exists (run `src.pipeline`).  
2. Start the Dash server:

```bash
python dashboard/app.py
```

3. Visit `http://127.0.0.1:8050` in your browser.

### Production / Cloud (High-Level)

- Use a production server wrapper (e.g., `gunicorn` with `dash` or a WSGI/ASGI bridge).  
- Configure environment variables for data paths and any secrets.  
- Schedule a recurring job (cron, Airflow, etc.) to rerun the pipeline as new data is made available.  
- Use centralized logging to capture validation summaries and error messages.

---

## 6. Error Handling and Logging

- Validate required columns at load time; raise a descriptive exception listing missing fields.  
- When filters yield no rows, return empty frames with clear messages to the dashboard instead of crashing.  
- Log validation outputs (row counts before/after cleaning, missingness in key fields, anomalous dates) for each run so data quality issues can be monitored over time.  
- Prefer structured log messages that identify which stage of the pipeline encountered issues.
