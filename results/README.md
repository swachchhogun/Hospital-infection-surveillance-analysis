# Result tables

These CSV files are aggregate analytical outputs used in the internship report and dashboard workflow.

They do **not** contain the raw hospital register. The resistance anomaly summary is deliberately aggregated and does not expose Sample IDs.

| File | Purpose |
|---|---|
| `project_key_metrics.csv` | Main study, regression, prediction and SPC summary metrics |
| `data_dictionary.csv` | Variables used in the analysis |
| `applicability_matrix.csv` | A-priori phenotype applicability rules |
| `resistance_profile.csv` | Organism-group resistance phenotype profile |
| `organism_applicability_audit.csv` | Audit across the 136 organism labels |
| `resistance_anomaly_summary.csv` | Aggregate incompatible organism-phenotype records |
| `bivariate_association_results.csv` | Chi-square, Holm-adjusted p-values and Cramer's V |
| `bivariate_sensitivity_analysis.csv` | One-isolate-per-sample sensitivity analysis |
| `logistic_adjusted_odds_ratios.csv` | Adjusted odds ratios and confidence intervals |
| `logistic_threshold_performance.csv` | Held-out classification metrics by threshold |
| `logistic_calibration.csv` | Predicted versus observed risk by decile |
| `logistic_grouped_cv.csv` | Grouped five-fold cross-validation results |
| `spc_p_chart_monthly.csv` | Monthly HAI proportions and month-specific control limits |
| `spc_special_cause_signals.csv` | SPC signal summary |
