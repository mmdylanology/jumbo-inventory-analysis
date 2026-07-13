# July 9, 2026 — VTEX Propagation & Delay Analysis Reference

> Generated 2026-07-13 by `scripts/vtex_crossref.py 0709`.
> Extends JUL9_COMPLETE_REFERENCE.md with VTEX SQS event layer + HANA-VTEX delay classification.

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events** | `analysis/vtex_events/jul09/vtex_jul09.jsonl` | 618,634 total (505,634 Jul 9 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **HANA parquets** | `data_samples/vw_daily_nrt_0709_*/*.parquet` | ~541M rows / 24 batches | 00:41–23:41 CLT |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0709_FULL.csv` | ~370K lines | From Redshift `io_items` |
| **Delay analysis output** | `analysis/jul9/preventable_with_vtex_jul9.csv` | 20,963,429 rows | All (item,store) pairs with HANA zeros |

### VTEX event window

```
CLT range: 2026-07-09 00:00:15 → 2026-07-09 18:29:59
Events cut off at ~18:30 CLT (SQS drain window limit)
```

---

## 2. VTEX Event Stats (Jul 9 CLT only)

| Metric | Value |
|--------|-------|
| Total events (in file) | 618,634 |
| Events in Jul 9 CLT window | 505,634 |
| jumboclj store events | 502,045 |
| OOS events (`outOfStock=true`) | 111,906 (22.3%) |
| IN_STOCK events | 390,139 (77.7%) |
| Distinct VTEX skuIds | 20,754 |
| Distinct (skuId, storeId) combos | 240,454 |
| Distinct stores | 40 |
| Mapping coverage | 20,728/20,754 (99.9%) |

**Jul 9 is the organic baseline.** At 502K events, it has 4.5x fewer events than Jul 7 (2.2M) and 3x fewer than Jul 8 (1.5M). No PUBLICADOR batch spike visible. This reveals the natural VTEX event rate without batch interference — roughly 500-680K events/day from organic stock changes.

Only 22.3% of events are OOS signals (vs 46.8% on Jul 7), suggesting PUBLICADOR disproportionately pushes OOS state changes.

---

## 3. Part 1 — Preventable Events (HANA <= 0 before order)

**File:** `analysis/jul9/preventable_events_jul9.csv`
**Rows:** 17,261

| Metric | Value |
|--------|-------|
| Preventable events | 17,261 |
| Distinct orders | 10,464 |
| Total CLP | 84,100,246 |
| Gap min/avg/max | 0 / 30 / 62 min |

### Status breakdown

| Status | Meaning | Lines | CLP |
|--------|---------|-------|-----|
| 1 | **Picker FOUND it** (false alarm) | 13,819 | 60,159,157 |
| 5 | **Picker NOT FOUND** (Pop A) | 1,485 | 9,297,004 |
| 4 | Substituted | 956 | 5,937,336 |
| 13 | Weight item OOS | 389 | 5,280,278 |
| 11 | Added item | 389 | 1,661,673 |
| 12 | Partial weight pick | 222 | 1,760,808 |
| 14 | Sin stock (sala) | 1 | 3,990 |

---

## 4. Part 2 — VTEX Delay Classification

**File:** `analysis/jul9/preventable_with_vtex_jul9.csv`
**Rows:** 20,963,429

### Case Breakdown — Jul 9

| Case | Count | % | Interpretation |
|------|-------|---|----------------|
| **C_SILENT_OOS** | 20,878,410 | 99.6% | Perpetual zero-stock: 20,854,235 have >=20 zero batches. Noise. |
| **A_SLOW_DETECTION** | 84,228 | 0.4% | VTEX eventually detected OOS but after multiple HANA cycles |
| **D_FULL_CYCLE** | 462 | 0.002% | Complete zero→restock cycle tracked by both systems |
| **B_RESTOCK_LAG** | 329 | 0.002% | HANA restocked but VTEX lagged behind |
| **A_DETECTED** | 0 | 0% | No items had <2 zero batches with VTEX detection |

### Delay Statistics

| Case | Metric | Value |
|------|--------|-------|
| **A_SLOW_DETECTION** | avg delay (first_zero→VTEX OOS) | 348 min |
| | median delay | 355 min |
| | p95 delay | 385 min |
| | avg missed HANA cycles | 5.3 |
| | avg zero batches | 23.8 |
| **D_FULL_CYCLE** | avg restock lag | 64 min |
| | median restock lag | 68 min |
| | avg detection delay | 184 min |
| **C_SILENT_OOS** | avg zero batches | 24.0 |
| | full day (>=20 batches) | 20,854,235 (99.9%) |
| | with any HANA restock | 12,208 |

### The Jul 9 anomaly

Jul 9's dramatically lower VTEX event volume (502K vs 2.2M on Jul 7) cascades through the delay analysis:

| Impact | Jul 7 | Jul 9 | Ratio |
|--------|-------|-------|-------|
| VTEX events | 2,244,877 | 502,045 | 4.5x fewer |
| A_SLOW_DETECTION | 271,981 | 84,228 | 3.2x fewer |
| D_FULL_CYCLE | 2,232 | 462 | 4.8x fewer |
| B_RESTOCK_LAG | 1,113 | 329 | 3.4x fewer |
| C_SILENT_OOS | 20,671,002 | 20,878,410 | ~same |

Without PUBLICADOR pushing batch events, far fewer items ever get a VTEX OOS signal at all — so they fall into C_SILENT_OOS instead. The A_SLOW_DETECTION p95 drops from 814→385 min because the items that DO get detected have a tighter delay distribution.

This strongly suggests that PUBLICADOR is the primary mechanism through which VTEX learns about stock state changes. On days without it, VTEX is largely blind to HANA stock movements.

### HANA Summary

| Metric | Value |
|--------|-------|
| Total (item,store) pairs with zeros | 20,963,429 |
| Pairs that restocked during the day | 12,999 (0.06%) |

---

## 5. Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul9/preventable_with_vtex_jul9.csv` | 20,963,429 | Part 2: All (item,store) pairs with delay classification |
| `analysis/jul9/preventable_events_jul9.csv` | 17,261 | Part 1: All HANA<=0 order lines |
| `analysis/jul9/restock_lag_jul9.csv` | 329 | Part 2 subset: B_RESTOCK_LAG cases |
| `analysis/jul9/silent_oos_jul9.csv` | 20,878,410 | Part 2 subset: C_SILENT_OOS cases |
| `analysis/jul9/vtex_crossref_jul9.xlsx` | 6 sheets | Summary + active cases + case breakdown |
| `analysis/jul9/delay_proof_0709_preventable.parquet` | 17,261 | Checkpoint: all HANA<=0 order lines (Parquet) |

---

## 6. Scripts Reference

| Script | What it does |
|--------|-------------|
| `scripts/vtex_crossref.py 0709` | Parameterized VTEX cross-ref (Part 1 + Part 2) |
| `scripts/delay_proof_0709.py` | HANA lookup for all orders (original Layer 1) |
| `scripts/evidence_pack_0709.py` | Pop A/B + confirmation tiers |
