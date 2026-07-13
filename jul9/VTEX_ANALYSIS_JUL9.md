# July 9, 2026 — VTEX Propagation Analysis Reference

> Generated 2026-07-13. Extends JUL9_COMPLETE_REFERENCE.md with VTEX SQS event layer.
> Answers: "What was VTEX telling the customer at the moment they ordered?"
> All stats use **Jul 9 CLT events only** (00:00-23:59 CLT).

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events (Jul 9)** | `analysis/vtex_events/jul09/vtex_jul09.jsonl` | 618,634 total (505,634 Jul 9 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **Preventable parquet** | `analysis/jul9/delay_proof_0709_preventable.parquet` | 17,261 rows | All HANA<=0 order lines (from JUL9 analysis) |
| **Enriched output** | `analysis/jul9/preventable_with_vtex_jul9.csv` | 17,261 rows | Every line enriched with Jul 9 VTEX state |

### Jul 9 VTEX event stats (CLT day only)

| Metric | Value |
|--------|-------|
| Total events | 505,634 |
| jumboclj store events | 502,045 |
| OOS events (`outOfStock=true`) | 111,906 (22.3%) |
| IN_STOCK events | 390,139 (77.7%) |
| Distinct VTEX skuIds | 20,754 |
| Distinct (skuId, storeId) combos | 240,454 |
| Distinct stores | 40 |
| Mapping coverage | 20,728/20,754 (99.9%) |
| CLT range | 00:00:15 → 18:29:59 |

**Jul 9 is the organic baseline.** At 502K events, it has 4.5x fewer than Jul 7 (2.2M) and 3x fewer than Jul 8 (1.5M). No PUBLICADOR batch spike. This is the natural VTEX event rate — roughly 500K events/day from organic stock changes only.

---

## 2. Core Analysis: VTEX State at Order Time

### Results

| VTEX state at order time | Lines | % | CLP |
|---|---|---|---|
| **IN_STOCK** | **12,586** | **71.3%** | **50,849,282** |
| NO_PRIOR_EVENT | 4,098 | 23.2% | 27,326,138 |
| NO_VTEX_EVENT | 899 | 5.1% | 6,934,288 |
| OOS | 51 | 0.3% | 298,372 |
| NO_MAPPING | 11 | 0.1% | 65,287 |

**71.3% of the time, VTEX confirmed "this item is available" while HANA already knew stock was at zero.**

This is the lowest IN_STOCK rate across all analyzed days. Without PUBLICADOR pushing batch events, fewer items had a recent VTEX update — 23.2% fall into NO_PRIOR_EVENT (vs 8.6% on Jul 7). The website was likely showing stale IN_STOCK state from a prior day for these 4,098 lines.

### Breakdown by picker status

| Status | Meaning | Total | VTEX IN_STOCK | VTEX OOS | NO_PRIOR | NO_VTEX |
|--------|---------|-------|---------------|----------|----------|---------|
| 1 | Picker FOUND it (false alarm) | 14,092 | 10,422 (74.0%) | 30 | 3,056 | 584 |
| 5 | Picker NOT FOUND (Pop A) | 1,522 | **789** (51.8%) | 4 | 601 | 128 |
| 4 | Substituted | 1,011 | 639 (63.2%) | 3 | 299 | 70 |
| 13 | Weight item OOS | 396 | 332 (83.8%) | 1 | 59 | 4 |
| 11 | Added item | 388 | 220 (56.7%) | 13 | 53 | 102 |
| 12 | Partial weight pick | 224 | 183 (81.7%) | 0 | 30 | 11 |
| 14 | Sin stock (sala) | 1 | 1 | 0 | 0 | 0 |

### The preventable core: Pop A with VTEX IN_STOCK

```
1,523 Pop A lines
|- VTEX said IN_STOCK at order time:     790 (51.8%)  <- CONFIRMED PREVENTABLE
|- VTEX had NO_PRIOR_EVENT:              601 (39.5%)  <- likely also IN_STOCK (stale from prior day)
|- VTEX had NO_VTEX_EVENT:              128 ( 8.4%)  <- no Jul 9 data
|- VTEX said OOS:                          4 ( 0.3%)  <- customer ordered despite OOS
```

**790 lines where all three sources agree: HANA knew, VTEX lied, picker confirmed.**

Jul 9's Pop A VTEX-confirmed count (790) is the lowest across all days — not because fewer stockouts occurred, but because without PUBLICADOR, fewer items had a provable Jul 9 VTEX IN_STOCK event. The 601 NO_PRIOR_EVENT lines were almost certainly also showing IN_STOCK (stale state from prior day), bringing the effective total to ~1,391 (91.3%).

---

## 3. Enriched Output File

**File:** `analysis/jul9/preventable_with_vtex_jul9.csv`
**Rows:** 17,261
**Columns:** Same as Jul 6 — see VTEX_ANALYSIS_JUL6.md Section 6 for full column reference.

Key columns: `vtex_state_at_order`, `vtex_last_event_utc`, `vtex_last_event_chile`, `vtex_events_count_jul9`, `vtex_all_events_jul9`

---

## 4. Cross-Day Comparison

| VTEX state at order | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| **IN_STOCK** | 14,875 (77.3%) | 16,149 (88.7%) | 13,823 (86.8%) | **12,586 (71.3%)** |
| NO_PRIOR_EVENT | 4,025 (20.9%) | 1,563 (8.6%) | 1,662 (10.4%) | **4,098 (23.2%)** |
| NO_VTEX_EVENT | 261 (1.4%) | 355 (1.9%) | 319 (2.0%) | **899 (5.1%)** |
| OOS | 77 (0.4%) | 139 (0.8%) | 105 (0.7%) | 51 (0.3%) |
| **Total** | **19,241** | **17,680** | **15,551** | **17,261** |

| Pop A + VTEX IN_STOCK | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| Lines | 1,769 | 1,687 | 1,157 | **790** |
| CLP | ~9.4M | 9.2M | 6.7M | **4.5M** |

### The PUBLICADOR effect on provability

| Metric | Jul 7 (PUBLICADOR) | Jul 9 (organic) | Ratio |
|--------|-------------------|-----------------|-------|
| VTEX events | 2,244,877 | 502,045 | 4.5x |
| Pop A IN_STOCK provable | 1,687 (82.3%) | 790 (51.8%) | 2.1x |
| Pop A NO_PRIOR_EVENT | 295 (14.4%) | 601 (39.5%) | 0.5x |

Without PUBLICADOR, 39.5% of Pop A lines become unprovable from same-day VTEX data alone — the website was showing state set on a prior day. The stockout still happened; we just can't prove what the customer saw from Jul 9 events alone.

---

## 5. The Three-Source Verdict

### The failure chain (Pop A, 790 VTEX-confirmed + ~601 likely preventable):

```
1. HANA batch arrives    -> qty = 0 for this item+store
2. [NOTHING HAPPENS]     -> no signal sent to VTEX
3. VTEX still shows      -> "in stock" on jumbo.cl (confirmed for 790, likely for 601 more)
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
| **NO_PRIOR_EVENT dominates** | 23.2% of lines (vs 8.6% on Jul 7). Without PUBLICADOR, many items had no Jul 9 VTEX event before the order. |
| **Lower provability** | Only 51.8% of Pop A lines have a confirmed Jul 9 VTEX IN_STOCK event (vs 82.3% on Jul 7). The other 39.5% were almost certainly also IN_STOCK but can't be proven from Jul 9 data alone. |
| **Mapping imperfect** | 11 lines with NO_MAPPING. |
