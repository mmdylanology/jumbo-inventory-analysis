# July 7, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-11. Same methodology as JUL1_COMPLETE_REFERENCE.md and JUL6_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0707_*/*.parquet` | 24 batches | 00:41–23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0707_FULL.csv` | 371,553 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~22K orders | Only 2 queries needed by evidence_pack_0707.py |
| **Evidence CSVs** | `analysis/jul7/evidence_pop_a_jul7.csv`, `evidence_pop_b_jul7.csv` | 1,978 + 8,510 | Output of evidence_pack_0707.py |
| **Evidence Excel** | `analysis/jul7/evidence_pack_jul7.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul7/delay_proof_0707_preventable.parquet` | 17,680 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul7/delay_proof_0707_raw.xlsx` | 17,680 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:41  01:41  02:40  03:41  04:40  05:41  06:40  07:40  08:40  09:41  10:41  11:40
12:40  13:40  14:40  15:40  16:41  17:41  18:41  19:40  20:41  21:40  22:41  23:41
```

Full 24/24 coverage — no gaps. This is the most complete day in the dataset (Jul 6 had a 4-hour gap from 17:41 to 21:40; Jul 1 had an overnight gap from 19:40 to 23:41). First batch at 00:41 means orders placed before 00:41 CLT have no HANA snapshot, but this window is effectively zero for ecommerce traffic.

HANA stats: 540,957,044 rows loaded, 502,427,779 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 7 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) |
|--------|------|-------|---|---------|
| **1** | Pickeado | **344,261** | 92.7% | 1,552.48 |
| **5** | Sin stock (OOS) | **10,646** | 2.9% | 59.15 |
| **4** | Sustituto | **8,418** | 2.3% | 42.62 |
| **11** | Agregado | **4,043** | 1.1% | 20.31 |
| **13** | Compuesto (weight OOS) | **2,722** | 0.7% | 41.07 |
| **12** | Faltante parcial | **1,462** | 0.4% | 13.79 |
| **3** | Sin stock (legacy) | **1** | 0.0% | 0.02 |
| | **TOTAL** | **371,553** | 100% | **1,729.45** |

Jul 7 differences from prior days:
- **OOS rate 3.6%** — better than Jul 6 (3.3%) but worse than Jul 1 (2.5%)
- **Status 14 and 15 absent** (both appeared on Jul 6 with 1 row each)
- **Status 3 present** with 1 row (as on Jul 6)


---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0707_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **371,553** |
| **Distinct orders** | **22,349** |
| Orders with processed lines | 22,349 |
| Orders with only status 0 lines | 0 |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 674,173 |
| Qty delivered (SUM pickingquantity) | 640,212 (95.0%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,729,454,718** | 100% |
| Lost to OOS (status 5 + 13) | 100,225,048 | 5.8% |
| Lost to substitution (status 4) | 42,622,053 | 2.5% |

### Order impact summary

| Category | Orders | % of 22,349 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **10,307** | **46.1%** |
| Had any OOS (status 5 or 13) | 8,447 | 37.8% |
| Had substitution (status 4) | 6,151 | 27.5% |
| Had OOS AND substitution | 2,556 | 11.4% |
| Had partial pick (status 12) | 1,327 | 5.9% |

**53.9% of all processed orders had at least one item affected by OOS or substitution.**
**100.2M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
22,349 orders in CSV (371,553 item lines)
|- 22,349 with processed lines (no status 0 on Jul 7)
|  |- 10,307 clean (no OOS, no sub)
|  |- 12,042 had OOS or sub or both
|     |- 10,488 status 3/5 OOS lines -> Pop A (1,978) + Pop B (8,510)  <- Sections 5-6
|     |-  2,722 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  8,418 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**371,553 order lines** from `all_orders_0707_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

For each order line, find the most recent HANA parquet batch BEFORE `fecha_creacion` and get `Ending_On_Hand_Qty` for that (item, store) pair.

```
371,553 total order lines
|- Has HANA snapshot before order : 366,613
|- No HANA snapshot              :   4,940  (order before 00:41 CLT or store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
366,613 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  17,680
|- HANA > 0 (ERP says in stock)      : 348,933
```

### Step 4a: HANA <= 0 branch -> 17,680 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (false alarm -- HANA was wrong) | **13,408** | 56.14 |
| 5 | **Picker NOT FOUND** (confirmed OOS) = **Pop A** | **1,978** | 11.57 |
| 4 | Substituted | 1,237 | 6.21 |
| 13 | Weight item OOS | 439 | 6.07 |
| 11 | Added item | 397 | 1.62 |
| 12 | Partial weight pick | 221 | 1.29 |

Key finding: **13,408 false alarms** -- HANA said stock was 0 but picker found it anyway. These are cases where HANA underreported stock (ERP book < physical shelf). Fewer false alarms than Jul 6 (13,980), consistent with the smaller HANA<=0 pool (17,680 vs 19,241).

### Step 4b: HANA > 0 branch -> 348,933 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (expected -- HANA was right) | **326,268** | 1,476.45 |
| 5 | **Picker NOT FOUND** (phantom inventory) = **Pop B** | **8,509** | 46.81 |
| 4 | Substituted | 7,083 | 35.98 |
| 13 | Weight item OOS | 2,251 | 34.52 |
| 11 | Added item | 3,598 | 18.49 |
| 12 | Partial weight pick | 1,223 | 12.40 |
| 3 | Sin stock (legacy) | 1 | 0.02 |

Key finding: **8,509 phantom inventory** at status-5 level. Pop B (via evidence_pack filter `status IN (3,5,7,14)`) captures 8,510 (includes st3=1 from the HANA>0 branch).

### Step 4c: No HANA snapshot -> 4,940 order lines

Orders placed before the first HANA batch (00:41 CLT) or for items/stores not in HANA. This window is much smaller than Jul 6 (12,709) and Jul 1 (12,936) due to full 24-hour HANA coverage.

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | Picker FOUND it | 4,585 | 19.89 |
| 5 | Picker NOT FOUND (OOS) | 159 | 0.78 |
| 4 | Substituted | 98 | 0.43 |
| 11 | Added item | 48 | 0.21 |
| 13 | Weight item OOS | 32 | 0.49 |
| 12 | Partial weight pick | 18 | 0.10 |

### Step 5: The two populations we analyze

From the drill-down above:

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  1,978  (11.57M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) :  8,510  (46.83M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              : 10,488  (58.40M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **1,978** |
| Total CLP | **11,570,805** (11.57M) |
| Stores affected | **~33** |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 391 | 19.8% | 1,705,622 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 145 | 7.3% | 783,497 |
| 3 | PICKED LATER -- only later, HANA flat | 139 | 7.0% | 817,702 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 642 | 32.5% | 3,834,190 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 661 | 33.4% | 4,429,794 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 391 | 1,705,622 |
| **YES -- someone got it after** (Tier 2+3) | 284 | 1,601,199 |
| **NO -- nobody got it** (Tier 4+5) | 1,303 | 8,263,984 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 1,978 | 100.0% |
| Picker marked OOS | 1,978 | 100.0% |
| Item not delivered (pick=0) | 1,978 | 100.0% |
| Order NOT cancelled | 1,970 | 99.6% |
| Order delivered/invoiced | 1,923 | 97.2% |
| Real picking session ran | 1,646 | 83.2% |

### Sample rows -- Pop A

**Tier 1 (PICKER MISS -- strongest evidence):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| v232807071jmch-01 | J414 | 780636-KG | Trucha Filete Fresca Granel | 41,980 | -7.8 | 1 |
| v232815154jmch-01 | J408 | 1462094-KG | Filete Al Vacio kg | 39,879 | -14.3 | 5 |
| v232808698jmch-01 | J775 | 1462094-KG | Filete Al Vacio kg | 39,879 | 0.0 | 1 |

> Interpretation: HANA said 0 (or negative), picker marked OOS, but 1-5 OTHER orders picked the same item at/before this order. The stock was there -- picker missed it.

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | name | CLP | HANA qty | orders_tried |
|----------|-------|-------|------|-----|----------|-------------|
| v232797681jmch-01 | J410 | 2002939 | Alimento Perro Amity Premium Cerdo Iberico y Arroz | 39,941 | 0 | 2 |
| v232774400jmch-01 | J403 | 2003780 | Estufa Simil Fuego Nex SF1608 Digital | 38,994 | 0 | 3 |
| v232785316jmch-01 | J403 | 2003780 | Estufa Simil Fuego Nex SF1608 Digital | 38,994 | 0 | 3 |

> Interpretation: HANA said 0, multiple orders tried this item all day, ALL failed. Real stockout confirmed by both HANA and picker, across multiple independent attempts.

---

## 6. Population B -- Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock != physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **8,510** |
| Total CLP | **46,827,311** (46.83M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 2,602 | 30.6% | 13,539,203 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 419 | 4.9% | 1,946,101 |
| 3 | PICKED LATER -- only later, HANA flat | 612 | 7.2% | 2,797,282 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 2,170 | 25.5% | 11,696,955 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 2,707 | 31.8% | 16,847,770 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 2,602 | 13,539,203 |
| **YES -- someone got it after** (Tier 2+3) | 1,031 | 4,743,383 |
| **NO -- nobody got it** (Tier 4+5) | 4,877 | 28,544,725 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 8,510 | 100.0% |
| Picker marked OOS | 8,510 | 100.0% |
| Item not delivered (pick=0) | 8,510 | 100.0% |
| Order NOT cancelled | 8,484 | 99.7% |
| Order delivered/invoiced | 8,255 | 97.0% |
| Real picking session ran | 6,706 | 78.8% |

### Sample rows -- Pop B

**Tier 1 (PICKER MISS -- HANA showed stock, picker missed it):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| v232763018jmch-01 | J403 | 2029237 | Alimento Perro Doko Adulto All Breed Carne de Pollo | 179,880 | 21 | 1 |
| v232781057jmch-01 | J408 | 2006303 | Horno Electrico Ursus Trotter UT-Backofen38B | 84,990 | 1 | 1 |
| v232808149jmch-01 | J411 | 1996518-KG | Lomo Liso Desgrasado Bandeja 900 g | 70,713 | 6.5 | 8 |

> Interpretation: HANA said stock > 0 (even 21 units in one case), picker marked OOS, but other orders successfully picked the same item. Classic phantom inventory -- the stock exists but the picker couldn't find it or chose not to.

---

## 7. Combined Summary

| | Pop A (HANA <= 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 1,978 | 8,510 | **10,488** |
| **CLP** | 11,570,805 | 46,827,311 | **58,398,116** |
| YES at/before | 391 (19.8%) | 2,602 (30.6%) | 2,993 (28.5%) |
| YES after | 284 (14.4%) | 1,031 (12.1%) | 1,315 (12.5%) |
| NO nobody | 1,303 (65.9%) | 4,877 (57.3%) | 6,180 (58.9%) |

### Comparison: Jul 1, Jul 6, Jul 7

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul7 vs Jul6 |
|--------|-------|-------|-------|--------------|
| Order lines | 370,654 | 419,497 | 371,553 | -11.4% |
| Distinct orders | 21,635 | 23,639 | 22,349 | -5.5% |
| HANA batches | 18 | 17 | 24 | +41.2% |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | -0.4pp |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | -5.7pp |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | 100.2M | -15.5% |
| HANA<=0 order lines | 14,523 | 19,241 | 17,680 | -8.1% |
| False alarms | 10,903 | 13,980 | 13,408 | -4.1% |
| Pop A lines | 1,482 | 2,608 | 1,978 | -24.2% |
| Pop A CLP | 8.53M | 13.88M | 11.57M | -16.6% |
| Pop B lines | 6,911 | 10,613 | 8,510 | -19.8% |
| Pop B CLP | 37.59M | 55.56M | 46.83M | -15.7% |
| Combined lines | 8,393 | 13,221 | 10,488 | -20.7% |
| Combined CLP | 46.13M | 69.44M | 58.40M | -15.9% |
| No HANA snapshot lines | 12,936 | 12,709 | 4,940 | -61.1% |

Jul 7 represents a meaningful improvement over Jul 6 across all metrics, but remains worse than Jul 1. The 24-batch HANA coverage on Jul 7 drastically reduces the "no HANA snapshot" pool (4,940 vs 12,709 on Jul 6), giving a more complete picture of intraday stock state.

### What's NOT in Pop A/B

| Category | Lines | CLP (M) | Why excluded |
|----------|-------|---------|--------------|
| Status 13 weight OOS (HANA <= 0) | 439 | 6.07 | Filter is `status IN (3,5,7,14)` -- misses 13 |
| Status 13 weight OOS (HANA > 0) | 2,251 | 34.52 | Same |
| Status 13 weight OOS (no HANA) | 32 | 0.49 | Same |
| Status 5 OOS (no HANA snapshot) | 159 | 0.78 | No HANA batch before order |
| **Total excluded OOS** | **2,881** | **41.86** | |

---

## 8. Methodology: How We Got Here

### Step-by-step (what we did, in order)

**Step 1 -- Export all order lines from Redshift**
```
Script: scripts/export_orders_0707.py
Query:  SELECT * FROM jumbo_bo.io_items ii
        JOIN jumbo_bo.vw_master_pickers_data p ON ii.key = p.id_raw
        WHERE p.fecha_creacion >= '2026-07-07' AND p.fecha_creacion < '2026-07-08'
        AND p.id_tienda LIKE 'J%'
Output: exports/cencommerce_quiebre/all_orders_0707_FULL.csv (371,553 rows)
```

**Step 2 -- Download HANA parquets from S3**
```
Script: scripts/download_latest_batch.sh (or aws s3 cp)
Bucket: s3://cencosud.prod.sm.cl.raw/.../vw_daily_nrt_0707_*/
Output: data_samples/vw_daily_nrt_0707_*/*.parquet (24 batches)
Needs:  AWS SSO + VPN
```

**Step 3 -- Generate delay_proof_0707_preventable.parquet**
```
Script: scripts/delay_proof_0707.py
Input:  HANA parquets + orders CSV (local only, no Redshift)
Output: analysis/jul7/delay_proof_0707_preventable.parquet (17,680 rows)
        analysis/jul7/delay_proof_0707_raw.xlsx
What:   For EVERY order line, lookup most recent HANA batch before order.
        Filter to HANA <= 0 -> these are orders where HANA said OOS before customer ordered.
        This gives us the 17,680 "HANA was out of stock" lines.
```

**Step 4 -- Break down the 17,680 by picking status**
```
From the 17,680 HANA <= 0 lines:
  13,408 = picker FOUND it (false alarm -- HANA wrong)
   1,978 = picker NOT FOUND (confirmed OOS)                 -> Pop A
   1,237 = substituted
     439 = weight OOS (status 13)
     397 = added
     221 = partial pick
```

**Step 5 -- Same for HANA > 0 (348,933 lines)**
```
From the 348,933 HANA > 0 lines:
 326,268 = picker FOUND it (expected)
   8,509 = picker NOT FOUND (phantom inventory)             -> Pop B
   7,083 = substituted
   2,251 = weight OOS (status 13)
   3,598 = added
   1,223 = partial pick
       1 = sin stock legacy (status 3)
```

**Step 6 -- Run evidence_pack_0707.py for Pop A + Pop B**
```
Script: scripts/evidence_pack_0707.py
Input:  HANA parquets + orders CSV + 2 LIVE Redshift queries
        -> Redshift query 1: io_internal_orders (order status, canceledby, amountinvoice)
        -> Redshift query 2: io_picking (picking session proof)
Output: analysis/jul7/evidence_pack_jul7.xlsx
        analysis/jul7/evidence_pop_a_jul7.csv (1,978 lines)
        analysis/jul7/evidence_pop_b_jul7.csv (8,510 lines)
What:   For each Pop A/B line, attaches:
        - HANA daily profile (all 24 batches for that item+store)
        - Cross-order corroboration (did another order pick the same item?)
        - Timing corroboration (before or after this order?)
        - Restock signal (did HANA qty rise after?)
        - Redshift order status (delivered? invoiced? cancelled?)
        - Redshift picking session (real picker or system?)
        -> Assigns confirmation tier (1-5) based on evidence
```

---

## 9. Scripts Reference

| Script | What it does | Data sources | Redshift needed? |
|--------|-------------|--------------|------------------|
| `evidence_pack_0707.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0707.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0707.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
| `rs.py` | Redshift connection helper (persistent conn) | -- | YES |

### CSV read gotcha (DuckDB) -- Jul 7 specific

```python
read_csv_auto('path.csv',
    nullstr='None',
    types={'dateupdate': 'VARCHAR',
           'inicio_picking': 'VARCHAR',
           'fin_picking': 'VARCHAR',
           'venta_neta': 'VARCHAR',
           'items_solicitados': 'VARCHAR',
           'items_faltantes': 'VARCHAR'})
```

Same 6-column VARCHAR override required as Jul 6. The three extra fields (`venta_neta`, `items_solicitados`, `items_faltantes`) contain empty strings in some rows that DuckDB auto-detects incorrectly. Without these overrides, DuckDB will either raise a type error or silently drop rows.

---

## 10. Confirmation Tier Logic

```sql
CASE
  WHEN got_before > 0
     THEN '1 PICKER MISS (same item picked at/before this order - restock impossible)'
  WHEN got_after > 0 AND hana_rose_after
     THEN '2 RESTOCK LIKELY (same item picked only later + HANA rose after)'
  WHEN got_after > 0
     THEN '3 PICKED LATER (only later, HANA flat - probable picker miss)'
  WHEN n_orders_item > 1
     THEN '4 CONFIRMED STOCKOUT (multiple orders, all failed)'
  ELSE '5 SINGLE ATTEMPT (not delivered; cause unconfirmed)'
END
```

---

## 11. Key Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul7/evidence_pack_jul7.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul7/evidence_pop_a_jul7.csv` | 1,978 | Pop A full detail |
| `analysis/jul7/evidence_pop_b_jul7.csv` | 8,510 | Pop B full detail |
| `analysis/jul7/delay_proof_0707_preventable.parquet` | 17,680 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul7/delay_proof_0707_raw.xlsx` | 17,680 | Same, Excel format with summaries |

Note: No separate false_alarms CSV was produced for Jul 7 (unlike Jul 6 which had `false_alarms_jul6.csv`). The 13,408 false alarm lines are derivable from `delay_proof_0707_preventable.parquet` filtered to `status = 1`.

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 17,680 |
| Distinct orders | 10,662 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 61 |

The gap distribution is notably tighter on Jul 7 (mean 30 min, max 61 min) compared to Jul 6 (mean 40.4 min, max 239 min). The Jul 6 outlier at 239 min was caused by the 4-hour HANA coverage gap (17:41–21:40); with full 24-batch coverage on Jul 7, the maximum gap is bounded by the ~60-minute HANA polling interval.

### Three-day delay proof comparison

| Metric | Jul 1 | Jul 6 | Jul 7 |
|--------|-------|-------|-------|
| HANA<=0 events | 14,523 | 19,241 | 17,680 |
| Distinct orders | -- | 11,505 | 10,662 |
| Gap mean (min) | -- | 40.4 | 30 |
| Gap max (min) | -- | 239 | 61 |
| HANA batches | 18 | 17 | 24 |
