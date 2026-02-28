# SPD Personnel Complaints Analysis Pipeline

This repository contains a reproducible analysis pipeline and interactive dashboard built around the SPD Personnel Complaints dataset (2021 to present). The project transforms raw complaint records into a cleaned, documented dataset and seven core visualizations that help stakeholders explore complaint volume, allegation types, case status (open vs. closed), and resolution times while keeping data limitations front and center.

---

## Project Motivation

Complaint data can be messy and hard to interpret: fields are inconsistently formatted, outcomes are often missing, and charts are easy to misread without proper context. This project addresses that by:

- Standardizing ingestion and cleaning of the SPD Personnel Complaints dataset.  
- Producing repeatable metrics and visualizations that can be regenerated as new data arrives.  
- Explicitly documenting limitations such as missing outcomes, recency bias, and censoring of open cases so users do not over-interpret the results.

---

## Key Features

- **Reproducible pipeline** from raw CSV → cleaned dataset → aggregated outputs for charts.  
- **Interactive dashboard** (Plotly Dash) with seven views:  
  1. Complaints per Year  
  2. Complaints by Allegation Type (Top 15)  
  3. Disposition Outcomes by Year (subset with recorded outcomes)  
  4. Open vs. Closed Complaints by Quarter  
  5. Resolution Time Histogram (closed cases only)  
  6. Resolution Time by Allegation Type (top allegation categories)  
  7. Complaints by Intake Source  
- **Filters and controls** for date range, allegation type, and intake source.  
- **Validation summaries** reporting row counts, missingness in key fields, and obvious anomalies.

---

## Repository Structure

```text
spd-complaints-analysis/
├── data/
│   ├── raw/                      # Original CSVs (e.g., SPD_Personnel_Complaints_2021_to_Present.csv)
│   └── processed/                # Cleaned / derived outputs used by charts
├── src/
│   ├── data_acquisition.py       # Load raw data and basic schema checks
│   ├── data_cleaning.py          # Cleaning, normalization, and feature engineering
│   ├── analysis.py               # Aggregations used by the 7 visualizations
│   ├── visualizations.py         # Reusable plotting helpers
│   └── pipeline.py               # Orchestrated raw → clean → analyzed run
├── dashboard/
│   └── app.py                    # Plotly Dash application
├── tests/
│   ├── test_cleaning.py
│   ├── test_analysis.py
│   └── test_pipeline.py
├── notebooks/                    # Exploratory analysis and ad-hoc checks
├── reports/                      # Written outputs (e.g., architecture review, phase summaries)
├── docs/
│   ├── TECHNICAL.md              # Developer / maintainer guide
│   └── METHODOLOGY.md            # Analysis methodology and caveats
└── README.md                     # You are here
```

---

## Installation

```bash
git clone https://github.com/<your-username>/spd-complaints-analysis.git
cd spd-complaints-analysis

python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Core dependencies include Python 3.9+, Pandas, NumPy, Plotly, Dash, and Pytest.

---

## Usage

### 1) Run the pipeline

Run the end-to-end pipeline to generate a processed dataset under `data/processed/`:

```bash
python -m src.pipeline \
  --input data/raw/SPD_Personnel_Complaints_2021_to_Present.csv \
  --output data/processed/spd_complaints_processed.csv
```

This command loads the raw CSV, parses complaint and closure dates, derives fields such as `Year`, `YearQuarter`, `Is_Open`, and `Resolution_Days`, normalizes allegation and intake-source text, and writes a cleaned dataset for downstream analysis.

### 2) Launch the dashboard

With the processed data in place, start the Plotly Dash application:

```bash
python dashboard/app.py
```

Then open your browser at:

- http://127.0.0.1:8050

Use the controls on the page to apply filters; charts update reactively and expose details through tooltips.

---

## Screenshots

Dashboard and chart screenshots are stored under `docs/images/`:

```md
![Dashboard overview](docs/images/dashboard-overview.png)
![Complaints per year](docs/images/complaints-per-year.png)
![Resolution time histogram](docs/images/resolution-time-hist.png)
```

These images show the main layout and examples of the yearly and resolution-time views.

---

## Data Source and Citations

- Dataset: SPD Personnel Complaints 2021–present, provided in this repository as `data/raw/SPD_Personnel_Complaints_2021_to_Present.csv`.  
- If you replace or refresh the dataset, keep the same column structure so the pipeline and dashboard continue to run without modification.

When citing this work, reference both the original data publisher and this repository’s analysis and pipeline.

---

## Known Limitations

- Outcome variables (sustained, unfounded, unsubstantiated) are missing for many records, so outcome-based charts describe only the subset with recorded outcomes and may not generalize to all complaints.  
- Recent quarters naturally contain more open cases and partial data entry; trends in open-case counts over time are influenced by process lag and reporting practices.  
- Resolution-time metrics exclude currently open complaints, so long-running unresolved cases are not represented in those distributions.  
- Allegation and intake-source labels are cleaned and mapped through a set of rules (e.g., fixing common typos, collapsing near-duplicates); new or rare categories may require periodic updates to these mappings.

---

## Maintainer

Project maintainer: **Ninad Bhikaje**.  
For questions or suggestions, please open a GitHub issue in this repository.
