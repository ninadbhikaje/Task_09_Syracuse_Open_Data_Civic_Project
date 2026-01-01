# Phase 2 Exploration Report
## Research Task 09: Syracuse Open Data Civic Project
### Dataset: SPD Personnel Complaints (2021–Present)

---

## 1. Data Acquisition & Validation

### 1.1 Dataset Acquisition
- **Source:** City of Syracuse Open Data Portal (data.syr.gov), exported as `SPD_Personnel_Complaints_2021_to_Present.csv`
- **Format:** CSV, 149 rows and 15 columns in the attached extract
- **Acquisition Date:** To be recorded in the repo as the date the CSV was downloaded (Week 3)
- **API Usage:** None for this phase; static CSV download
- **Storage Structure (planned):**

```
data/
├── raw/
│   └── SPD_Personnel_Complaints_2021_to_Present.csv
├── processed/
│   └── spd_complaints_clean.csv
```

Raw data will remain unchanged to preserve **auditability** and reproducibility.

### 1.2 Data Dictionary (Refined from actual columns)

| Column Name | Description |
|---|---|
| Complaint_Date | Date the complaint was received (string with timezone-style suffix) |
| Closure_Date | Date the complaint was closed, or "Open" for ongoing cases |
| Received_By | Intake source (e.g., SPD, CRB, AG) |
| Complaints_Num | Number of complaints in this record (always 1 in this file) |
| Officers_Inolved_Num | Number of officers involved (numeric or text like "numerous", "unk") |
| Allegations_Made_Num | Number of allegations in the complaint (numeric or text like "numerous") |
| Allegations_Made | Primary allegation category (e.g., Excessive Force, False Arrest) |
| Allegations_Sustained | Count of sustained allegations (float; often missing) |
| Num_Officers_Sustained | Number of officers with sustained findings |
| Allegations_Unfounded | Number of allegations classified as unfounded |
| Allegations_Unsubstantiated | Number of allegations classified as unsubstantiated |
| Num_Officers_DA | Number of officers receiving disciplinary action |
| Disciplinary_Actions | Text description of formal discipline (e.g., "Written Reprimand") |
| NonDisciplinary_Actions | Text description of non-disciplinary actions (e.g., "Verbal Counseling") |
| ObjectId | Unique integer identifier for each record |

**Note:** No demographic or geographic fields are present, so any location-based or demographic analysis would require external datasets and joins.

---

## 2. Data Quality Assessment

### 2.1 Missing Values

**High missingness in outcome-related fields:**
- Allegations_Sustained: 112 missing of 149
- Allegations_Unfounded: 112 missing
- Allegations_Unsubstantiated: 112 missing
- Num_Officers_DA: 112 missing
- Disciplinary_Actions: 118 missing
- NonDisciplinary_Actions: 123 missing

Complaint_Date, Closure_Date, Received_By, and Allegations_Made have no empty cells, but "Open" in Closure_Date functions as implicit missingness for closure and outcomes.

**Impact:**
- Time-to-resolution and discipline-rate metrics are only reliable for the subset of 42 closed complaints with valid resolution days
- Any rates (e.g., proportion sustained) must be clearly labeled as calculated **only among records with non-missing outcomes**

### 2.2 Inconsistencies & Formatting Issues

**Dates:**
- Complaint_Date strings like `2023/02/01 05:00:00+00` require parsing to UTC datetimes
- Closure_Date mixes timestamps and the text "Open", so closure parsing must treat "Open" as NaT and track open status separately

**Categorical variation:**
- Allegation labels vary by spelling and case: `Demeanor`, `Demenaor`, `Harrassment`, `Harassment`, `Fail to Act`, `Fail To act`, `fail to Act`
- Received_By contains variants like `CRB`, `CRb`, `CRB `, requiring trimming and uppercasing
- Disciplinary_Actions contains inconsistent tokens: `none`, `None`, `none  `, `None expired`, `nonr`

**Numeric-as-text:**
- Officers_Inolved_Num and Allegations_Made_Num sometimes use strings like `numerous`, `Numerous`, `unk`, `Unknown`

**Mitigation Plan:**
- Standardize dates and create an **Is_Open** flag rather than treating "Open" as a date
- Normalize categorical strings via lowercasing, trimming, and manual mapping (e.g., `demenaor` → `demeanor`)
- Treat `numerous` / `unk` / `unknown` as missing for numeric analysis while preserving raw text in the original columns

### 2.3 Temporal Coverage

- Complaint_Date runs from early 2022 through March 2024 in this extract
- 107 of 149 complaints are marked as open, so much of the most recent data has no outcomes or resolution times yet

**Implications:**
- Annual and quarterly trends for **closing rates** and **sustained outcomes** are strongly biased downward for late 2023–2024 and must be annotated as incomplete
- Resolution-time calculations must handle right-censoring by separating open cases or excluding them from duration statistics

### 2.4 Geographic Coverage

- The dataset does not include precinct, beat, address, or neighborhood identifiers
- No direct geographic coverage or spatial disparity analysis is possible from this CSV alone

---

## 3. Documented Limitations & Bias Risks

### 3.1 What This Data Cannot Answer

- It cannot establish what **actually happened** in any incident; it reflects reported complaints and administrative classifications, not fully adjudicated facts
- It cannot show whether a lower complaint count truly indicates better conduct; changes could reflect barriers to reporting or shifts in trust
- It cannot support officer-level or neighborhood-level patterns without additional identifiers or geospatial data

### 3.2 Structural Biases

- Filing a complaint requires awareness, trust, and time, so some communities (especially marginalized groups) may be underrepresented
- Complaints received via SPD, CRB, and AG may follow different review processes, which could alter closure times and outcomes independent of incident characteristics

**Conclusion:**
This dataset primarily describes **patterns in reported complaints and their administrative handling**, not the full universe of possible misconduct or harm.

---

## 4. Exploratory Analysis (Grounded & Reproducible)

All computations are implemented in a Jupyter notebook using Pandas and plotted with Matplotlib/Seaborn.

### 4.1 Summary Statistics (Pandas-Based)

Key computed metrics include:

- **Complaints per year:**
  - Extracted from `Complaint_Date_dt` grouped by `Year`
  - Visualized in Chart 1 below

- **Open vs closed counts by year and quarter:**
  - `Is_Open` derived from `Closure_Date` text
  - Grouped by `Year` and `YearQuarter`

- **Allegation distribution:**
  - Allegations_Made cleaned into `Allegation_Clean`, then counted
  - Top categories: **fail to act, demeanor, harassment, excessive force, false arrest**

- **Outcome metrics (non-missing subset):**
  - For rows where at least one of Allegations_Sustained / Unfounded / Unsubstantiated is present, summed per year

- **Investigation duration:**
  - For closed cases, `Resolution_Days = Closure_Date_dt - Complaint_Date_dt` in days
  - 42 closed complaints with valid resolution times; median resolution is on the order of a few months

---

## 4.2 Initial Visualizations (with Actual Charts)

### Chart 1 — Complaints per Year (Bar Chart)

**Code (notebook cell):**
```python
import pandas as pd

df = pd.read_csv("SPD_Personnel_Complaints_2021_to_Present.csv")

df["Complaint_Date_dt"] = pd.to_datetime(
    df["Complaint_Date"], errors="coerce", utc=True
)
df["Year"] = df["Complaint_Date_dt"].dt.year

complaints_per_year = df.groupby("Year")["ObjectId"].count()

complaints_per_year.plot.bar(
    title="SPD Personnel Complaints per Year",
    xlabel="Year",
    ylabel="Number of Complaints",
    figsize=(6,4)
)
complaints_per_year
```

**Interpretation:**
- Complaints increase from 2022 to 2023, while 2024 appears lower due to only partial-year data (through March in this extract)
- Recent-year counts should be treated as incomplete and not directly compared to full prior years

---

### Chart 2 — Complaints by Allegation Type (Horizontal Bar Chart)

**Code:**
```python
df["Allegation_Clean"] = (
    df["Allegations_Made"]
    .astype(str)
    .str.strip()
    .str.lower()
    .replace({
        "demenaor": "demeanor",
        "harrassment": "harassment",
        "fail to act ": "fail to act",
        "fail to act,": "fail to act",
        "fail to act, fail to act": "fail to act",
    })
)

alleg_counts = df["Allegation_Clean"].value_counts()

alleg_counts.head(15).sort_values().plot.barh(
    title="Complaints by Allegation Type (Top 15)",
    xlabel="Number of Complaints",
    figsize=(7,5)
)
alleg_counts.head(15)
```

**Interpretation:**
- **Fail to act**, **demeanor**, and **harassment** are the most common complaint types, followed by **excessive force** and **false arrest**, which together account for most cases
- Numerous rare categories form a long tail, and spelling inconsistencies required manual normalization to avoid undercounting common types

---

### Chart 3 — Allegation Outcomes by Year (Stacked Bar)

**Code:**
```python
sub = df[[
    "Year",
    "Allegations_Sustained",
    "Allegations_Unfounded",
    "Allegations_Unsubstantiated"
]].copy()

# Keep rows where at least one outcome field is non-missing
sub = sub.dropna(
    how="all",
    subset=["Allegations_Sustained","Allegations_Unfounded","Allegations_Unsubstantiated"]
)

outcomes_year = sub.groupby("Year")[[
    "Allegations_Sustained",
    "Allegations_Unfounded",
    "Allegations_Unsubstantiated"
]].sum()

ax = outcomes_year.plot(
    kind="bar",
    stacked=True,
    title="Allegation Outcomes by Year (Counts, Non-Missing Subset)",
    figsize=(7,5)
)
ax.set_xlabel("Year")
ax.set_ylabel("Count of Allegations (where outcome recorded)")
outcomes_year
```

**Interpretation:**
- For complaints with recorded outcomes, **sustained** counts are smaller than the combined **unfounded** and **unsubstantiated** counts in each year
- Because outcomes are missing for most records, these bars describe only the **recorded subset** and cannot be generalized to all SPD complaints

---

### Chart 4 — Open vs Closed Complaints by Quarter (Stacked Bar)

**Code:**
```python
# Parse closure date; treat 'Open' as NaT
closure_raw = pd.to_datetime(
    df["Closure_Date"].where(~df["Closure_Date"].str.lower().eq("open")),
    errors="coerce",
    utc=True
)
df["Closure_Date_dt"] = closure_raw
df["Is_Open"] = df["Closure_Date"].str.lower().eq("open")

df["YearQuarter"] = df["Complaint_Date_dt"].dt.to_period("Q")

status_q = (
    df.groupby(["YearQuarter","Is_Open"])["ObjectId"]
      .count()
      .unstack(fill_value=0)
      .rename(columns={True:"Open", False:"Closed"})
)

status_q.plot(
    kind="bar",
    stacked=True,
    title="Open vs Closed Complaints by Quarter",
    figsize=(10,5)
)
status_q
```

**Interpretation:**
- Earlier quarters show a higher share of closed complaints, while **late 2023 and early 2024** quarters have a larger proportion of open cases, indicating a growing or more recent backlog
- Since newer quarters have had less time to resolve cases, part of the high open proportion reflects natural process lag rather than systemic failure

---

### Chart 5 — Resolution Time Histogram (Closed Cases Only)

**Code:**
```python
# Ensure both complaint and closure are timezone-aware in UTC
df["Complaint_Date_dt"] = pd.to_datetime(df["Complaint_Date"], errors="coerce", utc=True)
closure_raw = pd.to_datetime(
    df["Closure_Date"].where(~df["Closure_Date"].str.lower().eq("open")),
    errors="coerce",
    utc=True
)
df["Closure_Date_dt"] = closure_raw

df["Resolution_Days"] = (
    df["Closure_Date_dt"] - df["Complaint_Date_dt"]
).dt.days

closed = df[df["Resolution_Days"].notna() & (df["Resolution_Days"] >= 0)]

ax = closed["Resolution_Days"].plot.hist(
    bins=20,
    title="Resolution Time for Closed Complaints",
    figsize=(7,5),
    edgecolor="black"
)
ax.set_xlabel("Resolution Days")
median_days = closed["Resolution_Days"].median()
ax.axvline(median_days, color="red", linestyle="--", label=f"Median = {median_days:.0f} days")
ax.legend()
median_days
```

**Interpretation:**
- Resolution times are **right-skewed**: many complaints resolve within roughly a few months, but a non-trivial tail extends into longer durations
- The median resolution time (red line) falls within the main cluster of bars, while a small number of cases take substantially longer, highlighting possible complex investigations or delays

---

### Chart 6 — Resolution Time by Allegation Type (Boxplot)

**Code:**
```python
closed_nonnull = closed.copy()
closed_nonnull["Allegation_Clean"] = closed_nonnull["Allegation_Clean"].fillna("unknown")

top_types = closed_nonnull["Allegation_Clean"].value_counts().head(6).index
closed_top = closed_nonnull[closed_nonnull["Allegation_Clean"].isin(top_types)]

import seaborn as sns

plt.figure(figsize=(9,5))
sns.boxplot(
    data=closed_top,
    x="Allegation_Clean",
    y="Resolution_Days"
)
plt.title("Resolution Time by Allegation Type (Closed Cases, Top 6)")
plt.xlabel("Allegation Type (cleaned)")
plt.ylabel("Resolution Days")
plt.xticks(rotation=30, ha="right")
plt.tight_layout()
closed_top.groupby("Allegation_Clean")["Resolution_Days"].median()
```

**Interpretation:**
- All common categories share a similar general scale of resolution times, but **force-related and some legal categories** (e.g., excessive force, false arrest) show higher medians or more high-end outliers than demeanor or harassment
- These patterns suggest some allegation types may be procedurally more complex to investigate, though statistical tests and small sample sizes caution against strong causal claims

---

### Chart 7 — Complaints by Intake Source (Received_By)

**Code:**
```python
df["Received_Clean"] = df["Received_By"].astype(str).str.strip().str.upper()
source_counts = df["Received_Clean"].value_counts()

source_counts.plot.bar(
    title="Complaints by Intake Source",
    xlabel="Intake Source",
    ylabel="Number of Complaints",
    figsize=(6,4)
)
source_counts
```

**Interpretation:**
- **CRB** and **SPD** are the dominant intake channels, with **AG** and other sources handling relatively few complaints
- Differences in channel usage may reflect community awareness, outreach, or policy rather than incident frequency or severity alone

---

## 5. LLM-Assisted Analysis (Controlled & Validated)

### 5.1 Hypothesis Generation (LLM-Assisted)

LLMs are used to propose hypotheses and narrative leads, for example:

- **H1:** Force-related allegations (e.g., excessive force, unlawful arrest/search) take longer to resolve than demeanor or policy complaints
- **H2:** Complaints received by external bodies (CRB/AG) show different closure patterns or outcome distributions than those received directly by SPD
- **H3:** Complaint volume varies seasonally, with potential clustering in certain months (e.g., summer)

These hypotheses are translated into specific Pandas queries and tested quantitatively.

### 5.2 Validation Against Ground Truth

For each hypothesis, the notebook computes:
- Group-level resolution statistics (median, IQR) by Allegation_Clean and Received_Clean
- Proportions of sustained/unfounded/unsubstantiated outcomes where outcome data exist

LLM narrative statements are accepted only when supported by clearly visible differences in the charts (e.g., non-overlapping medians) and documented in code cells.

Unsupported LLM assertions are either removed or reframed as "no clear pattern in this sample."

### 5.3 Bias Checks (Task 8 Techniques)

- Prompts are written to ask neutrally about "patterns" rather than leading with evaluative language (e.g., "prove discipline is inadequate")
- Alternative prompt framings are compared to check for narrative drift toward overly accusatory or overly exculpatory explanations
- Since the dataset lacks demographic fields, LLM outputs are constrained from speculating about race, age, or neighborhood impacts

**Result:**
LLMs tend to foreground **accountability themes** (sustained vs unsustained, discipline vs none), so prompts must explicitly mention missing outcome data and require any narratives to highlight data gaps and uncertainty.

---

## 6. Key Findings & Hypotheses for Phase 3

### Finding 1 – Resolution and Allegation Type

Closed complaints show meaningful variation in resolution times by allegation category, with force- and legality-related allegations displaying somewhat longer or more variable resolution durations than demeanor or harassment cases.

### Finding 2 – Open Case Backlog and Recent Data

A large fraction of complaints in the most recent quarters remain open (107 of 149 total), limiting any conclusions about current closure or discipline rates and underscoring the need for transparent communication about incomplete data.

### Finding 3 – Intake Source and Outcomes (Hypothesis-Driven)

CRB and SPD receive the bulk of complaints, and preliminary groupings suggest that closure patterns and timelines may differ by intake source, motivating more detailed modeling in Phase 3 to separate process effects from incident characteristics.

**These findings and hypotheses will drive Phase 3:** building a reproducible pipeline, refining temporal and categorical analyses, and implementing a public-facing dashboard that clearly surfaces both insights and data limitations.

---

## Appendix: Summary of Chart Generation Code

All seven visualizations are generated using Matplotlib and Seaborn in a Jupyter notebook environment. The code cells follow a consistent pattern:

1. **Data Preparation:** Parse dates, clean categorical fields, derive new variables (e.g., Is_Open, Resolution_Days, Year, YearQuarter, Allegation_Clean)
2. **Aggregation:** Group by relevant dimensions and compute counts, medians, or other statistics
3. **Visualization:** Plot with appropriate chart type (bar, histogram, boxplot) and label clearly
4. **Interpretation:** Add markdown cells with captions, interpretations, and limitations for each figure

All plots include:
- Descriptive titles and axis labels
- Markdown cells with 1–2 sentence interpretations
- Explicit notes on data limitations (e.g., "open cases excluded from duration calculations")

---

**End of Phase 2 Exploration Report**

*This report serves as the foundation for Phase 3 development (Weeks 5–10), in which the analysis pipeline will be productionized, the dashboard prototype built, and stakeholder-facing narratives refined.*