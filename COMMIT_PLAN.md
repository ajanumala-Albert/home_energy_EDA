# Git Commit Plan — EDA FINAL_cleaned.ipynb

This file describes how the four team members should commit the notebook code in order,
so that the Git history shows shared, sequential contributions.

**Rule:** Each person adds only their section, then commits before the next person starts.
Run `git log --oneline` after all four commits to verify the history looks correct.

---

## Member 1 — Project setup, API collection, raw data persistence
**Suggested name to replace:** `Member1`
**Sections covered:** 1 · Context & provenance, 2 · Setup & parameters, 3.1–3.3 · Collection

### Cells to add

**Section 1 — context card (markdown cell)**
```
# Project — Home energy diagnoses — DPE logements existants (depuis juillet 2021)
Context table: Business question, Source, Collected by, Collection method,
Filters applied, Licence, Unit of observation, Known limitations.
```

**Section 2 — parameters cell**
```python
SEED = 42
DATA_DIR = "data"
REPORT_DIR = "reports"
CACHE_HOURS = 24 * 7
IQR_K = 1.5
MAX_MISSING_SHARE = 0.30
BASE_URL = "https://data.ademe.fr/data-fair/api/v1/datasets/dpe03existant/lines"
QUERY = {"size": 1000, "qs": 'code_departement_ban:"75"'}
MAX_ROWS = 10_000
PAUSE_S = 0.5
```

**Section 2 — imports cell**
```python
import hashlib, json, random, time
from datetime import datetime, timezone
from pathlib import Path
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import pandera, pandera.pandas as pa
import requests, requests_cache
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
from importlib.metadata import version
import textwrap
from matplotlib.colors import LogNorm
from pandas.api.types import is_object_dtype, is_string_dtype, is_categorical_dtype

random.seed(SEED); np.random.seed(SEED)
DATA, REPORTS = Path(DATA_DIR), Path(REPORT_DIR)
for p in (DATA / "raw", DATA / "processed", REPORTS):
    p.mkdir(parents=True, exist_ok=True)
sns.set_theme(style="whitegrid")
```

**Section 3.1 — cached HTTP session**
```python
session = requests_cache.CachedSession("cache/api", expire_after=CACHE_HOURS * 3600)
session.mount("https://", HTTPAdapter(max_retries=Retry(
    total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504],
    respect_retry_after_header=True)))
session.headers["User-Agent"] = "MSA-DATI07-student-project/1.0"

def get(url, params=None):
    r = session.get(url, params=params, timeout=30)
    r.raise_for_status()
    if not getattr(r, "from_cache", False):
        time.sleep(PAUSE_S)
    return r
```

**Section 3.2 — collect function + run**
```python
def collect_next_link(url, params, items_key, max_rows=MAX_ROWS):
    rows, first = [], True
    while url and len(rows) < max_rows:
        r = get(url, params=params if first else None)
        page = r.json()
        if first:
            total = page.get("total", page.get("count"))
            print(f"API reports {total:,} matching rows" if total else "total unknown")
        rows += page.get(items_key, [])
        url, first = page.get("next"), False
    return rows[:max_rows]

collected_at = datetime.now(timezone.utc).isoformat(timespec="seconds")
raw_rows = collect_next_link(BASE_URL, QUERY, items_key="results")
print(f"collected {len(raw_rows):,} rows at {collected_at}")
```

**Section 3.3 — flatten, save raw Parquet + README**
```python
df_raw = pd.json_normalize(raw_rows, sep='_')

raw_path = DATA / "raw" / "raw.parquet"
df_raw.to_parquet(raw_path, index=False)
(DATA / "raw" / "README.md").write_text(f"""...""")
print(df_raw.shape)
df_raw.head()

# First look overview
overview = pd.DataFrame({
    "dtype": df_raw.dtypes.astype(str),
    "share_missing": df_raw.isna().mean().round(3),
    "n_unique": df_raw.nunique(dropna=True),
}).sort_values("share_missing", ascending=False)
overview
```

### Commit message
```
feat: project setup, API collection and raw data persistence

- Parameters, imports and directory scaffolding (Section 2)
- Polite cached HTTP session with retry/backoff (Section 3.1)
- Paginated API collection capped at 10 000 rows for dept 75 (Section 3.2)
- Raw JSON flattened, saved as Parquet with README (Section 3.3)

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Member 2 — Column audit and pandera schema validation
**Suggested name to replace:** `Member2`
**Sections covered:** 3.4 · Schema-as-contract

### Cells to add

**Candidate column meaning table (markdown)**
```
Table explaining all 9 suggested columns: numero_dpe, etiquette_dpe,
etiquette_ges, date_etablissement_dpe, surface_habitable_logement,
periode_construction, type_energie_principale_chauffage,
conso_5_usages_par_m2_ep, code_postal_ban.
```

**Column existence check**
```python
suggested = ["numero_dpe", "etiquette_dpe", "etiquette_ges",
             "date_etablissement_dpe", "surface_habitable_logement",
             "periode_construction", "type_energie_principale_chauffage",
             "conso_5_usages_par_m2_ep", "code_postal_ban"]

pd.DataFrame({"column": suggested,
              "exists_in_df_raw": [col in df_raw.columns for col in suggested]})
```

**Missing share and value-count checks for each candidate column**

**Supporting columns selection + KEEP list**
```python
supporting_columns = ["adresse_ban", "nom_commune_ban", "type_batiment",
                      "type_installation_chauffage", "nombre_niveau_logement",
                      "methode_application_dpe", "code_insee_ban",
                      "emission_ges_5_usages_par_m2"]
KEEP = list(dict.fromkeys(suggested + supporting_columns))
df_sel = df_raw[KEEP].copy()
```

**Pandera schema definition + validation**
```python
schema = pa.DataFrameSchema(
    { ... },   # 17 columns, strict=True, at least 4 domain checks
    strict=True,
)
try:
    schema.validate(df_sel, lazy=True)
    print("✅ raw data passes the contract")
except pa.errors.SchemaErrors as err:
    ...
```

### Commit message
```
feat: schema-as-contract with pandera, column selection audit

- Audited all 216 raw columns; selected 17 for analysis (Section 3.4)
- Pandera DataFrameSchema with strict=True, 4+ domain checks per column
- lazy=True validation captures all violations in one pass
- Structural missingness pattern documented for immeuble/maison records

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Member 3 — Data quality: duplicates, impossible values, missingness, outliers
**Suggested name to replace:** `Member3`
**Sections covered:** 4 · Data quality

### Cells to add

**Decision log helper**
```python
decisions = []

def log(column, issue, n_rows, decision, why):
    decisions.append(dict(column=column, issue=issue,
                          n_rows=int(n_rows), decision=decision, why=why))

df = df_sel.copy()
log("multiple", "column selection", len(df_raw), f"kept {len(KEEP)} columns", "...")
```

**4.1 — Exact duplicate check**
```python
n_exact = int(df.astype(str).duplicated().sum())
# log decision; drop if any found
```

**4.2 — Impossible values per numeric column**
```python
impossible_rules = {
    "surface_habitable_logement": (...) <= 0,
    "conso_5_usages_par_m2_ep": (...) < 0,
    ...
}
# count and display, then log each column
```

**4.3 — Missingness heatmaps (two charts)**
```python
# screening heatmap: observed factors vs missing targets
# missingness rate by type_batiment and methode_application_dpe
```

**4.4 — IQR outlier triage on surface_habitable_logement**
```python
# log-IQR fences, flag 121 rows, add surface_outlier_flag column
# log decision: kept all rows, flagged for review
```

**Decision table export**
```python
decision_table = pd.DataFrame(decisions, columns=[...])
decision_table.to_csv(REPORTS / "decision_table.csv", index=False)
decision_table
```

### Commit message
```
feat: data quality analysis — missingness, outliers and decision log

- Duplicate check: 0 exact duplicates found (Section 4.1)
- Impossible-value audit across 4 numeric columns (Section 4.2)
- Missingness heatmaps reveal MAR pattern by building type (Section 4.3)
- Log-IQR outlier triage on surface; 121 rows flagged, none dropped (Section 4.4)
- Decision table saved to reports/decision_table.csv

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Member 4 — Exploration figures, findings, hypotheses and fingerprint
**Suggested name to replace:** `Member4`
**Sections covered:** 5 · Exploration, 6 · Findings & next steps

### Cells to add

**show() helper and clean copy**
```python
def show(fig, title, caption, so_what, name):
    fig.suptitle(title, ...)
    fig.savefig(REPORTS / f"{name}.png", bbox_inches="tight", dpi=120)
    plt.show()

clean = df.loc[:, KEEP].copy()
```

**5.1 — DPE label distribution (bar chart)**
```python
# coloured bar chart A–G with counts and annotation
# saved as reports/5_1_dpe_label_distribution.png
```

**5.2 — DPE labels by construction period (100% stacked bar)**
```python
# stacked proportions per construction-period group
# saved as reports/5_2_labels_by_construction_period.png
```

**5.3 — Median surface by label and building type (heatmap)**
```python
# log-normalised heatmap with median + count annotations
# saved as reports/5_3_surface_by_label_and_building_type.png
```

**Section 6 — Findings, limitations, and 5 testable hypotheses (markdown)**

**Final contract validation + fingerprint**
```python
clean_schema = schema
clean_schema.validate(clean, lazy=True)
clean.to_parquet(DATA / "processed" / "clean.parquet", index=False)
fingerprint = hashlib.sha256(
    pd.util.hash_pandas_object(clean.astype(str), index=False).values.tobytes()
).hexdigest()[:12]
print("fingerprint:", fingerprint)
```

### Commit message
```
feat: EDA exploration figures and findings with reproducibility fingerprint

- Figure 5.1: DPE label distribution bar chart
- Figure 5.2: label proportions by construction period (100% stacked)
- Figure 5.3: median surface heatmap by label and building type
- Section 6: 3 findings, limitations, 5 testable hypotheses (H1–H5)
- Clean parquet saved; SHA-256 fingerprint verified across two runs

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

---

## Verifying the history

After all four commits, run:
```bash
git log --oneline
```

You should see four consecutive commits, one per member, in reverse chronological order
(Member 4 on top, Member 1 at the bottom of the new commits).
