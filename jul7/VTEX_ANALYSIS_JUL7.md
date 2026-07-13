# July 7, 2026 — VTEX Propagation & Delay Analysis Reference

> Generated 2026-07-13 by `scripts/vtex_crossref.py 0707`.
> Extends JUL7_COMPLETE_REFERENCE.md with VTEX SQS event layer + HANA-VTEX delay classification.

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events** | `analysis/vtex_events/jul07/vtex_jul07.jsonl` | 2,401,483 total (2,293,460 Jul 7 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **HANA parquets** | `data_samples/vw_daily_nrt_0707_*/*.parquet` | 540,957,044 rows / 24 batches | 00:41–23:41 CLT |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0707_FULL.csv` | 371,553 lines | From Redshift `io_items` |
| **Delay analysis output** | `analysis/jul7/preventable_with_vtex_jul7.csv` | 20,946,328 rows | All (item,store) pairs with HANA zeros |

### VTEX event window

```
CLT range: 2026-07-07 00:00:02 → 2026-07-07 18:29:59
Events cut off at ~18:30 CLT (SQS drain window limit)
```

---

## 2. VTEX Event Stats (Jul 7 CLT only)

| Metric | Value |
|--------|-------|
| Total events (in file) | 2,401,483 |
| Events in Jul 7 CLT window | 2,293,460 |
| jumboclj store events | 2,244,877 |
| OOS events (`outOfStock=true`) | 1,050,299 (46.8%) |
| IN_STOCK events | 1,194,578 (53.2%) |
| Distinct VTEX skuIds | 29,396 |
| Distinct (skuId, storeId) combos | 861,363 |
| Distinct stores | 40 |
| Mapping coverage | 29,382/29,396 (100.0%) |

Jul 7 has the highest event volume of the three days analyzed (Jul 7/8/9), with a strong PUBLICADOR batch spike visible in the data. Nearly 47% of events are OOS signals.

---

## 3. Part 1 — Preventable Events (HANA <= 0 before order)

**File:** `analysis/jul7/preventable_events_jul7.csv`
**Rows:** 17,680

For each order line, the script finds the most recent HANA batch before `fecha_creacion`. If `Ending_On_Hand_Qty <= 0`, the event is preventable — HANA already knew stock was zero.

| Metric | Value |
|--------|-------|
| Preventable events | 17,680 |
| Distinct orders | 10,662 |
| Total CLP | 82,892,266 |
| Gap min/avg/max | 0 / 30 / 61 min |

### Status breakdown (what happened at picking)

| Status | Meaning | Lines | CLP |
|--------|---------|-------|-----|
| 1 | **Picker FOUND it** (false alarm) | 13,408 | 56,140,183 |
| 5 | **Picker NOT FOUND** (Pop A) | 1,978 | 11,570,805 |
| 4 | Substituted | 1,237 | 6,212,600 |
| 13 | Weight item OOS | 439 | 6,066,852 |
| 11 | Added item | 397 | 1,616,013 |
| 12 | Partial weight pick | 221 | 1,285,813 |

Note: This Part 1 output does NOT include `vtex_state_at_order` enrichment (unlike Jul 6's `preventable_with_vtex_jul6.csv` which added that column). Part 1 here is equivalent to the `delay_proof_0707.py` output.

---

## 4. Part 2 — VTEX Delay Classification

**File:** `analysis/jul7/preventable_with_vtex_jul7.csv`
**Rows:** 20,946,328 (all item-store pairs with at least one HANA zero batch)

For each (Item_Id, Location_Id) pair where HANA recorded stock <= 0:
1. Build HANA zero timeline (first zero, last zero, restock time)
2. Build VTEX OOS event timeline (first OOS event, last OOS, restock event)
3. Classify into delay case

### Case Definitions

| Case | Definition | Meaning |
|------|-----------|---------|
| **A_DETECTED** | VTEX OOS event exists, <2 zero batches, no restock | VTEX detected quickly (within 1 HANA cycle) |
| **A_SLOW_DETECTION** | VTEX OOS event exists, >=2 zero batches, no restock | VTEX detected but late (items sat at zero for multiple HANA batches before VTEX signaled) |
| **B_RESTOCK_LAG** | HANA restocked but VTEX restock event >60min late or missing | HANA shows stock back, VTEX still showing OOS |
| **C_SILENT_OOS** | No VTEX OOS event at all | HANA at zero the whole time, VTEX never sent an OOS signal. Overwhelmingly perpetual zero-stock items (not sold at that store). Noise. |
| **D_FULL_CYCLE** | HANA restocked AND VTEX restocked (both within 60min) | Full zero→restock cycle with VTEX events on both sides |

### Case Breakdown — Jul 7

| Case | Count | % | Interpretation |
|------|-------|---|----------------|
| **C_SILENT_OOS** | 20,671,002 | 98.7% | Perpetual zero-stock: 20,652,746 have >=20 zero batches (full day). Noise. |
| **A_SLOW_DETECTION** | 271,981 | 1.3% | VTEX eventually detected the OOS but after multiple HANA cycles |
| **D_FULL_CYCLE** | 2,232 | 0.01% | Complete zero→restock cycle tracked by both systems |
| **B_RESTOCK_LAG** | 1,113 | 0.01% | HANA restocked but VTEX lagged behind |
| **A_DETECTED** | 0 | 0% | No items had <2 zero batches with VTEX detection |

### Delay Statistics

| Case | Metric | Value |
|------|--------|-------|
| **A_SLOW_DETECTION** | avg delay (first_zero→VTEX OOS) | 401 min |
| | median delay | 266 min |
| | p95 delay | 814 min |
| | avg missed HANA cycles | 6.3 |
| | avg zero batches | 23.9 |
| **D_FULL_CYCLE** | avg restock lag (HANA restock→VTEX restock) | 67 min |
| | median restock lag | 68 min |
| | avg detection delay | 138 min |
| **C_SILENT_OOS** | avg zero batches | 24.0 |
| | full day (>=20 batches) | 20,652,746 (99.9%) |
| | with any HANA restock | 9,504 |

### HANA Summary

| Metric | Value |
|--------|-------|
| Total (item,store) pairs with zeros | 20,946,328 |
| Pairs that restocked during the day | 12,849 (0.06%) |

---

## 5. Comparison: Jul 7 vs Jul 8 vs Jul 9

### VTEX Event Volume

| Metric | Jul 7 | Jul 8 | Jul 9 |
|--------|-------|-------|-------|
| Events in file | 2,401,483 | 2,267,994 | 618,634 |
| Jul DD CLT events | 2,293,460 | 1,540,475 | 505,634 |
| jumboclj events | 2,244,877 | 1,510,177 | 502,045 |
| OOS events | 1,050,299 | 446,624 | 111,906 |
| OOS % | 46.8% | 29.6% | 22.3% |
| Distinct combos | 861,363 | 775,482 | 240,454 |

Jul 9 has dramatically fewer events (502K vs 2.2M) — no PUBLICADOR batch spike. This is the organic baseline.

### Delay Classification

| Case | Jul 7 | Jul 8 | Jul 9 |
|------|-------|-------|-------|
| C_SILENT_OOS | 20,671,002 (98.7%) | 20,771,542 (99.1%) | 20,878,410 (99.6%) |
| A_SLOW_DETECTION | 271,981 | 185,169 | 84,228 |
| D_FULL_CYCLE | 2,232 | 3,007 | 462 |
| B_RESTOCK_LAG | 1,113 | 1,407 | 329 |
| **Total** | **20,946,328** | **20,961,125** | **20,963,429** |

Jul 9's low VTEX event volume means fewer items get classified as A_SLOW_DETECTION (only 84K vs 272K on Jul 7). Without PUBLICADOR pushing batch events, fewer OOS signals reach VTEX at all.

### Preventable Events (Part 1)

| Metric | Jul 7 | Jul 8 | Jul 9 |
|--------|-------|-------|-------|
| HANA<=0 events | 17,680 | 15,551 | 17,261 |
| Distinct orders | 10,662 | 9,889 | 10,464 |
| Pop A (st 3/5/7/14) | 1,978 | 1,402 | 1,486 |
| Pop A CLP | 11.57M | 8.28M | 9.30M |
| False alarms (st 1) | 13,408 | 12,246 | 13,819 |
| Gap avg/max | 30/61 min | 30/61 min | 30/62 min |

---

## 6. Key Differences from Jul 6 Analysis

| Aspect | Jul 6 (`proof_0706_with_vtex.py`) | Jul 7/8/9 (`vtex_crossref.py`) |
|--------|----------------------------------|-------------------------------|
| **Part 1 output** | `preventable_with_vtex_jul6.csv` — 19,241 rows with `vtex_state_at_order` column | `preventable_events_jul7.csv` — 17,680 rows, raw preventable events only (no VTEX state enrichment) |
| **Part 2** | Not performed | A/B/C/D delay classification for all 20.9M item-store pairs |
| **DuckDB mode** | In-memory | Disk-backed (handles 540M+ HANA rows) |
| **Mapping** | XLSX (originally) | CSV (`vtex_sap_mapping.csv`) — better coverage |
| **CSV handling** | Direct type inference | `all_varchar=true` + `TRY_CAST` (bulletproof) |

The Jul 6 analysis answered: **"What was VTEX showing when the customer ordered?"** (per order line).
The Jul 7/8/9 analysis answers: **"How fast does VTEX detect HANA zero-stock events?"** (per item-store pair).

Both are complementary. The Jul 6 approach is more directly actionable (ties to specific orders + CLP lost). The Jul 7/8/9 approach reveals systemic delay patterns across the entire catalog.

---

## 7. Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul7/preventable_with_vtex_jul7.csv` | 20,946,328 | Part 2: All (item,store) pairs with HANA zeros, classified by delay case |
| `analysis/jul7/preventable_events_jul7.csv` | 17,680 | Part 1: All HANA<=0 order lines |
| `analysis/jul7/restock_lag_jul7.csv` | 1,113 | Part 2 subset: B_RESTOCK_LAG cases |
| `analysis/jul7/silent_oos_jul7.csv` | 20,671,002 | Part 2 subset: C_SILENT_OOS cases |
| `analysis/jul7/vtex_crossref_jul7.xlsx` | 6 sheets | Summary + active cases + case breakdown |
| `analysis/jul7/delay_proof_0707_preventable.parquet` | 17,680 | Checkpoint: all HANA<=0 order lines (Parquet) |

---

## 8. Scripts Reference

| Script | What it does | New in Jul 7/8/9? |
|--------|-------------|-------------------|
| `scripts/vtex_crossref.py` | Parameterized VTEX cross-ref (Part 1 + Part 2) | YES — generalized from `proof_0706_with_vtex.py` |
| `scripts/delay_proof_0707.py` | HANA lookup for all orders (original Layer 1) | No — per-day clone |
| `scripts/evidence_pack_0707.py` | Pop A/B + confirmation tiers | No — per-day clone |
