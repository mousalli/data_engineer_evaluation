Here’s a clean, drop-in **README.md** you can use. Want me to save it as a file and hand you a download link?

# Data Engineer Case Study (24-Hour Demo)

This repository contains a small analytics demo built in **Google Colab** with **SQLite**. It answers Sets #1–#4 on a MIMIC-style sample and presents a minimal end-to-end data architecture.
**Note:** This work was prepared as part of an **evaluation for a Data Engineer position**.

## Tech stack

* Google Colab (Python 3.x)
* pandas, numpy, matplotlib
* SQLite (via `sqlite3`)
* (Optional) DBeaver or ERAlchemy for ERD diagrams

## Data

Required CSVs (demo names may vary slightly):

* `admissions.csv`
* `patients.csv`
* `diagnoses_icd.csv`
* `labevents.csv`
* `d_icd_diagnoses.csv`
* `d_labitems.csv`

Optional (for Set #4 “split datasets”):

* `patient_1.csv`, `patient_2.csv` (and if provided, `admissions_1.csv`, `admissions_2.csv`)

### Load data from Google Drive into **/content**

In Colab, mount Drive and copy the CSVs so the notebook reads from **`/content/data`**:

```python
from google.colab import drive
drive.mount('/content/drive')

# Adjust the source path to where your files live in Drive
!mkdir -p /content/data
!cp "/content/drive/MyDrive/Colab Notebooks/data/"*.csv /content/data/
```

Now all code assumes:

```
/content
  └─ data/
      admissions.csv
      patients.csv
      diagnoses_icd.csv
      labevents.csv
      d_icd_diagnoses.csv
      d_labitems.csv
      (optional) patient_1.csv, patient_2.csv, admissions_1.csv, admissions_2.csv
```

## Quick start (Colab)

```python
import pandas as pd, sqlite3
from pathlib import Path

DB = "/content/mimic_demo.db"
DATA_DIR = Path("/content/data")
OUT = Path("/content/outputs"); OUT.mkdir(parents=True, exist_ok=True)

conn = sqlite3.connect(DB)

def load(name, filename):
    df = pd.read_csv(DATA_DIR/filename)
    df.to_sql(name, conn, if_exists="replace", index=False)
    print(f"Loaded {name}: {df.shape}")

load("admissions", "admissions.csv")
load("patients",   "patients.csv")
load("diagnoses",  "diagnoses_icd.csv")
load("labevents",  "labevents.csv")
load("d_icd",      "d_icd_diagnoses.csv")
load("d_lab",      "d_labitems.csv")

conn.executescript("""
CREATE INDEX IF NOT EXISTS idx_adm_hadm ON admissions(hadm_id);
CREATE INDEX IF NOT EXISTS idx_adm_subj ON admissions(subject_id);
CREATE INDEX IF NOT EXISTS idx_pat_subj ON patients(subject_id);
CREATE INDEX IF NOT EXISTS idx_dx_subj  ON diagnoses(subject_id);
CREATE INDEX IF NOT EXISTS idx_dx_code  ON diagnoses(icd_code, icd_version);
CREATE INDEX IF NOT EXISTS idx_lab_item ON labevents(itemid);
CREATE INDEX IF NOT EXISTS idx_lab_subj ON labevents(subject_id);
""")
print("Indexes created.")
```

## What’s included

* **Set #1:** Gender distribution, average & median age, LOS mean + histogram (0–14 days).
* **Set #2:** Average # of unique ICD codes per patient; Top-10 ICD codes with % and long title; per-subject unique code list.
* **Set #3:** Lab label stats — numeric mean (`valuenum`) and `value IS NULL` missing%.
* **Set #4:** Mean age across two patient datasets **without centralizing rows** (combine `(sum_age, n)` metrics only).

### Example: unique ICD codes per subject (SQLite)

```sql
WITH u AS (
  SELECT DISTINCT subject_id, icd_code, icd_version
  FROM diagnoses
)
SELECT subject_id,
       GROUP_CONCAT(icd_code || ':' || icd_version, ',') AS unique_icd_codes
FROM u
GROUP BY subject_id
ORDER BY subject_id;
```

### Example: Set #4 (metric federation)

```python
def split_mean_age_from_patients(p_csv):
    pd.read_csv(p_csv).to_sql("patients_tmp", conn, if_exists="replace", index=False)
    return pd.read_sql_query("""
    WITH first_adm AS (
      SELECT subject_id, MIN(admittime) AS first_admit
      FROM admissions
      WHERE subject_id IN (SELECT subject_id FROM patients_tmp)
      GROUP BY subject_id
    ),
    ages AS (
      SELECT (julianday(first_admit) - julianday(dob)) / 365.2425 AS age
      FROM patients_tmp JOIN first_adm USING(subject_id)
      WHERE dob IS NOT NULL AND first_admit IS NOT NULL
    )
    SELECT SUM(age) AS sum_age, COUNT(*) AS n FROM ages
    """, conn).iloc[0]

import math
s1 = split_mean_age_from_patients("/content/data/patient_1.csv")
s2 = split_mean_age_from_patients("/content/data/patient_2.csv")
combined_mean = (float(s1.sum_age)+float(s2.sum_age)) / (int(s1.n)+int(s2.n))
combined_mean
```

## ERD (schema)

You can generate an ER diagram via:

* **Matplotlib (no installs):** a small script that draws boxes/edges based on column presence.
* **DBeaver:** add **Virtual PK/UK** on parent columns and **Virtual FKs** on child columns, then export ERD.
* **ERAlchemy + Graphviz:** quick PNG from a “shadow” schema with declared FKs.

## Outputs

The notebook saves key artifacts to **`/content/outputs`**:

* `gender_distribution.csv`
* `age_summary.csv`
* `los_hist_0_14.png`
* `top10_icd.csv`
* `lab_label_stats.csv`
* (optional) `artifact_index.csv`

## Repository layout

```
/notebook.ipynb          # main Colab/Jupyter notebook
/content/data/           # CSVs copied from Google Drive (see instructions above)
/content/mimic_demo.db   # SQLite database (created by the notebook)
/content/outputs/        # generated charts & CSVs
README.md
```

## Notes

* This is a **time-boxed evaluation** artifact; it favors clarity and reproducibility over completeness.
* Demo data only. No PHI. Replace with your own datasets as needed.
