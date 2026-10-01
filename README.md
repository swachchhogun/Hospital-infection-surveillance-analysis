# Hospital Infection Surveillance Analysis

Statistical analysis of a 14-month hospital microbiology register, covering antimicrobial resistance, healthcare-associated infections, logistic regression, statistical process control, and an interactive Power BI surveillance dashboard.

## Project overview

This project was developed as part of an M.Sc. Statistics internship.

The analysis used a cleaned microbiology register covering:

- 10,943 distinct culture-positive samples
- 12,188 organism isolates
- 5,311 patients
- April 2025 to May 2026

The project follows the full workflow:

**Data cleaning → descriptive analysis → resistance phenotype profiling → chi-square/Cramer's V → logistic regression → prediction → SPC p-chart and trend test → Power BI dashboard**

## Main methods

### Descriptive analysis
- Frequencies and proportions
- Cross-tabulations
- Pivot tables and summary tables
- Monthly surveillance summaries

### Resistance analysis
- Documented resistance phenotype profiling
- Organism-level applicability rules
- Sample-level resistance burden for descriptive reporting

### Statistical analysis
- Chi-square tests of association
- Cramer's V effect sizes
- Holm correction for multiple tests
- Logistic regression
- Adjusted odds ratios and confidence intervals
- Cluster-robust standard errors by Sample ID
- Grouped train/test splitting
- ROC-AUC, Brier score and calibration
- Grouped 5-fold cross-validation
- SPC p-chart with month-specific 3-sigma limits
- Cochran-Armitage trend test

## Key results

- Overall culture-positive sample HAI proportion: **46.8%**
- Overall documented applicable resistance among eligible isolates: **55.0%**
- Organism group had the largest association with resistance (Cramer's V = **0.473**)
- Held-out logistic-regression AUC: **0.778**
- 5-fold grouped cross-validation mean AUC: **0.781**
- SPC identified **7 of 14 months** outside the control limits
- Cochran-Armitage trend test: **Z = -3.23, p = 0.001**

## Repository contents

```
.
├── README.md
├── .gitignore
├── DATA_ACCESS.md
├── notebooks/
│   └── Analysis_Notebook.ipynb
├── docs/
│   ├── Analysis_Code_Walkthrough.md
│   ├── Dashboard_Build_Guide.md
│   └── Methodology_Summary.md
├── report/
│   ├── Infection_Surveillance_Report_FINAL.pdf
│   └── Infection_Surveillance_Report_FINAL.docx
├── dashboard/
│   └── README.md
└── outputs/
    └── README.md
```

## Data access

The original hospital dataset is **not included in this repository**. It contains internship data from a hospital surveillance register and is not being published here.

The notebook is provided so that the statistical workflow can be studied and reproduced when the authorised dataset is available locally.

See [DATA_ACCESS.md](DATA_ACCESS.md).

## Report

The complete internship report is included in the `report/` folder.

## Dashboard

The Power BI dashboard is documented in [Dashboard_Build_Guide.md](docs/Dashboard_Build_Guide.md). The original hospital data used to build the dashboard is not included.

## Learning the analysis

The main notebook is [Analysis_Notebook.ipynb](notebooks/Analysis_Notebook.ipynb).

The accompanying [Analysis_Code_Walkthrough.md](docs/Analysis_Code_Walkthrough.md) explains the code and the reasoning behind each step.

## Tools

Python · Pandas · SciPy · statsmodels · Matplotlib · Microsoft Excel · Power BI

## Author

Swachchho Gun  
M.Sc. Statistics, St. Xavier's University, Kolkata
