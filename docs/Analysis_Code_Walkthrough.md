# Statistical Analysis — Reproducible Code Walkthrough

**Culture-positive infection surveillance · Steps 1–4**

Every cell below has been run end-to-end and reproduces the exact figures in the report. This covers the full analytical pipeline and all report figures; it does not include the Word/Excel export code, which is cosmetic. Run the cells **in order, top to bottom** — each builds on the last.

---

## Expected output at each stage

Use this as your checklist. If your numbers match, you've reproduced the analysis correctly.

| Stage | Expected result |
|---|---|
| Load | 12,188 rows · 10,943 samples |
| Step 1 | 8,162 eligible isolates · 7,336 samples · 4,491 resistant (55.0%) · 48 anomalies |
| Step 2 | Organism V=0.473 · Specimen V=0.149 · HAI V=0.089 · Unit V=0.044 · ICC=0.184 · DEFF=1.021 |
| Step 3 | n=8,147 · pseudo-R²=0.197 · Acinetobacter OR=2.12 · AUC=0.778 · CV AUC=0.781 |
| Step 4 | Centre line 46.8% · 7 signals · trend Z=−3.23, p=0.0012 |

---

# SETUP

## Cell 0.1 — Install (Colab only, run once)

```python
!pip install statsmodels scikit-learn openpyxl -q
```

*Colab has pandas/numpy/scipy already. This adds the modelling libraries.*

## Cell 0.2 — Upload the data (Colab only)

```python
from google.colab import files
uploaded = files.upload()      # select final_submission_prepped__3_.xlsx
```

*Colab runs on a remote machine, so the file must be uploaded before it can be read. If the filename differs, run `import os; print(os.listdir())` to see the exact name.*

## Cell 0.3 — Imports

```python
import pandas as pd
import numpy as np
from scipy import stats
import statsmodels.api as sm
import statsmodels.formula.api as smf
from statsmodels.stats.multitest import multipletests
from statsmodels.stats.outliers_influence import variance_inflation_factor
from sklearn.model_selection import GroupShuffleSplit, GroupKFold
from sklearn.metrics import roc_auc_score, roc_curve, confusion_matrix, brier_score_loss
import matplotlib.pyplot as plt
```

**What each library does**

| Library | Role |
|---|---|
| `pandas` | Handles the data table |
| `numpy` | Numerical operations |
| `scipy.stats` | Chi-square test, normal distribution |
| `statsmodels` | Logistic regression, VIF, multiple-testing correction |
| `sklearn` | Grouped train/test splitting and prediction metrics |
| `matplotlib` | Figures |

## Cell 0.4 — Load the data and name the columns

```python
FILE = "final_submission_prepped__3_.xlsx"     # adjust if your filename differs
df = pd.read_excel(FILE, sheet_name="Master")

# Column names stored as constants so a rename only has to be fixed in one place
SID   = "Sample ID"
ORG   = "Organism (standardized)"
RES   = "Resistance / strain"
SPEC  = "Sample Type (category)"      # the GROUPED column (32 values) - NOT "(cleaned)"
UNIT  = "Department_std"
HAI   = "HAI / Device-related"
MONTH = "Month"

print("rows:", len(df), "| samples:", df[SID].nunique())
```

> **Critical:** use `Sample Type (category)` — 32 grouped values. The file also contains `Sample Type (cleaned)` with 1,327 granular values ("BLOOD FROM RIGHT HAND"), which is *not* suitable for analysis.

**Expected:** `rows: 12188 | samples: 10943`

The gap exists because the data is stored at **isolate level** — one row per organism. A sample growing three organisms occupies three rows, all sharing one Sample ID.

## Cell 0.5 — Data-integrity checks

```python
# Do unit and HAI status ever disagree between rows of the same sample?
print("samples with inconsistent unit:", (df.groupby(SID)[UNIT].nunique() > 1).sum())
print("samples with inconsistent HAI :", (df.groupby(SID)[HAI].nunique() > 1).sum())
print("HAI values present:", df[HAI].dropna().unique())
```

**Expected:** both counts `0`, and HAI values `['OUTSIDE' 'HAI']`.

*This proves that unit and HAI are properties of the **sample**, so aggregating them to sample level (needed in Step 4) is exact rather than an approximation. Never assume this — verify it.*

---

# STEP 1 — Resistance phenotype profile

**The question:** which resistance phenotypes exist, in which organisms, and what counts as "resistant"?

This step is not a statistical test. It **defines the outcome variable** used by every later step.

## Cell 1.1 — Flag each phenotype separately

```python
PHENOTYPES = ["CRE", "ESBL", "MDR", "VRE", "MRSA", "MRCONS"]

R = df[RES].astype(str).str.upper()

for p in PHENOTYPES:
    df[p] = R.str.contains(rf"\b{p}\b", regex=True, na=False).astype(int)

# MSSA = methicillin-SENSITIVE: tracked separately, never counted as resistance
df["MSSA"] = R.str.contains(r"\bMSSA\b", regex=True, na=False).astype(int)

print(df[PHENOTYPES + ["MSSA"]].sum())
```

**Line by line**

- `.astype(str).str.upper()` — forces text and upper-cases it so matching is reliable.
- `rf"\b{p}\b"` — `\b` is a **word boundary**. Without it, searching "MRSA" would also match inside "MRCONS" and inflate the MRSA count. This is a real bug the boundary prevents.
- `.astype(int)` — converts True/False to 1/0.
- Separate columns per phenotype (not one combined flag) because the profile table needs each phenotype's rate individually.

**MSSA is deliberately excluded from the resistant set** — it is methicillin-*sensitive* and still treatable. Counting it would over-state the resistance burden.

## Cell 1.2 — The a priori applicability rule

```python
# Explicit organism sets - every one of the 136 organisms in the file is assigned.
ENTEROBACTERALES = {"KLEBSIELLA PNEUMONIAE","E. COLI","PROTEUS MIRABILIS","PROTEUS VULGARIS",
"PROTEUS SP","SERRATIA MARCESCENS","SERRATIA LIQUEFACIENS","ENTEROBACTER SP","ENTEROBACTER CLOACAE",
"CITROBACTER KOSERI","CITROBACTER SP","CITROBACTER FREUNDII","PROVIDENCIA SP","PROVIDENCIA RETTGERI",
"MORGANELLA MORGANII"}

NON_FERMENTERS = {"PSEUDOMONAS AERUGINOSA","PSEUDOMONAS SP","PSEUDOMONAS PUTIDA","ACINETOBACTER SP",
"ACINETOBACTER LWOFFII","ACINETOBACTER JUNII","STENOTROPHOMONAS MALTOPHILIA","BURKHOLDERIA CEPACIA",
"BURKHOLDERIA SP","CHRYSEOBACTERIUM INDOLOGENES","CHRYSEOBACTERIUM SP","ELIZABETHKINGIA MENINGOSEPTICA",
"ACHROMOBACTER XYLOSOXIDANS","ACHROMOBACTER SP","RALSTONIA PICKETTII","RALSTONIA INSIDIOSA",
"MYROIDES SP","SPHINGOMONAS PAUCIMOBILIS","SPHINGOBACTERIUM SPIRITIVORUM","SPHINGOBACTERIUM SP",
"COMAMONAS TESTOSTERONI","OCHROBACTRUM ANTHROPI"}

ENTEROCOCCI = {"ENTEROCOCCUS FAECIUM","ENTEROCOCCUS FAECALIS","ENTEROCOCCUS SP"}
S_AUREUS    = {"STAPHYLOCOCCUS AUREUS"}
CONS        = {"STAPHYLOCOCCUS (COAGULASE NEGATIVE)","STAPHYLOCOCCUS EPIDERMIDIS",
"STAPHYLOCOCCUS HAEMOLYTICUS","STAPHYLOCOCCUS HOMINIS","STAPHYLOCOCCUS LUGDUNENSIS","STAPHYLOCOCCUS SP"}

# Which phenotypes are DEFINED for each class - decided on microbiology, before seeing any outcome
APPLICABLE = {
    "Enterobacterales": ["CRE", "ESBL", "MDR"],   # CRE = carbapenem-resistant ENTEROBACTERALES
    "Non-fermenter":    ["MDR"],                   # NOT CRE - Pseudomonas/Acinetobacter aren't Enterobacterales
    "Enterococcus":     ["VRE"],
    "S. aureus":        ["MRSA"],
    "CoNS":             ["MRCONS"],
}

def classify_organism(o):
    o = str(o).strip()
    if o in ENTEROBACTERALES: return "Enterobacterales"
    if o in NON_FERMENTERS:   return "Non-fermenter"
    if o in ENTEROCOCCI:      return "Enterococcus"
    if o in S_AUREUS:         return "S. aureus"
    if o in CONS:             return "CoNS"
    return "INELIGIBLE"    # fungi, viruses, mycobacteria, parasites, enteric pathogens, etc.

df["org_class"] = df[ORG].apply(classify_organism)
print(df["org_class"].value_counts())
```

**Why this matters more than anything else in the analysis**

The six phenotypes are **bacterial susceptibility categories**. A *Candida* isolate with no CRE label does **not** mean "Candida was sensitive" — it means *the test does not exist for fungi*. Including such organisms would create **quasi-complete separation** (categories with 0% resistance), producing unstable or infinite odds ratios.

**Eligibility is decided a priori, on biology, before looking at any resistance counts.** Excluding a category *because it had zero events* would be selecting on the outcome — a genuine methodological error. The correct logic is: *biology → exclusion → then observe that excluded groups indeed had no events* (corroboration, not justification).

Two specific points that are easy to get wrong:

- **CRE applies to E. coli.** CRE means Carbapenem-Resistant *Enterobacterales*, and *E. coli* is one. The data contains 418 E. coli CRE records.
- **CRE does not apply to Pseudomonas or Acinetobacter.** They can be carbapenem-resistant, but it is not called CRE. Any such label is a data anomaly.

**Note the explicit sets.** A catch-all like `return "other bacteria"` would silently sweep in Chikungunya, Geotrichum, Scedosporium and Mycoplasma. Listing organisms explicitly makes the rule auditable.

## Cell 1.3 — Build the outcome

```python
elig = df[df["org_class"] != "INELIGIBLE"].copy()

# A phenotype counts ONLY if it is both observed AND applicable to that organism
for p in PHENOTYPES:
    elig["valid_" + p] = elig.apply(
        lambda r, p=p: int(r[p] == 1 and p in APPLICABLE[r["org_class"]]), axis=1)

elig["Y"] = elig[["valid_" + p for p in PHENOTYPES]].max(axis=1)

print("eligible isolates:", len(elig))
print("distinct samples :", elig[SID].nunique())
print("Y=1 (resistant)  :", elig["Y"].sum(), f"({100*elig['Y'].mean():.1f}%)")
```

**Expected:** `8162` · `7336` · `4491 (55.0%)`

- `.copy()` — avoids pandas' SettingWithCopyWarning when adding columns to a filtered frame.
- `lambda r, p=p:` — the `p=p` binds the current loop value; without it, all six columns would use the last `p`.
- `.max(axis=1)` — 1 if any valid phenotype is present.

**`Y = 0` means "no documented resistance phenotype", NOT "laboratory-confirmed susceptible."** A blank field is an absence of evidence. This must be stated as a limitation.

## Cell 1.4 — Data-quality anomalies

```python
# Runs on the WHOLE dataset, not just the eligible subset -
# eligibility decides who enters the analysis; the audit examines everything.
anomalies = []
for _, r in df.iterrows():
    applicable = set(APPLICABLE.get(r["org_class"], []))    # ineligible -> empty set
    for p in PHENOTYPES:
        if r[p] == 1 and p not in applicable:
            anomalies.append({"Sample ID": r[SID], "Organism": r[ORG],
                              "Class": r["org_class"], "Phenotype": p})

anomalies = pd.DataFrame(anomalies)
print("anomalous records:", len(anomalies))
print(anomalies.groupby(["Organism", "Phenotype"]).size())
```

**Expected:** `48` anomalous records across 11 organisms.

Examples: *E. coli* labelled VRE (2), *Pseudomonas* labelled CRE (6), *Acinetobacter* labelled CRE (14), *S. epidermidis* labelled MRSA (2 — should be MR-CoNS). These are biologically impossible combinations, so they are documented rather than counted toward `Y`.

## Cell 1.5 — The resistance phenotype profile table

```python
def organism_group(r):
    o = str(r[ORG]).upper(); c = r["org_class"]
    if "KLEBSIELLA PNEUMONIAE" in o: return "Klebsiella pneumoniae"
    if o == "E. COLI":               return "E. coli"
    if c == "Enterobacterales":      return "Other Enterobacterales"
    if "PSEUDOMONAS" in o:           return "Pseudomonas sp"
    if "ACINETOBACTER" in o:         return "Acinetobacter sp"
    if c == "Non-fermenter":         return "Other non-fermenters"
    if "FAECIUM" in o:               return "Enterococcus faecium"
    if c == "Enterococcus":          return "Other enterococci"
    if c == "S. aureus":             return "Staphylococcus aureus"
    return "Coagulase-negative staphylococci"

elig["organism_grp"] = elig.apply(organism_group, axis=1)

rows = []
for g, sub in elig.groupby("organism_grp"):
    app = APPLICABLE[sub["org_class"].iloc[0]]
    row = {"Organism group": g, "Isolates": len(sub)}
    for p in PHENOTYPES:
        row[p] = f"{100*sub['valid_'+p].mean():.1f}%" if p in app else "—"
    row["Any documented applicable resistance"] = f"{100*sub['Y'].mean():.1f}%"
    rows.append(row)

profile = pd.DataFrame(rows).sort_values("Isolates", ascending=False)
print(profile.to_string(index=False))
```

**The groups are deliberately homogeneous in applicability** — every organism within a group shares the same applicable phenotype set. If a group mixed enterococci and staphylococci, the "applicable" set would be a union and the dashes would become meaningless.

**Two symbols, two meanings:**
- `—` = phenotype **not applicable** to that organism class
- `0.0%` = phenotype **is applicable** but was not observed

**Expected highlights:** Klebsiella 67.7% CRE · E. coli 39.8% ESBL · Acinetobacter 79.8% MDR

---

# STEP 2 — Bivariate association (chi-square + Cramér's V)

**The question:** looking at one factor at a time, is it associated with resistance — and how strongly?

## Cell 2.1 — Build the predictors

```python
# Unit type: critical care vs general ward
CRIT = "ICU|ITU|HDU|CCU|NICU|PICU|GITU|SICU|HDI"
elig["unit_grp"] = np.where(
    elig[UNIT].astype(str).str.upper().str.contains(CRIT), "Critical care", "General ward")

# Infection origin
elig["hai_grp"] = np.where(
    elig[HAI].astype(str).str.strip().str.upper() == "HAI", "HAI", "Community/outside")

# Specimen: keep the 6 most common individually, group the rest
JUNK = {"OTHER / REVIEW", "(blank)", "OTHER / FLAGGED", ""}
ok = ~elig[SPEC].astype(str).str.strip().isin(JUNK)
top6 = elig.loc[ok, SPEC].value_counts().head(6).index
elig["specimen_grp"] = np.where(elig[SPEC].isin(top6), elig[SPEC].astype(str).str.title(), "Other")
elig.loc[~ok, "specimen_grp"] = np.nan       # 15 unusable records excluded from specimen analyses

for c in ["organism_grp", "specimen_grp", "unit_grp", "hai_grp"]:
    print(elig[c].value_counts(dropna=False), "\n")
```

*`np.where(condition, a, b)` is a vectorised if/else across the whole column.*

## Cell 2.2 — Chi-square with effect size

```python
def cramers_v(ct, chi2):
    """Effect size for a contingency table: 0 = no association, 1 = perfect."""
    n = ct.values.sum()
    return np.sqrt(chi2 / (n * (min(ct.shape) - 1)))

results = []
for label, col in [("Organism group","organism_grp"), ("Specimen type","specimen_grp"),
                   ("Hospital unit type","unit_grp"), ("HAI status","hai_grp")]:
    d  = elig.dropna(subset=[col])
    ct = pd.crosstab(d[col], d["Y"])                 # the contingency table
    chi2, p, dof, expected = stats.chi2_contingency(ct)
    results.append({"Variable": label, "n": ct.values.sum(), "chi2": round(chi2,2),
                    "df": dof, "p": p, "Cramers_V": round(cramers_v(ct,chi2),3),
                    "min_expected": round(expected.min(),1)})
    pct = (ct[1] / ct.sum(axis=1) * 100).round(1)
    print(f"\n=== {label} ===")
    print(pd.DataFrame({"n": ct.sum(axis=1), "resistant": ct[1], "%": pct})
            .sort_values("%", ascending=False).to_string())

S2 = pd.DataFrame(results)

# Holm-Bonferroni correction for running four tests
reject, p_adj, _, _ = multipletests(S2["p"], method="holm")
S2["p_holm"] = p_adj
S2["significant"] = np.where(reject, "Yes", "No")
print("\n", S2.to_string(index=False))
```

**How chi-square works:** it computes what each cell count *would* be if the two variables were unrelated (`expected = row total × column total ÷ grand total`), then sums `(observed − expected)² ÷ expected`. A large total means the data is far from "no association".

**Why `min_expected` is printed:** chi-square requires expected counts ≥5 in every cell. Below that, Fisher's exact test is needed instead. Here the minimum is 68.4, so chi-square is valid throughout.

**Why Cramér's V is essential:** with 8,162 isolates, almost anything reaches p<0.001. V measures *how strong* the association is (0.1 small, 0.3 moderate, 0.5 large), independent of sample size.

**Expected:**

| Variable | χ² | df | Cramér's V |
|---|---|---|---|
| Organism group | 1824.97 | 9 | **0.473** |
| Specimen type | 179.87 | 6 | 0.149 |
| HAI status | 65.33 | 1 | 0.089 |
| Hospital unit type | 15.61 | 1 | **0.044** |

Unit type is "highly significant" (p<0.001) with an effect size of 0.044 — statistically detectable, practically negligible. This is the clearest possible demonstration of why effect sizes must accompany p-values.

## Cell 2.3 — Clustering: ICC and design effect

```python
n = len(elig); k = elig[SID].nunique(); mbar = n / k

grp  = elig.groupby(SID)["Y"]
ni   = grp.size().values
yi   = grp.mean().values
ybar = elig["Y"].mean()

MSB = np.sum(ni * (yi - ybar)**2) / (k - 1)                                  # between clusters
MSW = np.sum([((s - s.mean())**2).sum() for _, s in grp]) / (n - k)          # within clusters
m0  = (n - np.sum(ni**2)/n) / (k - 1)

icc  = (MSB - MSW) / (MSB + (m0 - 1) * MSW)
deff = 1 + (mbar - 1) * icc

print(f"average cluster size m̄ = {mbar:.4f}")
print(f"ICC                    = {icc:.4f}")
print(f"design effect          = {deff:.4f}")
print(f"effective sample size  = {n/deff:.0f}")
```

**Expected:** m̄=1.113 · ICC=0.184 · DEFF=1.021 · effective n≈7,997

**The problem this quantifies:** chi-square assumes independent observations, but 8,162 isolates come from only 7,336 samples — isolates sharing a sample are correlated.

**The design effect is NOT the average cluster size.** `DEFF = 1 + (m̄ − 1) × ICC` requires the intraclass correlation. Here the ICC is substantial (0.18) but few samples are multi-isolate, so the overall penalty is ~2%.

## Cell 2.4 — Sensitivity analysis

```python
tests = [("Organism group","organism_grp"), ("Specimen type","specimen_grp"),
         ("Hospital unit type","unit_grp"), ("HAI status","hai_grp")]

sens = {lab: [] for lab, _ in tests}
for seed in range(200):
    one = elig.sample(frac=1, random_state=seed).drop_duplicates(SID)   # 1 isolate per sample
    for lab, col in tests:
        d  = one.dropna(subset=[col])
        ct = pd.crosstab(d[col], d["Y"])
        chi2, p, _, _ = stats.chi2_contingency(ct)
        sens[lab].append((cramers_v(ct, chi2), p))

for lab, _ in tests:
    vs = [v for v, _ in sens[lab]]; ps = [p for _, p in sens[lab]]
    print(f"{lab:20s} mean V = {np.mean(vs):.3f} | significant in {sum(p<0.05 for p in ps)}/200")
```

**Logic:** shuffle, keep one isolate per sample (a genuinely independent dataset), recompute, repeat 200 times.

**Expected:** V values essentially unchanged (0.473 / 0.148 / 0.048 / 0.094), significant in 200/200.

**Correct interpretation:** this does **not** prove the dependence is immaterial. It shows the substantive conclusions are **robust** to it. Keep the full dataset as the primary analysis — discarding organisms from polymicrobial samples would throw away real information.

## Cell 2.5 — Step 2 figures

```python
TEAL, SLATE, MUTED = "#0F766E", "#334155", "#64748B"

def resistance_bar(col, color, fname_title):
    """Horizontal bar of % resistant by category (Figures 5.1 and 5.2)."""
    d  = elig.dropna(subset=[col])
    ct = pd.crosstab(d[col], d["Y"])
    tab = pd.DataFrame({"n": ct.sum(axis=1), "pct": 100*ct[1]/ct.sum(axis=1)}).sort_values("pct")
    fig, ax = plt.subplots(figsize=(7.4, 4.6))
    ax.barh(range(len(tab)), tab["pct"], color=color, height=0.72)
    ax.set_yticks(range(len(tab))); ax.set_yticklabels(tab.index, fontsize=9.5)
    for i, (pv, nv) in enumerate(zip(tab["pct"], tab["n"])):
        ax.text(pv+1, i, f"{pv:.1f}%  (n={nv:,})", va="center", fontsize=8.6, color=SLATE)
    ax.set_xlabel("% with documented applicable resistance phenotype", color=SLATE)
    ax.set_xlim(0, 100); ax.spines[["top","right"]].set_visible(False)
    ax.set_title(fname_title, fontsize=10, color=SLATE)
    plt.tight_layout(); plt.show()

# Figure 5.1 and Figure 5.2
resistance_bar("organism_grp", TEAL,      "Resistance by organism group")
resistance_bar("specimen_grp", "#14B8A6", "Resistance by specimen type")

# Figure 5.3 - effect size comparison (the key teaching figure)
labels = S2["Variable"].tolist(); vals = S2["Cramers_V"].tolist()
colors = [TEAL if v >= 0.3 else "#14B8A6" if v >= 0.1 else "#CBD5E1" for v in vals]

fig, ax = plt.subplots(figsize=(7, 3.4))
ax.bar(labels, vals, color=colors, width=0.58)
for i, v in enumerate(vals):
    ax.text(i, v+0.012, f"{v:.3f}", ha="center", fontsize=10, fontweight="bold", color=SLATE)
for y, name in [(0.1, "small"), (0.3, "moderate")]:
    ax.axhline(y, ls="--", lw=0.9, color=MUTED)
    ax.text(len(labels)-0.42, y+0.008, name, fontsize=8, color=MUTED, ha="right")
ax.set_ylabel("Cramér's V (effect size)", color=SLATE); ax.set_ylim(0, 0.55)
ax.spines[["top","right"]].set_visible(False)
ax.set_title("All four associations are significant (p<0.001);\nonly organism shows a substantial effect size",
             fontsize=10, color=SLATE, pad=10)
plt.tight_layout(); plt.show()
```

*Figure 5.3 is the one worth studying. The dashed guides at 0.1 and 0.3 make the point visually: every bar is statistically significant, but only organism clears the "moderate" line.*

---

# STEP 3 — Multivariable logistic regression

**The question:** when all factors are considered together, which remain independently associated with resistance — and how well can the model estimate resistance probability for unseen samples?

One model, two uses: **inference** (odds ratios) and **prediction** (probabilities).

## Cell 3.1 — Prepare and check

```python
d = elig.dropna(subset=["specimen_grp"]).rename(columns={SID: "sample_id"}).copy()
print("modelling n =", len(d), "| samples =", d["sample_id"].nunique())

# (a) SEPARATION: no category may be 0% or 100% resistant
for c in ["organism_grp", "specimen_grp", "unit_grp", "hai_grp"]:
    t = d.groupby(c)["Y"].agg(n="size", events="sum")
    t["pct"] = (100 * t.events / t.n).round(1)
    bad = t[(t.events == 0) | (t.events == t.n)]
    print(f"\n{c}: separation ->", "NONE" if len(bad) == 0 else list(bad.index))
    print(t.sort_values("pct", ascending=False).to_string())

# (b) EVENTS PER PARAMETER: need >= 10
n_params = (d.organism_grp.nunique()-1) + (d.specimen_grp.nunique()-1) + 1 + 1
print(f"\nparameters={n_params}  events={d.Y.sum()}  EPV={d.Y.sum()/n_params:.0f}")

# (c) MULTICOLLINEARITY: VIF should be < 5
X = pd.get_dummies(d[["organism_grp","specimen_grp","unit_grp","hai_grp"]], drop_first=True).astype(float)
X = sm.add_constant(X)
vif = pd.DataFrame({"variable": X.columns,
                    "VIF": [variance_inflation_factor(X.values, i) for i in range(X.shape[1])]})
print("\nmax VIF =", round(vif[vif.variable != "const"].VIF.max(), 2))
```

**Expected:** no separation · EPV=264 · max VIF=4.77

**Why each check matters**
- **Separation** — a category that is 0% or 100% resistant makes the coefficient infinite and the model won't converge.
- **Events per parameter** — fewer than ~10 events per parameter gives unstable, over-fitted estimates.
- **VIF** — if predictors overlap heavily, coefficients become unreliable. Above 5 is a concern.

## Cell 3.2 — Fit with cluster-robust standard errors

```python
formula = ("Y ~ C(organism_grp, Treatment(reference='E. coli'))"
           " + C(specimen_grp, Treatment(reference='Urine'))"
           " + C(unit_grp,     Treatment(reference='General ward'))"
           " + C(hai_grp,      Treatment(reference='Community/outside'))")

model_naive = smf.logit(formula, data=d).fit(disp=0)

model = smf.logit(formula, data=d).fit(
    disp=0,
    cov_type="cluster",
    cov_kwds={"groups": d["sample_id"]})        # standard errors clustered by sample

print(model.summary())
print("\npseudo-R2 =", round(model_naive.prsquared, 4))
```

**Reading the formula**
- `C(...)` marks a variable as categorical.
- `Treatment(reference='E. coli')` sets the baseline; every odds ratio is *relative to that category*, holding the others constant.
- References were chosen for interpretability and stability, not arbitrarily.

**Why logistic and not linear:** the outcome is binary. Linear regression could predict 1.3 or −0.2, which is meaningless for a yes/no. Logistic models the log-odds, keeping fitted probabilities inside [0,1].

**Why cluster-robust:** two isolates from one sample are correlated. `cov_type="cluster"` widens the standard errors appropriately instead of treating them as independent. `disp=0` just suppresses the optimiser log.

## Cell 3.3 — Odds ratios

```python
ci = model.conf_int()
OR = pd.DataFrame({
    "OR":      np.exp(model.params),
    "CI_low":  np.exp(ci[0]),
    "CI_high": np.exp(ci[1]),
    "p":       model.pvalues,
}).drop("Intercept").round(3)
print(OR.to_string())
```

**`np.exp()` is the key step.** Logistic regression returns coefficients on the log-odds scale; exponentiating converts them to **odds ratios**, which are interpretable.

- **OR > 1** → higher odds than the reference
- **OR < 1** → lower odds
- **CI excludes 1** → statistically significant

**Expected:** Acinetobacter 2.12 (1.68–2.68) · Klebsiella 2.09 (1.83–2.39) · HAI 1.89 (1.69–2.12) · Critical care 1.24 (1.11–1.39)

> Use "**associated with**", never "causes". This is retrospective observational data.

## Cell 3.4 — Prediction with grouped splitting

```python
splitter = GroupShuffleSplit(n_splits=1, test_size=0.25, random_state=42)
train_i, test_i = next(splitter.split(d, d["Y"], groups=d["sample_id"]))
train, test = d.iloc[train_i].copy(), d.iloc[test_i].copy()

print("train:", len(train), "isolates from", train.sample_id.nunique(), "samples")
print("test :", len(test),  "isolates from", test.sample_id.nunique(),  "samples")
print("overlapping samples:", len(set(train.sample_id) & set(test.sample_id)), "(must be 0)")

model_train = smf.logit(formula, data=train).fit(disp=0)
test["p"] = model_train.predict(test)

auc   = roc_auc_score(test["Y"], test["p"])
brier = brier_score_loss(test["Y"], test["p"])
print(f"\nAUC   = {auc:.4f}")
print(f"Brier = {brier:.4f}   (0.25 = uninformative at this prevalence)")
```

**`GroupShuffleSplit` rather than `train_test_split` is essential.** With an ordinary split, *E. coli* from sample 101 could land in training while *Klebsiella* from sample 101 lands in testing — the model would have already seen part of that specimen. That is **information leakage**, and it inflates performance. Grouping by `sample_id` keeps every sample wholly on one side.

**Expected:** train 6,115 / test 2,032 · overlap 0 · **AUC 0.778** · Brier 0.190

- **AUC** — probability the model ranks a random resistant isolate above a random non-resistant one. 0.5 = chance, 1.0 = perfect.
- **Brier** — mean squared error of the predicted probabilities. Lower is better.

## Cell 3.5 — Thresholds and calibration

```python
for th in [0.3, 0.4, 0.5, 0.6, 0.7]:
    pred = (test["p"] >= th).astype(int)
    tn, fp, fn, tp = confusion_matrix(test["Y"], pred).ravel()
    print(f"threshold {th}: sensitivity={tp/(tp+fn):.3f}  specificity={tn/(tn+fp):.3f}  "
          f"PPV={tp/(tp+fp):.3f}  accuracy={(tp+tn)/len(test):.3f}")

test["decile"] = pd.qcut(test["p"], 10, labels=False, duplicates="drop")
calib = test.groupby("decile").agg(n=("Y","size"), predicted=("p","mean"), observed=("Y","mean")).round(3)
print("\n", calib.to_string())
```

**Threshold 0.5 is a convention, not a clinical optimum.** The right cut-off depends on the relative cost of missing a resistant isolate versus over-calling one.

**Calibration** asks a different question from AUC: *when the model says 70%, does resistance actually occur about 70% of the time?* Predicted and observed should track closely — they do here.

**Expected at 0.5:** sensitivity 0.816 · specificity 0.600 · accuracy 0.719


## Cell 3.5b — ROC and calibration figures

```python
# Figure 5.5 - ROC curve
fpr, tpr, thresholds = roc_curve(test["Y"], test["p"])

fig, ax = plt.subplots(figsize=(4.8, 4.4))
ax.plot(fpr, tpr, color="#0F766E", lw=2.4, label=f"Model (AUC = {auc:.3f})")
ax.plot([0, 1], [0, 1], ls="--", color="#64748B", lw=1, label="Chance (AUC = 0.500)")
ax.set_xlabel("1 − specificity"); ax.set_ylabel("Sensitivity")
ax.legend(fontsize=9, frameon=False, loc="lower right")
ax.spines[["top","right"]].set_visible(False)
plt.tight_layout(); plt.show()

# Figure 5.6 - calibration plot
fig, ax = plt.subplots(figsize=(4.8, 4.4))
ax.plot([0, 1], [0, 1], ls="--", color="#64748B", lw=1, label="Perfect calibration")
ax.plot(calib["predicted"], calib["observed"], "o-", color="#0F766E", lw=2, ms=6, label="Observed")
ax.set_xlabel("Mean predicted probability"); ax.set_ylabel("Observed proportion resistant")
ax.set_xlim(0, 1); ax.set_ylim(0, 1)
ax.legend(fontsize=9, frameon=False, loc="upper left")
ax.spines[["top","right"]].set_visible(False)
plt.tight_layout(); plt.show()
```

**The two figures answer different questions.** The ROC curve shows *discrimination* — can the model rank resistant above non-resistant? The calibration plot shows whether the probabilities are *honest* — when it says 70%, does resistance occur about 70% of the time? A model can discriminate well yet be badly calibrated, so both are reported.

*`roc_curve` returns the false-positive and true-positive rates at every possible threshold; plotting one against the other traces the curve, and AUC is the area beneath it.*

## Cell 3.6 — Grouped cross-validation

```python
aucs, briers = [], []
for tr_idx, te_idx in GroupKFold(n_splits=5).split(d, d["Y"], groups=d["sample_id"]):
    m  = smf.logit(formula, data=d.iloc[tr_idx]).fit(disp=0)
    pp = m.predict(d.iloc[te_idx])
    aucs.append(roc_auc_score(d.iloc[te_idx]["Y"], pp))
    briers.append(brier_score_loss(d.iloc[te_idx]["Y"], pp))

print("AUC per fold:", np.round(aucs, 4))
print(f"mean AUC = {np.mean(aucs):.4f} (SD {np.std(aucs):.4f})")
```

**`GroupKFold`, not `StratifiedKFold`** — same reasoning as the split. Five folds each act as a test set once, so the result doesn't hinge on one lucky partition.

**Expected:** mean AUC 0.7813 (SD 0.0088) — the small SD indicates stable performance.

## Cell 3.7 — Forest plot

```python
R2 = OR.copy()
R2["label"] = [i.split("[T.")[-1].rstrip("]") for i in R2.index]
R2 = R2.sort_values("OR")

fig, ax = plt.subplots(figsize=(7.5, 6.5))
y = np.arange(len(R2))[::-1]
colors = ["#0F766E" if v > 1 else "#BE123C" for v in R2["OR"]]
ax.hlines(y, R2["CI_low"], R2["CI_high"], color=colors, lw=2)
ax.scatter(R2["OR"], y, color=colors, s=40, zorder=3)
ax.axvline(1, ls="--", color="grey")
ax.set_yticks(y); ax.set_yticklabels(R2["label"], fontsize=9)
ax.set_xscale("log")
ax.set_xlabel("Adjusted odds ratio (log scale), 95% CI")
plt.tight_layout(); plt.show()
```

*The log scale matters: it makes OR = 0.5 and OR = 2 appear equally far from 1, which is correct since they are reciprocal effects.*

---

# STEP 4 — Statistical process control (p-chart)

**The question:** did the HAI burden genuinely change over the 14 months, or is the month-to-month movement just noise?

## Cell 4.1 — Aggregate to sample level by month

```python
samp = df.groupby(SID).agg(month=(MONTH, "first"), hai=(HAI, "first")).reset_index()
samp["hai"] = (samp["hai"].astype(str).str.strip().str.upper() == "HAI").astype(int)

MONTHS = ["APRIL","MAY","JUNE","JULY","AUGUST","SEPTEMBER","OCTOBER","NOVEMBER","DECEMBER",
          "JANUARY 2026","FEBRUARY 2026","MARCH 2026","APRIL 2026","MAY 2026"]
LABELS = ["Apr-25","May-25","Jun-25","Jul-25","Aug-25","Sep-25","Oct-25","Nov-25","Dec-25",
          "Jan-26","Feb-26","Mar-26","Apr-26","May-26"]

C = pd.DataFrame([{"Month": lab,
                   "n":   len(samp[samp.month == m]),
                   "HAI": int(samp[samp.month == m].hai.sum())}
                  for m, lab in zip(MONTHS, LABELS)])
C["p"] = C["HAI"] / C["n"]
print(C.to_string(index=False))
```

**This step is sample-level, not isolate-level** — the question concerns the infection burden, so each sample counts once. Cell 0.5 proved HAI is constant within a sample, so `"first"` is exact.

**All 10,943 samples are used**, not just the bacterial subset, since this is about overall burden.

> **What this statistic is:** the monthly proportion of *culture-positive samples* classified as HAI.
> **What it is not:** a hospital-wide HAI incidence rate — that would need patient-days, which the register lacks.

## Cell 4.2 — Control limits (month-specific)

```python
pbar = C["HAI"].sum() / C["n"].sum()          # centre line

C["CL"]  = pbar
C["SE"]  = np.sqrt(pbar * (1 - pbar) / C["n"])       # depends on THAT month's n
C["UCL"] = (pbar + 3 * C["SE"]).clip(upper=1)
C["LCL"] = (pbar - 3 * C["SE"]).clip(lower=0)

C["Signal"] = np.where(C["p"] > C["UCL"], "Above UCL",
              np.where(C["p"] < C["LCL"], "Below LCL", ""))

print(f"centre line = {100*pbar:.1f}%")
print(f"monthly n ranges {C['n'].min()}-{C['n'].max()} -> limits VARY by month")
print(C[["Month","n","HAI","p","LCL","UCL","Signal"]].round(3).to_string(index=False))
```

**The limits must be computed per month.** The standard error is `√(p̄(1−p̄)/nₜ)` — it depends on that month's sample count. Since n ranges 721–854, drawing a single flat pair of lines would be wrong. A month with fewer samples gets **wider** limits, correctly reflecting greater uncertainty.

**Why 3 sigma:** in a stable process ~99.7% of points fall within ±3 SE, so a point outside is unlikely to be chance — it signals a real change.

**Expected:** centre line 46.8% · **7 of 14 months outside limits**

## Cell 4.3 — Runs rule

```python
side = np.where(C["p"] > pbar, 1, -1)
runs, current, length = [], side[0], 1
for s in side[1:]:
    if s == current:
        length += 1
    else:
        runs.append((current, length)); current, length = s, 1
runs.append((current, length))

print("runs:", [("above" if s > 0 else "below", l) for s, l in runs])
print("special-cause run (>=8):", any(l >= 8 for _, l in runs))
```

**A second kind of signal.** Even if every point sits inside the limits, **8+ consecutive points on one side of the centre line** is very unlikely by chance and indicates a sustained shift.

**Expected:** a run of **9 months above** the centre line (Jun-25 → Feb-26).

## Cell 4.4 — Trend test

```python
x = np.arange(len(C), dtype=float)
k = C["HAI"].values.astype(float)
n = C["n"].values.astype(float)

N, K = n.sum(), k.sum()
pb = K / N
T   = np.sum(x * (k - n * pb))
var = pb * (1 - pb) * (np.sum(n * x**2) - (np.sum(n * x))**2 / N)
Z   = T / np.sqrt(var)
p   = 2 * (1 - stats.norm.cdf(abs(Z)))

print(f"Cochran-Armitage trend: Z = {Z:.3f}, p = {p:.4f}")
```

**Cochran–Armitage tests for a *monotonic trend* across ordered groups**, which ordinary chi-square cannot do (it treats categories as unordered). Negative Z = declining.

**Expected:** Z = −3.232, p = 0.0012 — a significant declining trend.

> **Why no ARIMA or forecasting:** 14 monthly points is far too few. ARIMA needs ~50, seasonal decomposition needs 24+. Trend testing plus SPC is the appropriate depth, and saying so demonstrates judgement.

## Cell 4.5 — The chart

```python
x = np.arange(len(C))
fig, ax = plt.subplots(figsize=(8, 4.8))

ax.step(x, 100*C["UCL"], where="mid", color="#BE123C", ls="--", lw=1.3, label="Control limits (3σ)")
ax.step(x, 100*C["LCL"], where="mid", color="#BE123C", ls="--", lw=1.3)
ax.axhline(100*pbar, color="#0F766E", lw=1.6, label=f"Centre line ({100*pbar:.1f}%)")

ax.plot(x, 100*C["p"], color="#334155", lw=1.6, zorder=2)
inside = C["Signal"] == ""
ax.scatter(x[inside],  100*C["p"][inside],  color="#334155", s=45, zorder=3, label="In control")
ax.scatter(x[~inside], 100*C["p"][~inside], color="#BE123C", s=95, zorder=4,
           edgecolor="white", linewidth=1.2, label="Outside limits")

ax.set_xticks(x); ax.set_xticklabels(C["Month"], rotation=45, ha="right")
ax.set_ylabel("% of culture-positive samples classified as HAI")
ax.legend(fontsize=8.5, frameon=False)
plt.tight_layout(); plt.show()
```

**`ax.step(..., where="mid")`** draws the limits as a stepped band rather than a smooth line — visually correct, since each month has its own limit.

---

# Summary of decisions

Everything above follows from a small number of choices. These are the ones to be able to defend:

| Decision | Reason |
|---|---|
| Organism-level rows, not collapsed to sample | Collapsing with "first organism + max resistance" mismatched organism and phenotype in **109 samples** |
| `Sample Type (category)` not `(cleaned)` | The cleaned column has 1,327 granular values ("BLOOD FROM RIGHT HAND") |
| MSSA excluded from resistance | It is methicillin-**sensitive** |
| `\b` word boundaries in the regex | Prevents "MRSA" matching inside "MRCONS" |
| Eligibility decided a priori on biology | Excluding on observed zero events would be selecting on the outcome |
| CRE applies to E. coli, not to Pseudomonas | CRE = carbapenem-resistant **Enterobacterales** |
| `—` vs `0%` in the profile | Not applicable vs applicable-but-not-observed |
| Anomalies documented, not counted | 48 biologically impossible organism–phenotype combinations |
| Cramér's V with every p-value | With n = 8,162 nearly everything is "significant" |
| Cluster-robust standard errors | Isolates from one sample are correlated (ICC = 0.18) |
| `GroupShuffleSplit` / `GroupKFold` | Prevents leakage of the same sample across train and test |
| Month-specific control limits | Monthly denominators vary (721–854) |
| No ARIMA / forecasting | Only 14 time points |

## Known limitations to state in the report

1. `Y = 0` means no *documented* phenotype — not confirmed susceptibility.
2. Pseudo-R² = 0.197: the register lacks patient-level covariates (prior antibiotics, devices, length of stay, comorbidity).
3. Retrospective observational data — associations only, never causation.
4. The SPC statistic is a proportion of culture-positive samples, not a true incidence rate.
5. The final months may be incomplete owing to late-arriving results.
6. The model is internally validated only; external validation would be required before clinical use.