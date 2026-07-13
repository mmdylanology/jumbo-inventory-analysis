# July 8, 2026 — VTEX Propagation & Delay Analysis Reference

> Generated 2026-07-13 by `scripts/vtex_crossref.py 0708`.
> Extends JUL8_COMPLETE_REFERENCE.md with VTEX SQS event layer + HANA-VTEX delay classification.

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events** | `analysis/vtex_events/jul08/vtex_jul08.jsonl` | 2,267,994 total (1,540,475 Jul 8 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **HANA parquets** | `data_samples/vw_daily_nrt_0708_*/*.parquet` | ~541M rows / 24 batches | 00:41–23:41 CLT |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0708_FULL.csv` | ~370K lines | From Redshift `io_items` |
| **Delay analysis output** | `analysis/jul8/preventable_with_vtex_jul8.csv` | 20,961,125 rows | All (item,store) pairs with HANA zeros |

### VTEX event window

```
CLT range: 2026-07-08 00:00:01 → 2026-07-08 18:29:59
Events cut off at ~18:30 CLT (SQS drain window limit)
```

---

## 2. VTEX Event Stats (Jul 8 CLT only)

| Metric | Value |
|--------|-------|
| Total events (in file) | 2,267,994 |
| Events in Jul 8 CLT window | 1,540,475 |
| jumboclj store events | 1,510,177 |
| OOS events (`outOfStock=true`) | 446,624 (29.6%) |
| IN_STOCK events | 1,063,553 (70.4%) |
| Distinct VTEX skuIds | 25,511 |
| Distinct (skuId, storeId) combos | 775,482 |
| Distinct stores | 40 |
| Mapping coverage | 25,477/25,511 (99.9%) |

Jul 8 has moderate event volume — less than Jul 7 (2.2M→1.5M) but still includes a PUBLICADOR batch spike. OOS share dropped from 46.8% to 29.6%.

---

## 3. Part 1 — Preventable Events (HANA <= 0 before order)

**File:** `analysis/jul8/preventable_events_jul8.csv`
**Rows:** 15,551

| Metric | Value |
|--------|-------|
| Preventable events | 15,551 |
| Distinct orders | 9,889 |
| Total CLP | 75,166,442 |
| Gap min/avg/max | 0 / 30 / 61 min |

### Status breakdown

| Status | Meaning | Lines | CLP |
|--------|---------|-------|-----|
| 1 | **Picker FOUND it** (false alarm) | 12,246 | 52,388,751 |
| 5 | **Picker NOT FOUND** (Pop A) | 1,400 | 8,279,194 |
| 4 | Substituted | 961 | 5,867,759 |
| 11 | Added item | 403 | 1,749,682 |
| 13 | Weight item OOS | 350 | 5,772,696 |
| 12 | Partial weight pick | 189 | 1,106,082 |
| 14 | Sin stock (sala) | 1 | 1,780 |
| 3 | Sin stock (legacy) | 1 | 498 |

Jul 8 has the fewest Pop A lines (1,400) and lowest OOS CLP (8.28M) of the four days analyzed — the best day for stockout performance.

---

## 4. Part 2 — VTEX Delay Classification

**File:** `analysis/jul8/preventable_with_vtex_jul8.csv`
**Rows:** 20,961,125

### Case Breakdown — Jul 8

| Case | Count | % | Interpretation |
|------|-------|---|----------------|
| **C_SILENT_OOS** | 20,771,542 | 99.1% | Perpetual zero-stock: 20,740,488 have >=20 zero batches. Noise. |
| **A_SLOW_DETECTION** | 185,169 | 0.9% | VTEX eventually detected OOS but after multiple HANA cycles |
| **D_FULL_CYCLE** | 3,007 | 0.01% | Complete zero→restock cycle tracked by both systems |
| **B_RESTOCK_LAG** | 1,407 | 0.01% | HANA restocked but VTEX lagged behind |
| **A_DETECTED** | 0 | 0% | No items had <2 zero batches with VTEX detection |

### Delay Statistics

| Case | Metric | Value |
|------|--------|-------|
| **A_SLOW_DETECTION** | avg delay (first_zero→VTEX OOS) | 296 min |
| | median delay | 243 min |
| | p95 delay | 751 min |
| | avg missed HANA cycles | 4.6 |
| | avg zero batches | 23.8 |
| **D_FULL_CYCLE** | avg restock lag | 67 min |
| | median restock lag | 68 min |
| | avg detection delay | 221 min |
| **C_SILENT_OOS** | avg zero batches | 24.0 |
| | full day (>=20 batches) | 20,740,488 (99.9%) |
| | with any HANA restock | 9,185 |

Jul 8 shows improved VTEX detection speed vs Jul 7: A_SLOW_DETECTION avg delay dropped from 401→296 min, and missed cycles from 6.3→4.6. But Jul 8 also had more D_FULL_CYCLE (3,007 vs 2,232) and B_RESTOCK_LAG (1,407 vs 1,113) cases — more items went through the zero→restock cycle.

### HANA Summary

| Metric | Value |
|--------|-------|
| Total (item,store) pairs with zeros | 20,961,125 |
| Pairs that restocked during the day | 13,599 (0.06%) |

---

## 5. Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul8/preventable_with_vtex_jul8.csv` | 20,961,125 | Part 2: All (item,store) pairs with delay classification |
| `analysis/jul8/preventable_events_jul8.csv` | 15,551 | Part 1: All HANA<=0 order lines |
| `analysis/jul8/restock_lag_jul8.csv` | 1,407 | Part 2 subset: B_RESTOCK_LAG cases |
| `analysis/jul8/silent_oos_jul8.csv` | 20,771,542 | Part 2 subset: C_SILENT_OOS cases |
| `analysis/jul8/vtex_crossref_jul8.xlsx` | 6 sheets | Summary + active cases + case breakdown |
| `analysis/jul8/delay_proof_0708_preventable.parquet` | 15,551 | Checkpoint: all HANA<=0 order lines (Parquet) |

---

## 6. Scripts Reference

| Script | What it does |
|--------|-------------|
| `scripts/vtex_crossref.py 0708` | Parameterized VTEX cross-ref (Part 1 + Part 2) |
| `scripts/delay_proof_0708.py` | HANA lookup for all orders (original Layer 1) |
| `scripts/evidence_pack_0708.py` | Pop A/B + confirmation tiers |
