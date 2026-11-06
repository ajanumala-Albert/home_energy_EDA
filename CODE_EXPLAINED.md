# Code Explained — EDA FINAL_cleaned.ipynb

This document explains every major step in the notebook in plain language.
It is written to help the team present and answer questions confidently.

---

## Section 1 · Context & provenance

**What it does**
Records where the data comes from, what question it answers, who collected it, and what
its known limitations are — before a single line of analysis runs.

**Why we did it**
In data science a "provenance card" is the first thing a reviewer or examiner checks.
It proves we know the source, the licence, and what the data can and cannot answer.
It also captures the collection date so someone running the notebook six months later
knows whether the data is fresh or stale.

**Key choices**
- We filtered to `code_departement_ban:"75"` (Paris) to keep the dataset small and
  geographically coherent for a student project. The full national dataset has over
  800,000 rows, which would be slow to collect and analyse.
- We capped collection at 10,000 rows. This is enough to see real distributions without
  overloading the API or our machines.

---

## Section 2 · Setup & parameters

**What it does**
Defines every tuneable number (seed, paths, cache duration, outlier multiplier, API URL,
page size, row cap) in one cell, then imports all libraries and creates the folder structure.

**Why we did it**
Putting all parameters at the top means anyone can rerun the notebook on a different
department or with a different row cap by editing one cell. This is a standard
"single point of truth" pattern taught in reproducible research.

**Key choices**
- `SEED = 42` — fixed random seed so shuffling and sampling produce the same result
  every run.
- `IQR_K = 1.5` — the classic Tukey fence multiplier for outlier detection (explained
  further in Section 4.4).
- `MAX_MISSING_SHARE = 0.30` — columns with more than 30% missing values would be
  flagged for exclusion (none reached that threshold in our data).

---

## Section 3.1 · Polite, cached HTTP session

**What it does**
Creates an HTTP session that caches responses to disk, retries on server errors with
exponential backoff, and pauses between requests that actually hit the network.

**Why we did it**
The course (Session 1) requires this exact pattern. Beyond the grade, it is the
professional standard for anyone consuming a public API:
- **Cache** — rerunning the notebook uses stored responses instead of hammering the server.
  One hour of work does not make hundreds of repeated requests.
- **Retry with backoff** — servers occasionally return a 429 (Too Many Requests) or a
  transient 500. Retrying after 1 s, 2 s, 4 s, … gives the server time to recover
  without overwhelming it.
- **Respecting `Retry-After`** — if the server explicitly tells us to wait, we wait
  exactly that long.
- **User-Agent** — identifying our client by name is good citizenship; it lets the
  API provider contact us if we are causing problems.

**Key choices**
- `expire_after=CACHE_HOURS * 3600` (one week) — long enough to survive multiple work
  sessions on the same dataset without re-downloading.
- `status_forcelist=[429, 500, 502, 503, 504]` — we retry server-side errors but NOT
  client-side 4xx errors (except 429), because a 404 or 403 will not fix itself with a retry.
- `timeout=30` — every request has a timeout so a hanging server cannot stall the
  notebook forever.

---

## Section 3.2 · Paginated collection

**What it does**
Follows the API's `next` link from page to page, collecting records until it has
`MAX_ROWS` rows or the API runs out of pages.

**Why we did it**
The ADEME API returns at most 1,000 records per request. To get 10,000 rows we need
to follow 10 pages. The `collect_next_link` function handles this automatically and
works for any API that uses the same "next link" pagination pattern.

**Key choices**
- We read the total row count from the first response and print it. This lets us verify
  that our filter (`code_departement_ban:"75"`) matched real rows before waiting for
  all 10 pages.

---

## Section 3.3 · Flatten, save raw Parquet, write README

**What it does**
Converts the list of JSON objects into a flat DataFrame, saves it as a Parquet file,
and writes a README alongside it.

**Why we did it**
- **`pd.json_normalize`** — the API returns nested JSON (some fields contain objects).
  `json_normalize` flattens nested keys into column names joined by `_`, giving us a
  proper tabular structure.
- **Parquet instead of CSV** — Parquet preserves data types (strings stay strings,
  floats stay floats). CSV turns everything into text, which forces us to re-infer types
  on every load. Parquet is also smaller and faster to read.
- **README** — documents the shape, column list, and collection timestamp so anyone
  opening the `data/raw/` folder knows what they are looking at without opening the notebook.

---

## Section 3.4 · Column audit and pandera schema

**What it does**
1. Audits 216 raw columns, selects 17 that are relevant to our question.
2. Writes a formal `DataFrameSchema` that specifies the expected type, allowed values,
   and nullability of every selected column.
3. Validates the data against that schema and reports any violations.

**Why we did it**
A schema is a written contract between the collector and the analyst. If the API changes
a column name or starts returning unexpected values, the schema catches it immediately
rather than letting bad data flow silently into the analysis.

**Key choices**
- `strict=True` — the schema rejects any column not listed in it. This prevents us from
  accidentally using a column we did not inspect.
- `lazy=True` — without this, pandera stops at the first violation. `lazy=True` collects
  all violations in one pass so we can see the full picture.
- `nullable=True` only where justified — every nullable column has a comment explaining
  why (e.g. `surface_habitable_logement` is missing for collective-building records where
  the surface is recorded at building level, not dwelling level).
- Domain checks (examples): `etiquette_dpe` must be one of A–G; `code_postal_ban`
  must match `^75\d{3}$`; `surface_habitable_logement` must be greater than 0;
  `conso_5_usages_par_m2_ep` must be ≥ 0.

---

## Section 4 · Data quality

### 4.1 · Exact duplicates

**What it does**
Counts rows where every selected column is identical across two records.

**Why we did it**
An exact duplicate contributes no new information and would inflate counts. We found
zero duplicates because `numero_dpe` (the DPE identifier) is unique by design.

---

### 4.2 · Impossible values

**What it does**
Checks numeric columns for values that are physically impossible (e.g. a surface ≤ 0 m²,
a negative energy consumption).

**Why we did it**
API data can contain data-entry errors. Surfaces of 0 or negative values are impossible;
a dwelling that consumes negative energy does not exist. We checked four columns and found
no impossible values, so no rows were removed.

---

### 4.3 · Missingness — mechanism before method

**What it does**
Plots two heatmaps showing the missingness rate of the three partially-missing columns
(`surface_habitable_logement`, `nombre_niveau_logement`, `type_installation_chauffage`)
broken down by building type and by DPE application method.

**Why we did it**
Before deciding how to handle missing values you must understand *why* they are missing.
There are three mechanisms:
- **MCAR** (Missing Completely At Random) — no pattern; safe to drop or impute freely.
- **MAR** (Missing At Random) — missingness depends on another observed variable; can
  be imputed using that variable.
- **MNAR** (Missing Not At Random) — missingness depends on the missing value itself;
  imputation is unreliable.

Our heatmaps showed a clear pattern: `surface_habitable_logement` and
`nombre_niveau_logement` are missing almost exclusively for `immeuble` (collective
building) records, and `type_installation_chauffage` is missing for all `maison` records.

**Key choices — why no imputation**
Imputing the surface of an apartment using the mean surface of other apartments would
invent a number that may not reflect reality for that specific building. Because the
missingness is structural (the field is recorded differently or not at all for certain
record types), we chose to keep the missing values as `NaN` and document the reason.
Imputing would bias any model trained on this data.

---

### 4.4 · Outlier triage

**What it does**
Applies a log-IQR fence to `surface_habitable_logement`, flags the 121 values outside
the fence, and adds a boolean `surface_outlier_flag` column. No rows are removed.

**Why we did it**
The course (Session 4) requires an outlier *decision table*, not automatic deletion.
Dropping outliers without inspection destroys real information and is not defensible.

**Why log-IQR, not raw IQR**
Surface area is right-skewed: most dwellings are small (20–60 m²) but a few large
apartments and collective buildings have surfaces in the hundreds or thousands.
Applying the IQR fence directly to skewed data flags too many legitimate small values
as outliers and misses large ones. Taking the log first compresses the long right tail
and makes the distribution roughly symmetric, so the fence behaves as intended.

**Key choices**
- `IQR_K = 1.5` — the standard Tukey multiplier. A value of 3.0 would be more
  conservative (fewer flags).
- Flagged, not dropped — the 121 flagged values included large apartments in the 7th and
  16th arrondissements (Haussmann-era buildings with genuinely large surfaces) and
  collective-building records. These are real, so we kept them and noted the flag.

---

## Section 5 · Exploration

### 5.1 · DPE label distribution (Figure 1)

**Chart type:** Coloured vertical bar chart, one bar per label A–G.

**What it shows:** How many records fall into each energy-performance category.

**Why this chart:** A bar chart is the clearest way to compare counts across a small
number of ordered categories. We used the official DPE colours (green → red) so the
chart is immediately readable to anyone familiar with French energy labels.

**Finding:** Labels D and E together represent ~63% of the sample. Labels A and B are
very rare. This class imbalance matters for modelling: a classifier that always predicts
D would score around 27% accuracy but would still look "decent" on a headline metric.

---

### 5.2 · DPE labels by construction period (Figure 2)

**Chart type:** 100% stacked bar chart, one bar per construction period.

**What it shows:** How the mix of DPE labels changes across construction eras.

**Why this chart:** A 100% stacked bar normalises for group size, so a period with
5 records and one with 6,000 are shown on the same scale. We care about proportions,
not counts, because construction periods have very different sample sizes.

**Finding:** Buildings from before 1948 have over 65% of records in labels E–G.
Buildings from 2006–2012 have 0% in E–G. This confirms that construction period is a
strong predictor of energy performance, which supports Hypothesis H1.

---

### 5.3 · Median surface by label and building type (Figure 3)

**Chart type:** Heatmap with log colour scale and annotated medians.

**What it shows:** Median dwelling surface for each combination of DPE label (A–G)
and building type (appartement, immeuble, maison).

**Why this chart:** A heatmap lets us show three variables at once (surface, label,
building type) in a compact grid. The log scale is used because collective-building
surfaces are 10–50× larger than apartment surfaces; a linear scale would wash out all
variation in the apartment rows.

**Finding:** Among apartments, median surface decreases as the label worsens (from ~69 m²
for label B to ~27 m² for label G). Smaller apartments tend to have worse labels,
which supports Hypothesis H2. The collective-building cells are based on very few records
and should not be over-interpreted.

---

## Section 6 · Findings, hypotheses, and fingerprint

**What it does**
Summarises three main findings, lists what cannot be trusted, states five testable
hypotheses for the modelling phase, re-validates the clean data against the schema,
saves the processed Parquet, and prints a SHA-256 fingerprint.

**Why the fingerprint**
The course (Session 5) requires that the notebook produces an identical fingerprint
on two consecutive runs. This proves the analysis is deterministic: the same input
data produces the same output every time, with no hidden randomness. If the fingerprint
changes between runs, something in the pipeline is non-deterministic (e.g. an unsorted
groupby or a time-based value embedded in the output data).

**Why we validate the clean data again**
The schema we wrote in Section 3.4 covers the raw selected data. After Section 4 we
added a new column (`surface_outlier_flag`). Re-validating at the end confirms the
clean DataFrame still matches the contract before we hand it to a modeller.

**The five hypotheses**
Each hypothesis specifies: the variable, the expected direction of the effect, the
population it applies to, and the statistical test that could confirm or refute it.
This format follows the Session 5 requirement and makes hypotheses falsifiable.

| # | Variable | Expected direction | Test |
|---|---|---|---|
| H1 | `periode_construction` | Older → more E–G labels | Chi-square + proportion comparison |
| H2 | `surface_habitable_logement` | Smaller surface → worse label (apartments) | Spearman correlation or Kruskal-Wallis |
| H3 | `type_energie_principale_chauffage` | Fossil fuels → more E–G labels | Chi-square or logistic regression |
| H4 | `etiquette_ges` | Worse GES → worse DPE | Chi-square + Cramér's V |
| H5 | `type_batiment` | Label distributions differ by building type | Chi-square or multinomial logistic regression |
