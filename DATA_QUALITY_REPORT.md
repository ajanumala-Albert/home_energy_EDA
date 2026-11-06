# Data-Quality Report, DPE logements existants

**Group:** Group 6 · **Date:** 2026-11-06 · **Notebook:** `EDA FINAL_cleaned.ipynb` · **Commit:** N/A
**Clean-data fingerprint:** `82c5585f6f33`

> The report uses the 10,000-record Paris snapshot collected on 2026-11-05. Claims refer to the notebook sections and saved figures.

## 1. Source & provenance

| | |
|---|---|
| Business question | Among DPE diagnosis records in department 75, how are energy labels distributed, and how are they associated with construction period and dwelling surface? |
| Source(s) & endpoint / URL | ADEME open-data portal, dataset `dpe03existant`: https://data.ademe.fr/datasets/dpe03existant; API: https://data.ademe.fr/data-fair/api/v1/datasets/dpe03existant/lines |
| Collection method & date | REST API, paginated in batches of 1,000, cached with `requests-cache`, retried on temporary errors, paused between network requests. Filter: `code_departement_ban:"75"`. Collected 2026-11-05 23:31:51 UTC. |
| Licence / constraints | Licence Ouverte / Etalab 2.0, subject to the ADEME dataset terms. The notebook uses the API politely with caching, retries and throttling. |
| Unit of observation | One row is one DPE diagnosis record identified by `numero_dpe`. A DPE record is not necessarily one unique dwelling; several records may share an address. |
| Final size | 10,000 rows × 17 columns. The API reported 862,377 matching records, but collection was capped at 10,000. |

## 2. Schema, the contract

The Pandera schema is defined in notebook Section 3.4. It uses `strict=True` and validates the 17 selected columns.

- `numero_dpe` — string and unique — protects against duplicate diagnosis identifiers.
- `etiquette_dpe`, `etiquette_ges` — values restricted to A–G — protects against invalid labels.
- `date_etablissement_dpe` — parseable date — protects against malformed dates.
- `surface_habitable_logement` — positive when present — protects against impossible surfaces.
- `conso_5_usages_par_m2_ep`, `emission_ges_5_usages_par_m2` — non-negative when present — protects against impossible energy and emissions values.
- `nombre_niveau_logement` — at least 1 when present — protects against impossible level counts.
- `code_postal_ban`, `code_insee_ban` — five-digit strings; postal code begins with 75 — protects against malformed geographic codes.
- `nullable=True` is used for fields with observed missingness: surface, consumption, emissions, heating installation and number of levels.

Validation on raw data: **0 violations reported**. Validation on clean data: **passes**.

## 3. Missingness

| Column | % missing | Mechanism | Evidence | Decision |
|---|---:|---|---|---|
| `nombre_niveau_logement` | 3.01% | Structural, compatible with MAR | 301 of 302 `immeuble` records are missing | Keep as `NaN`; no imputation |
| `surface_habitable_logement` | 3.00% | Structural, compatible with MAR | Missingness is concentrated in collective-building records | Keep as `NaN`; no imputation |
| `type_installation_chauffage` | 0.46% | Structural, compatible with MAR | All 46 missing values are in `maison` records | Keep as `NaN`; no imputation |
| `conso_5_usages_par_m2_ep` | 0.01% | Not identifiable | One missing value | Keep as `NaN` |
| `emission_ges_5_usages_par_m2` | 0.01% | Not identifiable | One missing value | Keep as `NaN` |

The missing values were not imputed because the patterns are related to the DPE record type. Filling them would invent information, for example treating a missing dwelling surface in a collective-building record as an average surface.

## 4. Duplicates & outliers, decision table

| Column | Issue | n rows | Decision | Why |
|---|---|---:|---|---|
| multiple | Column selection | 10,000 | Kept 17 columns | Nine brief-suggested fields plus eight provenance and quality fields were retained. |
| all | Exact duplicates | 0 | Kept all rows | No complete duplicate rows were found. |
| `surface_habitable_logement` | Impossible values | 0 | Kept all rows | No non-positive surfaces were found. |
| `conso_5_usages_par_m2_ep` | Impossible values | 0 | Kept all rows | No negative consumption values were found. |
| `emission_ges_5_usages_par_m2` | Impossible values | 0 | Kept all rows | No negative emissions values were found. |
| `nombre_niveau_logement` | Impossible values | 0 | Kept all rows | No non-missing value below one was found. |
| `nombre_niveau_logement` | Structural missingness | 301 | Kept missing values; no imputation | Missingness depends on building type and DPE method. |
| `surface_habitable_logement` | Structural missingness | 300 | Kept missing values; no imputation | Missingness is concentrated in collective-building records. |
| `type_installation_chauffage` | Structural missingness | 46 | Kept missing values; no imputation | All missing values occur in house records. |
| `conso_5_usages_par_m2_ep` | Isolated missing value | 1 | Kept missing value | Too few values to identify a mechanism. |
| `emission_ges_5_usages_par_m2` | Isolated missing value | 1 | Kept missing value | Too few values to identify a mechanism. |
| `surface_habitable_logement` | IQR outliers | 121 | Kept all rows; flagged for review | Values were statistically unusual but plausible for apartments, houses and collective buildings. |

## 5. Key findings

1. **DPE labels are concentrated in D and E.** Evidence: `5_1_dpe_label_distribution.png`. D and E represent about 64% of the sample, while A and B are rare. Modelling should account for class imbalance and report performance by label.

2. **Older buildings are associated with poorer labels.** Evidence: `5_2_labels_by_construction_period.png`. E–G represent 65.5% of records built before 1948, compared with 0.0% for 2006–2012. Construction period is a useful candidate predictor, but the relationship is observational.

3. **Among apartments, median surface decreases as the DPE label worsens.** Evidence: `5_3_surface_by_label_and_building_type.png`. Median apartment surface decreases from 69.2 m² for B to 27.4 m² for G. Sparse cells, especially collective-building cells, should not be overinterpreted.

## 6. What cannot be trusted

- `surface_habitable_logement` medians for `immeuble` records are unreliable because only a few non-missing values are available.
- Label proportions for very small construction groups, especially `après 2021` with 5 records, cannot be generalised.
- `type_installation_chauffage` cannot be analysed within `maison` because all 46 values are missing.
- The results are descriptive for a capped 10,000-record Paris snapshot and should not be treated as representative of all 862,377 matching records or as causal effects.

## 7. Hypotheses for modelling

- **H1:** Older `periode_construction` is associated with a higher proportion of E–G labels among Paris DPE records; test with a chi-square test and E–G proportion comparison by period.
- **H2:** Smaller `surface_habitable_logement` is associated with worse labels among apartments; test with Spearman correlation or Kruskal–Wallis using an ordinal A–G score.
- **H3:** Fossil heating energy, especially gas or fioul, is associated with a higher proportion of E–G labels than electricity or urban heating; test with chi-square or logistic regression for E–G versus A–D.
- **H4:** Worse `etiquette_ges` is associated with worse `etiquette_dpe`; test with a chi-square test and Cramér's V.
- **H5:** DPE-label distributions differ by `type_batiment`; test with a chi-square test or multinomial logistic regression.

## 8. How to reproduce

Run from the project root with the cached API data available:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
jupyter nbconvert --to notebook --execute "EDA FINAL_cleaned.ipynb"
```

Expected output: 10,000 clean rows and fingerprint `82c5585f6f33`.

The fingerprint is expected to remain stable when the cache is used. A fresh API collection after the cache expires may produce a different snapshot and therefore a different fingerprint. The collection timestamp and raw Parquet file should be retained with the submission.
