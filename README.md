# Oncology Clinical Trial Success Analysis
### Data Engineering & Analytics Pipeline | Interview Assignment

---

## What This Project Does

This repository implements a reproducible end-to-end pipeline that transforms
a raw 1,000-trial oncology registry extract into actionable insights on 
clinical trial success rates.

The raw data is a flat Excel file structured for data entry — not analysis.
It contains list-valued cells, no success flag, inconsistent date coverage,
and seven columns storing Python objects as plain strings.

This pipeline solves each of those problems systematically before computing
stratified success rates across four analytical dimensions.

---

## Repository Structure

oncology-trial-analysis/
├── data/
│   └── SampleDateExtract.xlsx        # Raw input (do not modify)
├── notebooks/
│   ├── 01_data_quality.ipynb         # Part 1A: profiling & audit
│   ├── 02_schema_cleaning.ipynb      # Part 1B + 2A: cleaning & success proxy
│   ├── 03_success_analysis.ipynb     # Part 2B: stratified success rates
│   └── 04_extended_audit.ipynb       # Extended: structural checks & improved definitions
├── outputs/
│   ├── quality_report.csv            # Field completeness report
│   ├── clean_trials.csv              # Core clean table (1000 rows, 15 cols)
│   ├── trial_indications.csv         # Normalised indications (2231 rows)
│   ├── trial_drugs.csv               # Normalised drugs (2716 rows)
│   ├── trial_technologies.csv        # Normalised technologies (1966 rows)
│   ├── sr_by_phase.csv / .png        # Success rates by phase
│   ├── sr_by_technology.csv / .png   # Success rates by technology
│   ├── sr_by_indication.csv / .png   # Success rates by indication group
│   ├── sr_heatmap_indication_phase   # Indication × phase cross-table
│   └── success_definition_comparison # Four success definitions compared
├── requirements.txt
└── README.md

---

## How to Run

### 1. Install dependencies
```bash
pip install -r requirements.txt
```

### 2. Place the raw data file
Put `SampleDateExtract.xlsx` in the `data/` folder.


### 3. Run notebooks in order
Each notebook is independently reproducible — it reads from source files,
not from another notebook's memory.

01_data_quality.ipynb      → generates quality_report.csv
02_schema_cleaning.ipynb   → generates clean_trials.csv + child tables
03_success_analysis.ipynb  → generates all sr_by_*.csv and plots
04_extended_audit.ipynb    → generates success_definition_comparison.png

---

## Key Design Decisions

### 1. Why ast.literal_eval and not eval?
Seven columns store Python lists as plain strings — a data entry artifact.
`ast.literal_eval` safely parses these back to Python lists.
`eval()` was deliberately avoided — it executes arbitrary code and is a
security risk in a pipeline processing external data.

### 2. Why normalise into child tables?
Multi-valued fields (indications, drugs, technologies, targets) cannot be
aggregated with GROUP BY in their raw form. Exploding them into 1:many
child tables enables clean per-dimension analysis without string parsing
at query time.

### 3. Why is the success denominator 620, not 1000?
380 trials are unresolved: still recruiting, active, pending, or unknown.
Including them as failures would undercount success. Including them as
successes would overcount. The only defensible choice is exclusion.
Denominator = COMPLETED + TERMINATED + WITHDRAWN + SUSPENDED = 620.

### 4. Why three success definitions?
The binary proxy (COMPLETED = 1) is administratively defensible but
biologically thin. Two additional definitions are implemented:
- **Enrollment-gated**: requires actual patients enrolled > 0 (removes
  phantom completions — trials that completed on paper with zero patients)
- **Phase-weighted**: multiplies completion by a phase weight (0.3 for
  Phase 1, 0.9 for Phase 3) — reflecting the biological evidence bar
  each phase actually clears

---

## Key Findings

| Dimension | Key Finding |
|---|---|
| Overall | 73.1% process success rate among 620 resolved trials |
| Phase | Phase 1 (73.9%) outperforms Phase 2 (70.1%) — Phase 2 is the valley of death |
| Technology | Small Molecule dominates (1446 entries); ADC/CAR-T emerging but low n |
| Indication | Lung Cancer lowest (~60%) — saturated checkpoint inhibitor landscape |
| Late-phase | Only ~7% of all trials achieve Phase 3/4 completion — the real signal |

---

## Important Limitations

**This pipeline measures process success, not therapeutic success.**

COMPLETED means the trial ran to protocol. It does not mean the drug worked.
A Phase 3 trial can COMPLETED and still fail its primary endpoint.

To measure therapeutic success, this dataset would need to be joined to
the ClinicalTrials.gov results tables, which contain primary endpoint
met (Yes/No) for each completed trial.

---

## Data Quality Notes

- **Encoding issue**: Greek letters in target names (PDGFRα, TNF-α) are
  stored with corrupted encoding (PDGFRî±). Flagged but not corrected —
  affects string matching on target names only, not success rate calculations.
  Fix: `unicodedata.normalize()` in a production pipeline.

- **Structurally explained nulls**: 52 missing completion dates belong
  entirely to trials that never completed (active/withdrawn). These are
  not data entry errors.

- **List field mismatches**: Some trials have more drugs than technology
  annotations — those drugs are excluded from technology-stratified analysis.
  Flagged in Notebook 4.

---

## Requirements
