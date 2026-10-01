# Methodology summary

## Data structure

The cleaned dataset contains one row per organism isolate. A single sample can produce multiple organism isolates, so the project keeps organism-level rows while deriving sample-level measures from distinct Sample IDs.

The final cleaned register contains 10,943 distinct culture-positive samples and 12,188 organism isolates.

## Stage 1 — Cleaning and descriptive analysis

The raw register was cleaned and standardised by:

- standardising organism names
- mapping hospital units
- grouping specimen descriptions
- deriving resistance and infection-origin indicators
- handling blanks and non-organism entries
- reconciling duplicate records
- retaining multiple organisms from the same sample as separate isolate rows

Descriptive analysis was performed using frequencies, proportions, cross-tabulations and monthly summaries.

## Stage 2 — Resistance phenotype profile

The register records resistance phenotypes rather than a complete organism-by-antibiotic susceptibility matrix, so a conventional cumulative antibiogram was not constructed.

Resistance phenotypes were profiled using the organism-level applicability framework. The six main phenotype indicators were CRE, ESBL, MDR, VRE, MRSA and MR-CoNS.

The statistical resistance outcome is:

**Y = 1** when an applicable resistance phenotype is documented for the isolate.

A blank resistance field means no documented phenotype; it is not treated as laboratory-confirmed susceptibility.

## Stage 3 — Bivariate association

Four categorical predictors were compared with the binary resistance outcome:

- organism group
- specimen type
- hospital unit type
- infection origin

For each, a contingency table, chi-square statistic, p-value and Cramer's V were calculated. Holm correction was applied across the four tests.

Because multiple isolates can come from the same sample, a sensitivity analysis used one randomly selected isolate per Sample ID and repeated the calculations 200 times.

## Stage 4 — Logistic regression

The multivariable model is:

logit(P(Y=1)) = β0 + β1X1 + ... + β17X17

The predictors are organism group, specimen type, hospital unit type and infection origin.

Cluster-robust standard errors are used with Sample ID as the clustering variable.

The model is used for:

- **Inference:** adjusted odds ratios and confidence intervals
- **Prediction:** held-out AUC, Brier score, calibration and grouped cross-validation

Grouped splitting is used so that isolates from the same sample never appear in both training and test data.

## Stage 5 — SPC

The monitored quantity is the monthly proportion of culture-positive samples classified as healthcare-associated.

A p-chart is used with:

- a study-period centre line
- month-specific 3-sigma control limits
- a runs rule for sustained movement
- a Cochran-Armitage trend test

The chart is not a hospital-wide HAI incidence rate because patient-day or device-day denominators were not available.

## Main analytical flow

Raw register  
↓  
Cleaning and standardisation  
↓  
Descriptive analysis  
↓  
Resistance phenotype profile  
↓  
Chi-square + Cramer's V  
↓  
Logistic regression + prediction  
↓  
SPC + trend test  
↓  
Power BI dashboard
