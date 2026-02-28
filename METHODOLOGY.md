# Methodology — SPD Personnel Complaints Analysis Pipeline

## 1. Objective and Scope

The project analyzes SPD personnel complaint records from 2021 onward to provide transparent, descriptive views of complaint volume, allegation categories, intake channels, case outcomes (where recorded), and resolution times. The emphasis is on reproducible processing and responsible interpretation, not on causal inference or predictive modeling at this stage.

---

## 2. Data Processing Pipeline

### 2.1 Ingestion

- Load the raw CSV from `data/raw/` (e.g., `SPD_Personnel_Complaints_2021_to_Present.csv`).  
- Standardize column names and validate the presence of required fields (complaint date, closure date, allegation text, intake source, unique identifier, outcome columns).  

### 2.2 Date Handling and Derived Time Features

- Parse `Complaint_Date` and `Closure_Date` using timezone-aware parsing.  
- Treat textual markers such as `"Open"` in the closure field as open cases and set closure date to missing.  
- Derive:  
  - `Year` as the calendar year of the complaint date.  
  - `YearQuarter` as a quarterly period based on the complaint date.  
  - `Is_Open` as a boolean indicating whether a complaint is open at data pull.  
  - `Resolution_Days` as the difference (in days) between closure and complaint dates for closed cases, dropping negative or nonsensical values.  

### 2.3 Categorical Normalization

- **Allegation Types**  
  - Convert text to lowercase, strip whitespace, fix common misspellings, and collapse near-duplicate labels into a cleaned `Allegation_Clean` field.  
  - This supports top-N analyses and boxplots by allegation type.  

- **Intake Source**  
  - Normalize `Received_By` into a small set of consistent labels (e.g., CRB, SPD, AG, Other) via trimming, case normalization, and rule-based mapping.  

### 2.4 Subsets for Specific Analyses

- Closed-case subset (non-missing, non-negative `Resolution_Days`) for resolution-time charts.  
- Recorded-outcome subset (non-missing sustained / unfounded / unsubstantiated fields) for disposition charts.  

---

## 3. Analytical Approach and Visuals

The analysis is exploratory and descriptive, implemented through seven main visualizations:

1. **Complaints per Year**  
   - Bar chart of total complaints by calendar year, highlighting temporal trends and noting that the most recent year may be incomplete.  

2. **Complaints by Allegation Type (Top 15)**  
   - Horizontal bar chart of the most frequent cleaned allegation categories, showing which allegation types account for most complaints.  

3. **Disposition Outcomes by Year**  
   - Stacked bar chart of sustained, unfounded, and unsubstantiated counts by year, restricted to records with recorded outcomes.  

4. **Open vs. Closed Complaints by Quarter**  
   - Stacked bar chart counts of open and closed complaints by complaint quarter, emphasizing how recent quarters naturally show more open cases.  

5. **Resolution Time Histogram (Closed Cases)**  
   - Histogram of `Resolution_Days` with a median reference line, illustrating how quickly most complaints are resolved and the presence of a long tail.  

6. **Resolution Time by Allegation Type**  
   - Boxplot of `Resolution_Days` for the most common allegation types, allowing visual comparison of medians and spread.  

7. **Complaints by Intake Source**  
   - Bar chart showing the number of complaints by normalized intake source (e.g., CRB, SPD, AG, Other).  

These visuals are intended to spark questions and support oversight discussions, not to provide final answers by themselves.

---

## 4. LLM Usage and Validation

When a language model is used in this project, its role is strictly limited to assisting with narrative text (such as figure captions and short summaries) rather than generating new metrics.

- The model receives summary statistics (counts, medians, percentages) as input and is instructed to describe them without inventing new numbers or categories.  
- Generated text is checked against the computed outputs; any numeric discrepancies are corrected or the text is discarded.  
- Captions and summaries are prompted to include explicit caveats about missing data, recency bias, and censoring where relevant.  

This workflow treats the LLM as a writing assistant layered on top of a deterministic analysis pipeline.

---

## 5. Statistical Methods

The analysis relies on:

- Grouped counts and proportions (e.g., complaints per year, share of complaints by allegation type or intake source).  
- Distribution summaries (histograms, medians, interquartile ranges) for resolution times.  
- Visual comparisons across groups (boxplots) to highlight potential differences in resolution patterns.  

No formal hypothesis tests or regression models are implemented in this phase; any apparent differences between groups should be treated as exploratory and may motivate future, more formal studies.

---

## 6. Limitations and Caveats

- **Outcome Missingness:** many complaints lack recorded outcomes; charts based on sustained / unfounded / unsubstantiated fields describe only that subset and may not represent all cases.  
- **Recency and Backlog:** recent quarters show more open cases and partial data; trends in open-case counts over time are influenced by process lag and reporting practices.  
- **Censoring of Open Cases:** resolution-time metrics exclude open complaints, so long-running unresolved cases are not visible in those distributions.  
- **Data Entry Quality:** typos, inconsistent labels, and occasional impossible dates are cleaned where feasible but cannot be fully eliminated.  
- **Context Outside the Dataset:** the data does not encode investigative context, officer history, or policy changes, so charts should be interpreted alongside domain expertise and additional qualitative information.  

These limitations are also reflected in the dashboard text and README to help users interpret the outputs responsibly.
