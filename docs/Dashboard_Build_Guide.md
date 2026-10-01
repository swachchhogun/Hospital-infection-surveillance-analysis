# Power BI Dashboard — Complete Build Guide

**Infection & Antimicrobial Resistance Surveillance**
Data file: `PowerBI_Data.xlsx` · 12,188 rows · 25 columns

---

## Contents

1. [The one thing that will break everything](#part-1)
2. [Importing the data](#part-2)
3. [Checking the model](#part-3)
4. [Creating the measures](#part-4)
5. [Setting the theme](#part-5)
6. [Page 1 — Overview](#page-1)
7. [Page 2 — Organisms](#page-2)
8. [Page 3 — Antimicrobial resistance](#page-3)
9. [Page 4 — Hospital units](#page-4)
10. [Page 5 — Monthly surveillance](#page-5)
11. [Slicers and cross-page filtering](#part-11)
12. [Final polish](#part-12)
13. [Verification checklist](#part-13)
14. [Troubleshooting](#part-14)
15. [Writing it up in Chapter 6](#part-15)

---

<a name="part-1"></a>
## 1. The one thing that will break everything

Your data has **one row per organism**, not one row per sample.

- 12,188 rows (isolates)
- 10,943 samples
- The difference: 1,073 samples grew more than one organism

**So if you drag `Sample ID` onto a card and Power BI shows "Count of Sample ID" = 12,188, that is wrong.** Every sample-level figure must use **distinct count**.

| To count | Use | Correct answer |
|---|---|---|
| Samples | `DISTINCTCOUNT([Sample ID])` | 10,943 |
| Isolates | `COUNTROWS()` | 12,188 |
| Patients | `DISTINCTCOUNT([Patient ID])` | 5,311 |

The measures in Part 4 handle this. Use them rather than dragging raw fields onto visuals.

**The second rule:** every resistance visual must be filtered to `AMR eligible = "Yes"`. The six phenotypes (CRE, ESBL, MDR, VRE, MRSA, MR-CoNS) are bacterial categories — a *Candida* isolate has no CRE result because the test doesn't exist for fungi, not because it was sensitive. Including them would inflate your denominator and understate resistance. The measures handle this too.

---

<a name="part-2"></a>
## 2. Importing the data

### Step 2.1 — Turn off auto-detection first

Do this **before** importing. It is the root fix for the type errors.

> **File → Options and settings → Options → Current File → Data Load**
> Uncheck **"Automatically detect column types and headers for unstructured sources"**
> Click **OK**

*Why: Power BI samples the first 1,000 rows to guess each column's type. `Patient ID` starts with numeric values then switches to text (`AMHL.0002610706`), so the guess fails and throws errors on row 4,357 onward.*

### Step 2.2 — If you already imported the CSV, remove it

In the **Fields** pane on the right, right-click the old `PowerBI_Data` table → **Delete from model**. The bad type conversions are baked into that query and cannot be repaired cleanly.

### Step 2.3 — Import

> **Home → Get data → Excel workbook**
> Select `PowerBI_Data.xlsx`
> Tick the **`Data`** sheet
> Click **Load** (not Transform Data — no transformation is needed)

### Step 2.4 — Confirm the types

Click the table in the Fields pane, then **Table tools → Manage relationships** is not needed (single table). Instead check a few columns in **Data view** (the grid icon on the left):

| Column | Should be |
|---|---|
| `Sample ID`, `Patient ID`, `Unit`, `Specimen`, `Organism` | Text |
| `Month label` | Text (shows `2025-04 (Apr)`) |
| `Month order`, `Resistant flag`, `CRE`…`MSSA` | Whole number |

To change one: click the column → **Column tools → Data type**.

---

<a name="part-3"></a>
## 3. Checking the model

Before building anything, confirm the data loaded correctly.

> **Data view** (grid icon, left sidebar) → look at the row count at the bottom: **12,188 rows**

Then check there are no errors: the **Query Errors** folder that appeared last time should be gone.

**The columns you will use most:**

| Column | Values | Purpose |
|---|---|---|
| `Sample ID` | 10,943 unique | Distinct counts |
| `Patient ID` | 5,311 unique | Patient counts |
| `Month label` | `2025-04 (Apr)` … `2026-05 (May)` | Trend axis (sorts itself) |
| `Unit` | 51 wards | Detail breakdown |
| `Unit type` | Critical care / General ward | High-level split |
| `Specimen` | 32 categories | Breakdown, slicer |
| `Organism` | 136 organisms | Detail table |
| `Organism group` | 10 groups | Main organism analysis |
| `Organism kingdom` | Bacteria, Fungus, Virus, Mycobacteria, Parasite | Overview donut |
| `AMR eligible` | Yes / No | **Filter for resistance** |
| `Resistant` | Resistant / No documented phenotype | Legend, table |
| `Resistant flag` | 1 / 0 | Rate calculations |
| `Infection origin` | Healthcare-associated / Community | HAI analysis |
| `CRE`…`MRCONS` | 1 / 0 | Phenotype counts |

---

<a name="part-4"></a>
## 4. Creating the measures

Create each one via **Modeling → New measure**, paste the code, press Enter.

Put them in a tidy folder: select all measures in the Fields pane → in Properties set **Display folder** = `Measures`.

### 4.1 — Core counts

```dax
Samples = DISTINCTCOUNT('Data'[Sample ID])
```

```dax
Isolates = COUNTROWS('Data')
```

```dax
Patients = DISTINCTCOUNT('Data'[Patient ID])
```

```dax
Multi-organism Samples =
CALCULATE(
    [Samples],
    'Data'[Multi-organism] = "Yes"
)
```

### 4.2 — Healthcare-associated infection

```dax
HAI Samples =
CALCULATE(
    [Samples],
    'Data'[Infection origin] = "Healthcare-associated"
)
```

```dax
HAI Rate = DIVIDE([HAI Samples], [Samples])
```

> Format this as **Percentage, 1 decimal**: select the measure → **Measure tools → Format → Percentage → 1**.
>
> `DIVIDE` is used rather than `/` because it returns blank instead of an error when the denominator is zero — which happens the moment a slicer filters everything out.

### 4.3 — Resistance

```dax
Eligible Isolates =
CALCULATE(
    [Isolates],
    'Data'[AMR eligible] = "Yes"
)
```

```dax
Resistant Isolates =
CALCULATE(
    SUM('Data'[Resistant flag]),
    'Data'[AMR eligible] = "Yes"
)
```

```dax
Resistance Rate = DIVIDE([Resistant Isolates], [Eligible Isolates])
```

*Format as Percentage, 1 decimal.*

### 4.4 — Phenotype counts

Each is restricted to the organism class the phenotype applies to, so a filter can never produce something meaningless like "MRSA in Klebsiella".

```dax
CRE Isolates =
CALCULATE(SUM('Data'[CRE]), 'Data'[Organism class] = "Enterobacterales")
```

```dax
ESBL Isolates =
CALCULATE(SUM('Data'[ESBL]), 'Data'[Organism class] = "Enterobacterales")
```

```dax
MDR Isolates =
CALCULATE(SUM('Data'[MDR]), 'Data'[AMR eligible] = "Yes")
```

```dax
VRE Isolates =
CALCULATE(SUM('Data'[VRE]), 'Data'[Organism class] = "Enterococcus")
```

```dax
MRSA Isolates =
CALCULATE(SUM('Data'[MRSA]), 'Data'[Organism class] = "S. aureus")
```

```dax
MRCoNS Isolates =
CALCULATE(SUM('Data'[MRCONS]), 'Data'[Organism class] = "CoNS")
```

### 4.5 — Reference lines for the trend charts

```dax
HAI Rate Overall =
CALCULATE([HAI Rate], REMOVEFILTERS('Data'[Month label]))
```

```dax
Resistance Rate Overall =
CALCULATE([Resistance Rate], REMOVEFILTERS('Data'[Month label]))
```

*`REMOVEFILTERS` on the month means these always return the whole-period average (46.8% and 55.0%), so when plotted alongside the monthly line they draw as a flat reference line.*

### 4.6 — Dynamic title (optional but impressive)

```dax
Page Title =
"Infection Surveillance  |  " &
FORMAT([Samples], "#,0") & " samples  |  " &
IF(
    ISFILTERED('Data'[Unit type]),
    SELECTEDVALUE('Data'[Unit type], "All units"),
    "All units"
)
```

To use it: insert a **Card** visual, or select a visual's Title → click the **fx** button → Format by **Field value** → choose `Page Title`. The heading then updates as users filter.

---

<a name="part-5"></a>
## 5. Setting the theme

> **View → Themes → Customize current theme**

| Setting | Value |
|---|---|
| Theme colour 1 | `#0F766E` (teal — primary) |
| Theme colour 2 | `#14B8A6` (light teal — secondary) |
| Theme colour 3 | `#BE123C` (red — **resistance and alerts only**) |
| Theme colour 4 | `#334155` (slate — text/neutral) |
| Theme colour 5 | `#CBD5E1` (grey — context bars) |
| Theme colour 6 | `#B45309` (amber — warnings) |

Under **Text**, set the font to **Segoe UI**, general text size 10.

**The colour discipline that matters:** red means resistance or a warning, nothing else. If everything is red, nothing stands out.

---

<a name="page-1"></a>
## 6. PAGE 1 — Overview

**Question it answers:** what is the overall picture?
**Audience:** anyone opening the dashboard for the first time.

Rename the page (right-click the tab → Rename) to **Overview**.

### 6.1 — Header

> **Insert → Text box** → type `Infection & Antimicrobial Resistance Surveillance`
> Font 20pt, bold, colour `#0F766E`. Place across the top.
>
> Second, smaller text box beneath: `April 2025 – May 2026 · 14-month culture-positive register`, 11pt, grey.

### 6.2 — KPI cards (top row, five across)

For each: **Insert → Card**, then drag the measure into **Fields**.

| Card | Measure | Should read |
|---|---|---|
| 1 | `Samples` | 10,943 |
| 2 | `Patients` | 5,311 |
| 3 | `Isolates` | 12,188 |
| 4 | `HAI Rate` | 46.8% |
| 5 | `Resistance Rate` | 55.0% |

**Format each card** (paint-roller icon):
- **Callout value** → font size 28, colour `#0F766E` (use `#BE123C` for the Resistance Rate card)
- **Category label** → On, size 10
- **Effects → Background** → `#F0FDFA`; **Visual border** → On, rounded corners 8

### 6.3 — Monthly HAI trend (middle left)

> **Insert → Line chart**
> **X-axis:** `Month label`
> **Y-axis:** `HAI Rate`, then also add `HAI Rate Overall`

Format:
- **Lines** → for `HAI Rate`: width 3, colour `#0F766E`, markers On
- **Lines** → for `HAI Rate Overall`: width 2, colour `#CBD5E1`, dashed
- **Y-axis** → Start 0.30, End 0.65 (so the variation is visible rather than squashed)
- **Title:** `Monthly HAI rate against the 14-month average`

### 6.4 — Organism kingdom donut (middle right)

> **Insert → Donut chart**
> **Legend:** `Organism kingdom`
> **Values:** `Isolates`

Format: **Detail labels** → Label contents = *Category, percent of total*. Inner radius 60%.
**Title:** `Isolates by microbiological category`

### 6.5 — Leading organisms (bottom left)

> **Insert → Clustered bar chart**
> **Y-axis:** `Organism`
> **X-axis:** `Samples`
> **Filter:** Filters pane → `Organism` → Top N → Top **10** by `Samples`

Format: **Data labels** On. Sort descending by `Samples` (click the ⋯ menu → Sort axis).
**Title:** `Ten most frequently isolated organisms`

### 6.6 — Specimen distribution (bottom right)

> **Insert → Clustered bar chart**
> **Y-axis:** `Specimen`
> **X-axis:** `Samples`
> **Filter:** Top 10 by `Samples`

**Title:** `Culture-positive samples by specimen type`

---

<a name="page-2"></a>
## 7. PAGE 2 — Organisms

**Question:** what are we isolating, and which organisms carry the resistance?

### 7.1 — Resistance by organism group *(the headline visual)*

> **Insert → Clustered bar chart**
> **Y-axis:** `Organism group`
> **X-axis:** `Resistance Rate`
> Sort descending

Format:
- **Bars** → colour `#BE123C`
- **Data labels** → On
- **X-axis** → format as percentage

**Title:** `Documented resistance is highest in Acinetobacter and Klebsiella`

*Expected: Acinetobacter ≈ 79.8%, Klebsiella ≈ 77.8%, down to the other enterococci at ≈ 1.4%.*

### 7.2 — Volume by organism group

> **Insert → Clustered bar chart**
> **Y-axis:** `Organism group` · **X-axis:** `Eligible Isolates`
> Bars in `#0F766E`

*Placing this beside the rate chart lets viewers weigh a high rate against how many isolates it rests on.*

### 7.3 — Organism × phenotype matrix

> **Insert → Matrix**
> **Rows:** `Organism group`
> **Values:** `Eligible Isolates`, `Resistance Rate`, `CRE Isolates`, `ESBL Isolates`, `MDR Isolates`, `VRE Isolates`, `MRSA Isolates`

Format:
- **Cell elements** → select `Resistance Rate` → **Background colour** On → gradient, lowest `#FFFFFF`, highest `#FECDD3`
- **Specific column** → Values → font size 10

*The conditional formatting makes the high-resistance organisms visible without anyone reading a number.*

### 7.4 — Detailed organism table

> **Insert → Table**
> **Columns:** `Organism`, `Samples`, `Eligible Isolates`, `Resistance Rate`
> **Filter:** Top 20 by `Samples`

---

<a name="page-3"></a>
## 8. PAGE 3 — Antimicrobial resistance

**Question:** which resistance mechanisms are present, and where?

### 8.1 — Phenotype KPI cards (top row, six across)

| Card | Measure | Should read |
|---|---|---|
| CRE | `CRE Isolates` | 2,120 |
| ESBL | `ESBL Isolates` | 1,125 |
| MDR | `MDR Isolates` | 890 |
| VRE | `VRE Isolates` | 217 |
| MRSA | `MRSA Isolates` | 149 |
| MR-CoNS | `MRCoNS Isolates` | 10 |

> These are slightly lower than the raw counts in the resistance column, because each measure is restricted to the organism class the phenotype applies to. The difference is the 48 biologically incompatible labels identified in the data-quality audit (for example *Pseudomonas* recorded as CRE), which are excluded rather than counted.

Format all six: callout colour `#BE123C`, background `#FFF1F2`.

### 8.2 — Phenotype by organism group

> **Insert → Clustered bar chart**
> **Y-axis:** `Organism group`
> **X-axis:** `CRE Isolates`, `ESBL Isolates`, `MDR Isolates`, `VRE Isolates`, `MRSA Isolates`

**Title:** `Klebsiella is CRE-dominant while E. coli is ESBL-dominant`

*This is a genuinely useful clinical distinction — same bacterial family, different resistance mechanism.*

### 8.3 — Resistance by unit

> **Insert → Clustered bar chart**
> **Y-axis:** `Unit` · **X-axis:** `Resistance Rate`
> **Filter:** `Eligible Isolates` → is greater than or equal to **40**
> Sort descending, Top 15

> ⚠️ **The minimum-count filter is essential.** Without it, a ward with 3 isolates and 2 resistant shows "66.7%" and tops your chart — visually dominant, statistically meaningless.

### 8.4 — Resistance over time

> **Insert → Line chart**
> **X-axis:** `Month label` · **Y-axis:** `Resistance Rate` and `Resistance Rate Overall`

---

<a name="page-4"></a>
## 9. PAGE 4 — Hospital units

**Question:** where in the hospital is the burden concentrated?

### 9.1 — Unit type comparison (top)

> **Insert → Clustered column chart**
> **X-axis:** `Unit type` · **Y-axis:** `HAI Rate` and `Resistance Rate`

**Title:** `Critical care has more HAI, but resistance is similar hospital-wide`

*This visual carries one of your key findings: HAI concentrates in intensive care, but resistance does not.*

### 9.2 — HAI volume by unit

> **Insert → Clustered bar chart**
> **Y-axis:** `Unit` · **X-axis:** `HAI Samples`
> Top 15, sorted descending

### 9.3 — HAI rate by unit

> **Insert → Clustered bar chart**
> **Y-axis:** `Unit` · **X-axis:** `HAI Rate`
> **Filter:** `Samples` ≥ **40**
> Top 15, sorted descending

*Two charts, deliberately. Volume shows where the most cases are; rate shows where the risk is concentrated. They give different answers, and both matter.*

### 9.4 — Unit × specimen matrix

> **Insert → Matrix**
> **Rows:** `Unit` · **Columns:** `Specimen` · **Values:** `Samples`
> **Filter rows:** Top 15 by `Samples`

Format: **Cell elements → Background colour** On for `Samples` — creating a heatmap of the cross-tab.

---

<a name="page-5"></a>
## 10. PAGE 5 — Monthly surveillance

**Question:** is it getting better or worse?

### 10.1 — HAI rate with reference line

> **Insert → Line chart**
> **X-axis:** `Month label` · **Y-axis:** `HAI Rate`, `HAI Rate Overall`
> Y-axis Start 0.30, End 0.65

**Title:** `Monthly HAI rate: elevated mid-period, declining in the final quarter`

### 10.2 — Sampling volume

> **Insert → Clustered column chart**
> **X-axis:** `Month label` · **Y-axis:** `Samples`
> Bars in `#CBD5E1`

*Include this so viewers can see that rate changes are not simply volume changes — sampling stayed stable at 721–854 per month.*

### 10.3 — Monthly summary table

> **Insert → Table**
> **Columns:** `Month label`, `Samples`, `HAI Samples`, `HAI Rate`, `Eligible Isolates`, `Resistance Rate`

### 10.4 — Control limits (optional, advanced)

Power BI cannot compute month-specific 3σ limits natively. Two honest options:

**Option A — simple.** Use **Analytics pane → Average line** on the HAI rate chart. Sufficient for a dashboard, and reference the proper SPC chart in your report.

**Option B — proper.** Create a small 14-row table with the UCL/LCL values from your Step 4 Python output:

| Month label | UCL | LCL |
|---|---|---|
| 2025-04 (Apr) | 0.521 | 0.414 |
| … | … | … |

Load it, relate it to `Month label`, and add both as extra lines on the chart.

> **Never draw flat 3σ lines.** Your monthly denominators vary (721–854), so the limits genuinely differ each month — that is precisely the point made in §5.3 of the report.

---

<a name="part-11"></a>
## 11. Slicers and cross-page filtering

### 11.1 — Build the slicer panel

On **Page 1**, add four slicers down the left-hand side (**Insert → Slicer**, then drag the field in):

| Slicer | Field | Style |
|---|---|---|
| Month | `Month label` | Dropdown |
| Unit type | `Unit type` | Vertical list |
| Specimen | `Specimen` | Dropdown |
| Infection origin | `Infection origin` | Vertical list |

Format each: **Slicer settings → Style → Dropdown** for the long lists; **Selection → Multi-select with Ctrl** Off (so plain clicking multi-selects).

### 11.2 — Copy them to every page

Select all four slicers → **Ctrl+C** → go to each page → **Ctrl+V**. Keep them in the same position on every page so users always know where they are.

### 11.3 — Sync them

> **View → Sync slicers**
> Select each slicer, and tick **Sync** and **Visible** for all five pages.

Now a filter set on one page carries to all the others.

### 11.4 — Add a reset button

1. Clear all slicers.
2. **View → Bookmarks → Add** → rename it `Reset`.
3. **Insert → Buttons → Reset**, place it above the slicers.
4. Select the button → **Format → Action** → On → Type **Bookmark** → `Reset`.

---

<a name="part-12"></a>
## 12. Final polish

**Titles that state the finding, not the mechanics.** Compare:

- ❌ `Resistance rate by organism group`
- ✅ `Documented resistance is highest in Acinetobacter and Klebsiella`

The second tells the reader what to see. Do this for every chart.

**Alignment.** Select multiple visuals → **Format → Align** → Align left / Distribute horizontally. Misaligned visuals are the fastest way to look unprofessional.

**Data labels On** for all bar charts — people read numbers, not pixel lengths.

**Consistent decimals.** Percentages to 1 decimal place everywhere.

**Tooltips.** Select a visual → **Format → Tooltips** → add `Samples` and `Eligible Isolates` so hovering shows the underlying counts behind a rate.

**Page navigation.** **Insert → Buttons → Navigator → Page navigator** — inserts a linked button per page automatically. Place along the top of each page.

---

<a name="part-13"></a>
## 13. Verification checklist

Clear all slicers, then confirm these against the report. **Do this before you present.**

| Check | Expected |
|---|---|
| Samples | 10,943 |
| Isolates | 12,188 |
| Patients | 5,311 |
| Multi-organism samples | 1,073 |
| HAI rate | 46.8% |
| Eligible isolates | 8,162 |
| Resistant isolates | 4,491 |
| Resistance rate | 55.0% |
| Klebsiella pneumoniae resistance rate | 77.8% |
| Acinetobacter sp resistance rate | 79.8% |
| CRE isolates | 2,120 |
| Critical care HAI rate | higher than general ward |

**If a number is wrong, it is almost always one of two causes:**
1. A **count** used where a **distinct count** was needed (inflates sample figures)
2. A **missing `AMR eligible` filter** (deflates resistance rates)

---

<a name="part-14"></a>
## 14. Troubleshooting

**"Query errors" folder appears on import**
Auto-detection guessed a column type wrongly. Delete the query, turn off auto-detection (Part 2.1), re-import the `.xlsx`.

**Months sort as Apr, Aug, Dec, Feb…**
Should not happen with the `2025-04 (Apr)` format, which sorts alphabetically into the right order. If it does, select `Month label` → **Column tools → Sort by column → Month order**.

**A card shows 12,188 when you expect 10,943**
You dragged `Sample ID` directly instead of using the `Samples` measure. Replace it.

**Resistance rate looks too low (around 37%)**
The `AMR eligible` filter is missing, so fungi and viruses are diluting the denominator. Use the `Resistance Rate` measure rather than a raw average of `Resistant flag`.

**A tiny ward tops the rate chart at 100%**
Add the minimum-count filter: Filters pane → `Samples` (or `Eligible Isolates`) → is greater than or equal to 40.

**Blank values appear in a visual**
Some specimen records are unusable (15 rows). Filter them out in the visual's filter pane, or leave them — they are a negligible share.

**A slicer filters one page but not others**
Sync was not enabled. **View → Sync slicers** → tick Sync for every page.

---

<a name="part-15"></a>
## 15. Writing it up in Chapter 6

The dashboard chapter should present the tool as a **product**, not repeat the analysis. Structure it as:

**6.1 Purpose and audience** — who uses it (infection-control team, clinicians, administration) and what decisions it supports.

**6.2 Data model** — how the cleaned register feeds the dashboard: 12,188 organism-level rows, the derived columns (`Organism class`, `AMR eligible`, `Resistant flag`, `Unit type`, `Infection origin`), and the DAX measures. Explain the distinct-count requirement and the eligibility filter — these are genuine design decisions worth describing.

**6.3 Page walkthrough** — a screenshot of each page, explained by its **design rationale**: why these KPIs, why this layout, what a user learns at a glance.

**6.4 Interactivity** — a worked example: *"an infection-control nurse wants to know which organisms drive resistance in the NICU: she selects NICU in the unit slicer, and the organism page updates to show that ward's profile."*

**6.5 How it supports surveillance** — connect back to the analysis: the dashboard is where the findings of Chapters 4 and 5 become usable month to month.

> **Do not re-analyse charts already covered in Chapter 4.** Reference them (*"the organism distribution, §4.4"*) and spend the words on design and use instead. That distinction — analysis versus delivery — is what separates "I made a dashboard" from "I designed a decision tool".

---

## Build order

Don't build all five pages at once. Work in this sequence:

1. Import and verify (Parts 2–3) — confirm 12,188 rows, no errors
2. Create all measures (Part 4) — test each on a blank card
3. Set the theme (Part 5)
4. Build **Page 1** completely, including formatting
5. Run the verification checklist on Page 1's cards
6. Build Pages 2–5
7. Add slicers and sync (Part 11)
8. Polish (Part 12)
9. Full verification (Part 13)

**Get Page 1's five cards showing the right numbers before building anything else.** If those are correct, your model is sound and everything downstream will be too.