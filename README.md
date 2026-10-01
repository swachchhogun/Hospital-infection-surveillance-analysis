# Hospital Infection Surveillance Analysis

Statistical analysis of a 14-month hospital microbiology register covering antimicrobial resistance, healthcare-associated infections, logistic regression, statistical process control, and an interactive Power BI surveillance dashboard.

## Project overview

Developed as an M.Sc. Statistics internship project using a cleaned microbiology register containing **10,943 distinct culture-positive samples**, **12,188 organism isolates**, and **5,311 patients** from **April 2025 to May 2026**.

## Workflow

**Cleaning → descriptive analysis → resistance phenotype profiling → chi-square/Cramer's V → logistic regression → prediction → SPC/trend analysis → Power BI dashboard**

## Main methods

- Frequencies, proportions and cross-tabulations
- Resistance phenotype profiling
- Chi-square tests and Cramer's V
- Holm correction for multiple tests
- Multivariable logistic regression
- Adjusted odds ratios and confidence intervals
- Cluster-robust standard errors by Sample ID
- Grouped train/test split and grouped 5-fold cross-validation
- ROC-AUC, Brier score and calibration
- Statistical process control p-chart
- Cochran-Armitage trend test

## Main findings

- HAI proportion among culture-positive samples: **46.8%**
- Documented applicable resistance among eligible isolates: **55.0%**
- Organism group: **Cramer's V = 0.473**
- Held-out logistic-regression AUC: **0.778**
- Grouped 5-fold mean AUC: **0.781**
- **7 of 14 months** outside SPC control limits
- Cochran-Armitage trend test: **Z = -3.23, p = 0.001**

## Repository structure

```text
├── README.md
├── .gitignore
├── notebooks/
│   ├── Analysis_Notebook.ipynb
│   └── README.md
├── docs/
│   ├── Analysis_Code_Walkthrough.md
│   ├── Dashboard_Build_Guide.md
│   └── Methodology_Summary.md
└── report/
    ├── Infection_Surveillance_Report_FINAL.pdf
    └── Infection_Surveillance_Report_FINAL.docx
```

## Analysis notebook

[Analysis_Notebook.ipynb](notebooks/Analysis_Notebook.ipynb) contains the statistical workflow and analysis code. The accompanying [code walkthrough](docs/Analysis_Code_Walkthrough.md) explains the code and reasoning step by step.

## Dashboard

The Power BI dashboard is documented in [Dashboard_Build_Guide.md](docs/Dashboard_Build_Guide.md). The five dashboard pages are Overview, Organisms, Antimicrobial resistance, Hospital units, and Monthly surveillance.

## Report

The final internship report is provided in the `report/` folder, including the complete written analysis and embedded figures.

## Data

The underlying hospital dataset is not part of this repository.

## Tools

Python · Pandas · SciPy · statsmodels · Matplotlib · Microsoft Excel · Power BI

## Author

Swachchho Gun  
M.Sc. Statistics, St. Xavier's University, Kolkata