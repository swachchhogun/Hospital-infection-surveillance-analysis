# Hospital Infection Surveillance Analysis

Statistical analysis of a 14-month hospital microbiology register covering antimicrobial resistance, healthcare-associated infections, logistic regression, statistical process control, and an interactive Power BI surveillance dashboard.

## Project overview

This project was developed as part of an M.Sc. Statistics internship.

The analysis used a cleaned microbiology register containing:
- **10,943** distinct culture-positive samples
- **12,188** organism isolates
- **5,311** patients
- **April 2025 – May 2026**

The workflow was:
**Cleaning → descriptive analysis → resistance phenotype profile → chi-square/Cramer's V → logistic regression → prediction → SPC/trend analysis → Power BI dashboard**

## Statistical methods

- Frequencies, proportions and cross-tabulations
- Resistance phenotype profiling
- Chi-square tests of association
- Cramer's V effect sizes
- Holm correction for multiple tests
- Multivariable logistic regression
- Adjusted odds ratios and confidence intervals
- Cluster-robust standard errors by Sample ID
- Grouped train/test split
- ROC-AUC, Brier score and calibration
- Grouped 5-fold cross-validation
- Statistical process control p-chart
- Cochran-Armitage trend test

## Main findings

- HAI proportion among culture-positive samples: **46.8%**
- Documented applicable resistance among eligible isolates: **55.0%**
- Organism group showed the largest bivariate association with resistance: **Cramer's V = 0.473**
- Held-out logistic-regression AUC: **0.778**
- Grouped 5-fold mean AUC: **0.781**
- **7 of 14 months** were outside the SPC control limits
- Cochran-Armitage trend test: **Z = -3.23, p = 0.001**

## Repository structure

```text
Hospital-infection-surveillance-analysis/
│
├── README.md
├── DATA_ACCESS.md
├── PUBLISHING_CHECKLIST.md
├── .gitignore
│
├── notebooks/
│   ├── Analysis_Notebook.ipynb
│   └── README.md
│
├── docs/
│   ├── Analysis_Code_Walkthrough.md
│   ├── Dashboard_Build_Guide.md
│   └── Methodology_Summary.md
│
├── report/
│   └── README.md
│
├── dashboard/
│   └── README.md
│
└── outputs/
    └── README.md
```

## Analysis notebook

The main technical file is [Analysis_Notebook.ipynb](notebooks/Analysis_Notebook.ipynb).

It contains the code for the full statistical workflow and the report figures. The accompanying [code walkthrough](docs/Analysis_Code_Walkthrough.md) is the study/reference guide for understanding the code step by step.

## Dashboard

The Power BI dashboard is documented in [Dashboard_Build_Guide.md](docs/Dashboard_Build_Guide.md).

The dashboard has five pages:
1. Overview
2. Organisms
3. Antimicrobial resistance
4. Hospital units
5. Monthly surveillance

## Report

The university internship report is kept in the report folder. The final DOCX/PDF should be added there after the last university-specific details are filled in.

## Data access

The original hospital dataset is **not included**. See [DATA_ACCESS.md](DATA_ACCESS.md) for the publishing/data-handling note.

## Tools

Python · Pandas · SciPy · statsmodels · Matplotlib · Microsoft Excel · Power BI

## Author

Swachchho Gun  
M.Sc. Statistics, St. Xavier's University, Kolkata