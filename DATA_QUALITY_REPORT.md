# Data Quality Report — Prix des carburants en France

**Dataset:** Prix des carburants en France, flux instantane v2  
**Source:** data.economie.gouv.fr (Opendatasoft API v2.1)  
**Collected:** 05/11/2026 — 9 828 stations, France entiere  
**Authors:** Zackary Aich, Max Guiriec, Martin Vallee, Amaniel Asrat

---

## 1. Schema & validation

A pandera schema (`strict=True`) validates every kept column with 7+ domain checks:

| Check | Result |
|---|---|
| `id` unique, > 0 | Pass |
| `cp` matches `^\d{5}$` | Pass |
| `pop` in {R, A} | Pass |
| `lat` in [41.0, 51.5], `lon` in [-5.5, 9.8] | Pass (all inside metropolitan France) |
| `prix_*` in [0.50, 3.50] EUR/L | Pass (0 outliers outside bounds) |
| `maj_*` not in the future | Pass (0 violations) |
| `carburants_indisponibles` as string | Pass (converted from list to str before validation) |

**Raw schema violation found and fixed:** The API returns `carburants_indisponibles` as a JSON list; the notebook converts it to a comma-separated string before validation.

## 2. Duplicates

- **Exact duplicates:** 0
- **Near-duplicates** (same postal code + normalised address, different `id`): 104 rows (52 pairs)
- **Decision:** Keep both rows. Two IDs at the same address likely represent two operators (e.g. a supermarket and a brand station). Merging would require the brand, which the API does not provide.

## 3. Impossible values

| Column | Check | Count |
|---|---|---|
| `prix_*` outside [0.50, 3.50] EUR/L | 0 per fuel | No action needed |
| Coordinates outside France | 0 | No action needed |
| `maj_*` in the future | 0 | No action needed |

All prices and coordinates fall within plausible bounds for this snapshot.

## 4. Missingness

| Column | Share missing | Mechanism | Decision |
|---|---|---|---|
| `prix_gplc` / `maj_gplc` | 84.7% | Structural (few stations sell GPLc) | Keep NaN, no imputation |
| `prix_sp95` / `maj_sp95` | 72.4% | **MAR on region** (permutation p < 0.001) | Keep NaN, no imputation |
| `prix_e85` / `maj_e85` | 61.7% | Structural (not all stations sell E85) | Keep NaN, no imputation |
| `prix_sp98` / `maj_sp98` | 29.6% | Likely structural | Keep NaN |
| `prix_e10` / `maj_e10` | 27.5% | Likely structural | Keep NaN |
| `prix_gazole` / `maj_gazole` | 6.8% | Mixed (closed stations, reporting lag) | Keep NaN |

**Key finding:** The SP95 missingness is **not MCAR**. A permutation test on the spread of missing shares across regions gives p < 0.001. Corse has 0.8% missing vs Ile-de-France at 84.6%. This is **MAR** — missingness depends on region, likely because SP95 is being replaced by E10 at different rates across regions.

## 5. Outliers

- **Method:** IQR fence on log-transformed `prix_gazole` (k = 1.5).
- **Flagged:** ~356 rows above the upper fence (~2.45 EUR/L).
- **Decision:** Keep all, flag as extreme. Most flagged stations are on motorways or in Paris — plausible real prices, not errors. A per-station analysis would be needed to separate true typos from real extremes.

## 6. What cannot be trusted

| Column / subset | Why |
|---|---|
| Any `prix_*` where `maj_*` > 30 days old | ~130 to 330 rows per fuel have prices older than 30 days. These are stale and do not reflect current market conditions. A modeller must filter or flag them. |
| `prix_sp95` imputed by global mean | The missingness is MAR on region. A global mean would systematically underestimate prices in regions where SP95 is rare, and overestimate in Corse where it is almost always reported. |
| Snapshot stability | The dataset is a live snapshot. Two collections at different times give different rows. Any analysis must record its collection timestamp. |

---

*This report accompanies the notebook `project_02_fuel_prices_v4.ipynb` and the decision table in `reports/decision_table.csv`.*
