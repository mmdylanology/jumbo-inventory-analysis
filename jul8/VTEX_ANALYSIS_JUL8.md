# July 8, 2026 — VTEX Propagation Analysis Reference

> Generated 2026-07-13. Extends JUL8_COMPLETE_REFERENCE.md with VTEX SQS event layer.
> Answers: "What was VTEX telling the customer at the moment they ordered?"
> All stats use **Jul 8 CLT events only** (00:00-23:59 CLT).

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events (Jul 8)** | `analysis/vtex_events/jul08/vtex_jul08.jsonl` | 2,267,994 total (1,540,475 Jul 8 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **Preventable parquet** | `analysis/jul8/delay_proof_0708_preventable.parquet` | 15,551 rows | All HANA<=0 order lines (from JUL8 analysis) |
| **Enriched output** | `analysis/jul8/preventable_with_vtex_jul8.csv` | 15,551 rows | Every line enriched with Jul 8 VTEX state |

### Jul 8 VTEX event stats (CLT day only)

| Metric | Value |
|--------|-------|
| Total events | 1,540,475 |
| jumboclj store events | 1,510,177 |
| OOS events (`outOfStock=true`) | 446,624 (29.6%) |
| IN_STOCK events | 1,063,553 (70.4%) |
| Distinct VTEX skuIds | 25,511 |
| Distinct (skuId, storeId) combos | 775,482 |
| Distinct stores | 40 |
| Mapping coverage | 25,477/25,511 (99.9%) |
| CLT range | 00:00:01 → 18:29:59 |

---

## 2. Core Analysis: VTEX State at Order Time

### Results

| VTEX state at order time | Lines | % | CLP |
|---|---|---|---|
| **IN_STOCK** | **13,823** | **86.8%** | **63,590,589** |
| NO_PRIOR_EVENT | 1,662 | 10.4% | 10,156,446 |
| NO_VTEX_EVENT | 319 | 2.0% | 1,988,602 |
| OOS | 105 | 0.7% | 582,596 |
| NO_MAPPING | 7 | 0.0% | 37,854 |

**86.8% of the time, VTEX confirmed "this item is available" while HANA already knew stock was at zero.**

### Breakdown by picker status

| Status | Meaning | Total | VTEX IN_STOCK | VTEX OOS | NO_PRIOR | NO_VTEX |
|--------|---------|-------|---------------|----------|----------|---------|
| 1 | Picker FOUND it (false alarm) | 12,497 | 11,054 (88.5%) | 14 | 1,233 | 196 |
| 5 | Picker NOT FOUND (Pop A) | 1,436 | **1,156** (80.5%) | 7 | 239 | 33 |
| 4 | Substituted | 1,019 | 859 (84.3%) | 2 | 139 | 19 |
| 13 | Weight item OOS | 351 | 328 (93.4%) | 2 | 20 | 1 |
| 11 | Added item | 410 | 247 (60.2%) | 80 | 14 | 69 |
| 12 | Partial weight pick | 195 | 178 (91.3%) | 0 | 16 | 1 |
| 14 | Sin stock (sala) | 1 | 0 | 0 | 1 | 0 |
| 3 | Sin stock (legacy) | 1 | 1 | 0 | 0 | 0 |

### The preventable core: Pop A with VTEX IN_STOCK

```
1,437 Pop A lines
|- VTEX said IN_STOCK at order time:   1,157 (80.5%)  <- CONFIRMED PREVENTABLE
|- VTEX had NO_PRIOR_EVENT:              240 (16.7%)  <- likely also IN_STOCK
|- VTEX had NO_VTEX_EVENT:               33 ( 2.3%)  <- no Jul 8 data
|- VTEX said OOS:                          7 ( 0.5%)  <- customer ordered despite OOS
```

**1,157 lines where all three sources agree: HANA knew, VTEX lied, picker confirmed.**

Jul 8 has the best stockout performance of all analyzed days — fewest Pop A lines (1,437) and lowest VTEX-confirmed preventable count (1,157).

---

## 3. Enriched Output File

**File:** `analysis/jul8/preventable_with_vtex_jul8.csv`
**Rows:** 15,551
**Columns:** Same as Jul 6 — see VTEX_ANALYSIS_JUL6.md Section 6 for full column reference.

Key columns: `vtex_state_at_order`, `vtex_last_event_utc`, `vtex_last_event_chile`, `vtex_events_count_jul8`, `vtex_all_events_jul8`

---

## 4. Cross-Day Comparison

| VTEX state at order | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| **IN_STOCK** | 14,875 (77.3%) | 16,149 (88.7%) | **13,823 (86.8%)** | 12,586 (71.3%) |
| NO_PRIOR_EVENT | 4,025 (20.9%) | 1,563 (8.6%) | 1,662 (10.4%) | 4,098 (23.2%) |
| NO_VTEX_EVENT | 261 (1.4%) | 355 (1.9%) | 319 (2.0%) | 899 (5.1%) |
| OOS | 77 (0.4%) | 139 (0.8%) | 105 (0.7%) | 51 (0.3%) |
| **Total** | **19,241** | **17,680** | **15,551** | **17,261** |

| Pop A + VTEX IN_STOCK | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| Lines | 1,769 | 1,687 | **1,157** | 790 |
| CLP | ~9.4M | 9.2M | **6.7M** | 4.5M |

---

## 5. The Three-Source Verdict

### The failure chain (Pop A, 1,157 VTEX-confirmed preventable lines):

```
1. HANA batch arrives    -> qty = 0 for this item+store
2. [NOTHING HAPPENS]     -> no signal sent to VTEX
3. VTEX still shows      -> "in stock" on jumbo.cl
4. Customer sees         -> "in stock", places order
5. Picker walks to shelf -> item not there
6. Picker marks          -> SIN STOCK
7. Customer gets         -> "your item was not available" message
```

---

## 6. Limitations

| Limitation | Impact |
|---|---|
| **VTEX events are binary** | No actual quantity — only "in stock" or "out of stock". |
| **Events cut off at ~18:30 CLT** | SQS drain window limit. |
| **NO_PRIOR_EVENT is ambiguous** | 1,662 lines with no Jul 8 event before order. |
| **PUBLICADOR contaminates** | Batch push still present on Jul 8 (1.5M events). |
| **Mapping imperfect** | 7 lines with NO_MAPPING. |
