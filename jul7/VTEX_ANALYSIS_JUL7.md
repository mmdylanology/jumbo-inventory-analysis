# July 7, 2026 — VTEX Propagation Analysis Reference

> Generated 2026-07-13. Extends JUL7_COMPLETE_REFERENCE.md with VTEX SQS event layer.
> Answers: "What was VTEX telling the customer at the moment they ordered?"
> All stats use **Jul 7 CLT events only** (00:00-23:59 CLT).

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events (Jul 7)** | `analysis/vtex_events/jul07/vtex_jul07.jsonl` | 2,401,483 total (2,293,460 Jul 7 CLT) | 1 JSONL file |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **Preventable parquet** | `analysis/jul7/delay_proof_0707_preventable.parquet` | 17,680 rows | All HANA<=0 order lines (from JUL7 analysis) |
| **Enriched output** | `analysis/jul7/preventable_with_vtex_jul7.csv` | 17,680 rows | Every line enriched with Jul 7 VTEX state |

### Jul 7 VTEX event stats (CLT day only)

| Metric | Value |
|--------|-------|
| Total events | 2,293,460 |
| jumboclj store events | 2,244,877 |
| OOS events (`outOfStock=true`) | 1,050,299 (46.8%) |
| IN_STOCK events | 1,194,578 (53.2%) |
| Distinct VTEX skuIds | 29,396 |
| Distinct (skuId, storeId) combos | 861,363 |
| Distinct stores | 40 |
| Mapping coverage | 29,382/29,396 (100.0%) |
| CLT range | 00:00:02 → 18:29:59 (events cut off at ~18:30 CLT) |

Jul 7 has the highest event volume of all analyzed days (2.2M events). Nearly 47% are OOS signals — consistent with a PUBLICADOR batch spike pushing OOS state changes.

---

## 2. Core Analysis: VTEX State at Order Time

### Method

For each of the 17,680 order lines in `delay_proof_0707_preventable.parquet`:

1. Get `fecha_creacion` (order creation time, CLT timezone)
2. Map `item_id_for_hana` → VTEX `skuId` via `vtex_sap_mapping.csv`
3. Map `id_tienda` (e.g. `J989`) → VTEX `storeId` (e.g. `jumboclj989`)
4. Build timeline of **Jul 7 CLT events only** for that (skuId, storeId)
5. Find the last event with `ts_clt` < order time
6. Read the `outOfStock` field = what VTEX was showing when the customer ordered

### Results

| VTEX state at order time | Lines | % | CLP |
|---|---|---|---|
| **IN_STOCK** | **16,149** | **88.7%** | **70,385,297** |
| NO_PRIOR_EVENT | 1,563 | 8.6% | 10,263,201 |
| NO_VTEX_EVENT | 355 | 1.9% | 3,053,125 |
| OOS | 139 | 0.8% | 778,663 |
| NO_MAPPING | 3 | 0.0% | 19,180 |

**88.7% of the time, a Jul 7 VTEX event confirmed "this item is available" while HANA already knew stock was at zero.**

This is significantly higher than Jul 6 (77.3%), likely because Jul 7's higher VTEX event volume (2.2M vs 1.4M) means more items had a recent IN_STOCK event before the order.

### Breakdown by picker status

| Status | Meaning | Total | VTEX IN_STOCK | VTEX OOS | NO_PRIOR | NO_VTEX |
|--------|---------|-------|---------------|----------|----------|---------|
| 1 | Picker FOUND it (false alarm) | 13,773 | 12,440 (90.3%) | 34 | 1,078 | 221 |
| 5 | Picker NOT FOUND (Pop A) | 2,051 | **1,687** (82.3%) | 15 | 295 | 54 |
| 4 | Substituted | 1,298 | 1,122 (86.4%) | 11 | 148 | 18 |
| 13 | Weight item OOS | 443 | 423 (95.5%) | 0 | 18 | 2 |
| 11 | Added item | 413 | 266 (64.4%) | 76 | 11 | 60 |
| 12 | Partial weight pick | 227 | 211 (92.9%) | 3 | 13 | 0 |

### The preventable core: Pop A with VTEX IN_STOCK

Of the 2,051 Pop A lines (status 3/5/7/14, HANA<=0, picker confirmed OOS):

```
2,051 Pop A lines
|- VTEX said IN_STOCK at order time:   1,687 (82.3%)  <- CONFIRMED PREVENTABLE
|- VTEX had NO_PRIOR_EVENT:              295 (14.4%)  <- likely also IN_STOCK (stale from prior day)
|- VTEX had NO_VTEX_EVENT:                54 ( 2.6%)  <- no Jul 7 data
|- VTEX said OOS:                         15 ( 0.7%)  <- customer ordered despite OOS signal
```

**1,687 lines where all three sources agree on the failure chain (Jul 7 data only):**
- HANA: "stock is zero" (correct)
- VTEX: "item is available" (wrong — didn't update, confirmed by Jul 7 event)
- Picker: "item not found" (confirmed)

---

## 3. State Definitions

| State | Definition | Lines | What happened |
|---|---|---|---|
| **IN_STOCK** | Last Jul 7 VTEX event before order time had `outOfStock=false` | 16,149 | A Jul 7 VTEX event confirmed the item was shown as available to the customer at order time. |
| **NO_PRIOR_EVENT** | Jul 7 VTEX events exist for this (sku, store), but ALL are AFTER the order time | 1,563 | No Jul 7 VTEX update had arrived before the customer ordered. Website was showing whatever state was set on a previous day. |
| **NO_VTEX_EVENT** | Zero Jul 7 VTEX events for this (sku, store) | 355 | This item-store combo had no stock change signal on Jul 7. Stale state from a prior day. |
| **OOS** | Last Jul 7 VTEX event before order time had `outOfStock=true` | 139 | VTEX correctly showed OOS, but the customer ordered anyway. Likely a cached page or race condition. |
| **NO_MAPPING** | Item has no entry in `vtex_sap_mapping.csv` | 3 | Cannot look up VTEX data — item not in `dim_surtidos`. |

---

## 4. Enriched Output File

**File:** `analysis/jul7/preventable_with_vtex_jul7.csv`
**Rows:** 17,680 (every HANA<=0 order line)
**VTEX scope:** Jul 7 CLT events only

### Column reference

| Column | Source | Description |
|--------|--------|-------------|
| `order_id` | Redshift | Cencommerce order ID |
| `id_tienda` | Redshift | Store code (e.g. `J989`) |
| `tienda` | Redshift | Store name |
| `item_id_for_hana` | Redshift/HANA | 18-digit zero-padded SAP item ID |
| `sku` | Redshift | VTEX SKU ID (numeric) |
| `name` | Redshift | Product name |
| `refid` | Redshift | Cencommerce reference ID |
| `status` | Redshift | Picker status code |
| `originalquantity` | Redshift | Units customer ordered |
| `pickingquantity` | Redshift | Units picker actually picked |
| `monto_clp` | Redshift | Line value in CLP |
| `fecha_creacion` | Redshift | Order creation time (**CLT**) |
| `inicio_picking` | Redshift | Picker start time (CLT) |
| `fin_picking` | Redshift | Picker end time (CLT) |
| `hana_last_batch_ts` | HANA | Timestamp of last HANA batch before order (CLT) |
| `hana_qty_at_last_batch` | HANA | `Ending_On_Hand_Qty` (always <= 0) |
| `gap_minutes` | Computed | Minutes between HANA batch and order creation |
| `vtex_sku` | Mapping | Mapped VTEX skuId |
| `vtex_store` | Mapping | Mapped VTEX storeId (e.g. `jumboclj989`) |
| `vtex_state_at_order` | VTEX | What VTEX was showing at order time: `IN_STOCK`, `OOS`, `NO_PRIOR_EVENT`, `NO_VTEX_EVENT`, `NO_MAPPING` |
| `vtex_last_event_utc` | VTEX | Timestamp of last Jul 7 VTEX event before order (**UTC**) |
| `vtex_last_event_chile` | VTEX | Same timestamp converted to **CLT** |
| `vtex_events_count_jul7` | VTEX | Total number of Jul 7 VTEX events for this (sku, store) |
| `vtex_all_events_jul7` | VTEX | Full Jul 7 timeline: `09:21:54CLT=IN \| 15:43:35CLT=OOS \| ...` |

---

## 5. Cross-Day Comparison

| VTEX state at order | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| **IN_STOCK** | 14,875 (77.3%) | **16,149 (88.7%)** | 13,823 (86.8%) | 12,586 (71.3%) |
| NO_PRIOR_EVENT | 4,025 (20.9%) | 1,563 (8.6%) | 1,662 (10.4%) | 4,098 (23.2%) |
| NO_VTEX_EVENT | 261 (1.4%) | 355 (1.9%) | 319 (2.0%) | 899 (5.1%) |
| OOS | 77 (0.4%) | 139 (0.8%) | 105 (0.7%) | 51 (0.3%) |
| **Total** | **19,241** | **17,680** | **15,551** | **17,261** |

| Pop A + VTEX IN_STOCK | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|---|---|---|---|---|
| Lines | 1,769 | **1,687** | 1,157 | 790 |
| CLP | ~9.4M | **9.2M** | 6.7M | 4.5M |

Jul 7 has the highest IN_STOCK rate (88.7%) because its high VTEX event volume (2.2M) ensures most items had a recent IN_STOCK event before the order. Jul 9 has the lowest (71.3%) because fewer events fired (no PUBLICADOR), pushing more lines into NO_PRIOR_EVENT.

---

## 6. The Three-Source Verdict

For each of the 17,680 order lines where HANA <= 0:

```
Three sources, one question: should this order have been allowed?

HANA (ERP):     "Stock is zero"       (knew before the order)
VTEX (website): "Item is available"   (told the customer to order — 88.7% of the time)
PICKER (human): "Item not found"      (confirmed on the shelf — for Pop A)
                "Item found"          (contradicted HANA — for false alarms)
```

### The failure chain (Pop A, 1,687 VTEX-confirmed preventable lines):

```
1. HANA batch arrives    -> qty = 0 for this item+store
2. [NOTHING HAPPENS]     -> no signal sent to VTEX
3. VTEX still shows      -> "in stock" on jumbo.cl
4. Customer sees         -> "in stock", places order
5. Order enters queue    -> picker assigned
6. Picker walks to shelf -> item not there
7. Picker marks          -> SIN STOCK (status 5)
8. Customer gets         -> "your item was not available" message
```

---

## 7. Limitations

| Limitation | Impact |
|---|---|
| **VTEX events are binary** | No actual quantity — only "in stock" or "out of stock". |
| **Events cut off at ~18:30 CLT** | SQS drain window limit. Orders after 18:30 may have stale VTEX state. |
| **NO_PRIOR_EVENT is ambiguous** | 1,563 lines had no Jul 7 VTEX event before order time. Almost certainly IN_STOCK (order was allowed), but not provable from Jul 7 data alone. |
| **PUBLICADOR contaminates** | Batch push spike inflates event volume and may not reflect real physical stock changes. |
| **Mapping imperfect** | 3 lines with NO_MAPPING — items not in `dim_surtidos`. |
