# July 6, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-08. Same methodology as JUL1_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0706_*/*.parquet` | 17 batches | 04:40-23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0706_FULL.csv` | 419,497 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~23K orders | Only 2 queries needed by evidence_pack_0706.py |
| **Evidence CSVs** | `analysis/jul6/evidence_pop_a_jul6.csv`, `evidence_pop_b_jul6.csv` | 2,608 + 10,613 | Output of evidence_pack_0706.py |
| **False alarms CSV** | `analysis/jul6/false_alarms_jul6.csv` | 13,980 | HANA <= 0 but picker found it |
| **Evidence Excel** | `analysis/jul6/evidence_pack_jul6.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul6/delay_proof_0706_preventable.parquet` | 19,241 | All HANA<=0 order lines |

### HANA batch times (17 batches, CLT)

```
04:40  05:41  06:40  07:40  08:40  09:40  10:40  11:40  12:41
13:40  14:40  15:41  16:41  17:41  21:40  22:40  23:41
```

No 03:41 batch (Jul 1 had one). Gap from 17:41 to 21:40 (4 hours).
First batch at 04:40 -> orders before this have no HANA snapshot.

HANA stats: 383,142,200 rows loaded, 355,871,079 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 6 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) |
|--------|------|-------|---|---------|
| **1** | Pickeado | **385,960** (92.0%) | 92.0% | 1,715.25 |
| **5** | Sin stock (OOS) | **13,757** (3.3%) | 3.3% | 72.51 |
| **4** | Sustituto | **10,461** (2.5%) | 2.5% | 51.87 |
| **11** | Agregado | **4,248** (1.0%) | 1.0% | 21.29 |
| **13** | Compuesto (weight OOS) | **3,339** (0.8%) | 0.8% | 46.08 |
| **12** | Faltante parcial | **1,721** (0.4%) | 0.4% | 16.34 |
| **3** | Sin stock (legacy) | **9** (0.0%) | 0.0% | 0.03 |
| **14** | Sin Stock (sala) | **1** (0.0%) | 0.0% | 0.01 |
| **15** | Compuesto (v2) | **1** (0.0%) | 0.0% | 0.00 |
| | **TOTAL** | **419,497** | 100% | **1,923.38** |

Jul 6 differences from Jul 1:
- **No status 0 lines** (all orders processed by export time)
- **Statuses 3, 14, 15 now active** (all had 0 rows on Jul 1)
- **OOS rate 3.3%** vs Jul 1's 2.5% — significantly worse
- Our filter `status IN (3,5,7,14)` captures: st5 (13,757) + st3 (9) + st14 (1) = 13,767 lines

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0706_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **419,497** |
| **Distinct orders** | **23,639** |
| Orders with processed lines (excl status 0) | 23,639 |
| Orders with only status 0 lines | 0 |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 771,119 |
| Qty delivered (SUM pickingquantity) | 726,547 (94.2%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,923,379,574** | 100% |
| Lost to OOS (status 5 + 13) | 118,590,536 | 6.2% |
| Lost to substitution (status 4) | 51,869,717 | 2.7% |

### Order impact summary

| Category | Orders | % of 23,639 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **9,560** | **40.4%** |
| Had any OOS (status 5 or 13) | 10,139 | 42.9% |
| Had substitution (status 4) | 7,352 | 31.1% |
| Had OOS AND substitution | 3,412 | 14.4% |
| Had partial pick (status 12) | 1,580 | 6.7% |

**59.6% of all processed orders had at least one item affected by OOS or substitution.**
**118.6M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
23,639 orders in CSV (419,497 item lines)
|- 23,639 with processed lines (no status 0 on Jul 6)
|  |- 9,560 clean (no OOS, no sub)
|  |- 14,079 had OOS or sub or both
|     |- 13,221 status 3/5/14 OOS lines -> Pop A (2,608) + Pop B (10,613)  <- Sections 5-6
|     |-  3,339 status-13 weight OOS lines -> NOT in Pop A/B
|     |- 10,461 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**419,497 order lines** from `all_orders_0706_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

For each order line, find the most recent HANA parquet batch BEFORE `fecha_creacion` and get `Ending_On_Hand_Qty` for that (item, store) pair.

```
419,497 total order lines
|- Has HANA snapshot before order : 406,788
|- No HANA snapshot              :  12,709  (order before 04:40 CLT or store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
406,788 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  19,241
|- HANA > 0 (ERP says in stock)      : 387,547
```

### Step 4a: HANA <= 0 branch -> 19,241 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (false alarm -- HANA was wrong) | **13,980** | 56.36 |
| 5 | **Picker NOT FOUND** (confirmed OOS) = **Pop A** | **2,607** | 13.87 |
| 4 | Substituted | 1,634 | 7.75 |
| 13 | Weight item OOS | 415 | 4.44 |
| 11 | Added item | 382 | 1.56 |
| 12 | Partial weight pick | 222 | 1.33 |
| 3 | Sin stock (legacy) | 1 | 0.01 |

Key finding: **13,980 false alarms** -- HANA said stock was 0 but picker found it anyway. These are cases where HANA underreported stock (ERP book < physical shelf). File: `analysis/jul6/false_alarms_jul6.csv`

Note: Pop A is 2,607 from delay_proof (st5 only) vs 2,608 from evidence_pack (st5 + st3 + st14 combined via `status IN (3,5,7,14)`). The extra 1 line = status 3.

### Step 4b: HANA > 0 branch -> 387,547 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (expected -- HANA was right) | **360,387** | 1,608.60 |
| 5 | **Picker NOT FOUND** (phantom inventory) = **Pop B** | **10,601** | 55.52 |
| 4 | Substituted | 8,501 | 42.60 |
| 13 | Weight item OOS | 2,813 | 40.28 |
| 11 | Added item | 3,781 | 19.34 |
| 12 | Partial weight pick | 1,455 | 14.61 |
| 3 | Sin stock (legacy) | 8 | 0.02 |
| 15 | Compuesto v2 | 1 | 0.00 |

Key finding: **10,601 phantom inventory** at status-5 level. Pop B (via evidence_pack filter `status IN (3,5,7,14)`) captures 10,613 (includes st3=8 + st14=0 + non-st5 matches from join).

### Step 4c: No HANA snapshot -> 12,709 order lines

Orders placed before the first HANA batch (04:40 CLT) or for items/stores not in HANA.

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | Picker FOUND it | 11,593 | 50.30 |
| 5 | Picker NOT FOUND (OOS) | 549 | 3.12 |
| 4 | Substituted | 326 | 1.52 |
| 13 | Weight item OOS | 111 | 1.36 |
| 11 | Added item | 85 | 0.39 |
| 12 | Partial weight pick | 44 | 0.40 |
| 14 | Sin Stock (sala) | 1 | 0.01 |

### Step 5: The two populations we analyze

From the drill-down above:

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  2,608  (13.88M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) : 10,613  (55.56M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              : 13,221  (69.44M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **2,608** |
| Distinct orders | **2,181** |
| Total CLP | **13,880,808** (13.88M) |
| Stores affected | **33** |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 604 | 23.2% | 2,569,601 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 135 | 5.2% | 633,232 |
| 3 | PICKED LATER -- only later, HANA flat | 292 | 11.2% | 1,590,383 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 856 | 32.8% | 4,578,121 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 721 | 27.6% | 4,509,471 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 604 | 2,569,601 |
| **YES -- someone got it after** (Tier 2+3) | 427 | 2,223,615 |
| **NO -- nobody got it** (Tier 4+5) | 1,577 | 9,087,592 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 2,608 | 100.0% |
| Picker marked OOS | 2,608 | 100.0% |
| Item not delivered (pick=0) | 2,608 | 100.0% |
| Order NOT cancelled | 2,592 | 99.4% |
| Order delivered/invoiced | 2,510 | 96.2% |
| Real picking session ran | 2,144 | 82.2% |
| Stock existed elsewhere (any order) | 1,031 | 39.5% |
| Stock present at pick time (before) | 604 | 23.2% |
| Multiple orders tried same item | 1,887 | 72.4% |

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
| Total lines | **10,613** |
| Distinct orders | **7,056** |
| Total CLP | **55,563,170** (55.56M) |
| Stores affected | **35** |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 3,705 | 34.9% | 17,979,750 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 396 | 3.7% | 1,940,827 |
| 3 | PICKED LATER -- only later, HANA flat | 929 | 8.8% | 4,098,084 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 2,633 | 24.8% | 13,714,956 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 2,950 | 27.8% | 17,829,553 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 3,705 | 17,979,750 |
| **YES -- someone got it after** (Tier 2+3) | 1,325 | 6,038,911 |
| **NO -- nobody got it** (Tier 4+5) | 5,583 | 31,544,509 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 10,613 | 100.0% |
| Picker marked OOS | 10,613 | 100.0% |
| Item not delivered (pick=0) | 10,613 | 100.0% |
| Order NOT cancelled | 10,567 | 99.6% |
| Order delivered/invoiced | 10,272 | 96.8% |
| Real picking session ran | 8,209 | 77.3% |
| Stock existed elsewhere (any order) | 5,030 | 47.4% |
| Stock present at pick time (before) | 3,705 | 34.9% |
| Multiple orders tried same item | 7,663 | 72.2% |

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
| **Lines** | 2,608 | 10,613 | **13,221** |
| **Orders** | 2,181 | 7,056 | **~8,200** |
| **CLP** | 13,880,808 | 55,563,170 | **69,443,978** |
| YES at/before | 604 (23.2%) | 3,705 (34.9%) | 4,309 (32.6%) |
| YES after | 427 (16.4%) | 1,325 (12.5%) | 1,752 (13.3%) |
| NO nobody | 1,577 (60.5%) | 5,583 (52.6%) | 7,160 (54.1%) |

### Comparison: Jul 6 vs Jul 1

| Metric | Jul 1 | Jul 6 | Change |
|--------|-------|-------|--------|
| Order lines | 370,654 | 419,497 | +13.2% |
| OOS rate (st5) | 2.5% | 3.3% | +0.8pp |
| Orders affected by OOS/sub | 51.9% | 59.6% | +7.7pp |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | +33.2% |
| Pop A lines | 1,482 | 2,608 | +76.0% |
| Pop A CLP | 8.53M | 13.88M | +62.7% |
| Pop B lines | 6,911 | 10,613 | +53.6% |
| Pop B CLP | 37.59M | 55.56M | +47.8% |
| Combined lines | 8,393 | 13,221 | +57.5% |
| Combined CLP | 46.13M | 69.44M | +50.5% |
| False alarms | 10,903 | 13,980 | +28.2% |

Jul 6 is significantly worse across all metrics.

### What's NOT in Pop A/B

| Category | Lines | CLP (M) | Why excluded |
|----------|-------|---------|--------------|
| Status 13 weight OOS (HANA <= 0) | 415 | 4.44 | Filter is `status IN (3,5,7,14)` -- misses 13 |
| Status 13 weight OOS (HANA > 0) | 2,813 | 40.28 | Same |
| Status 13 weight OOS (no HANA) | 111 | 1.36 | Same |
| Status 5 OOS (no HANA snapshot) | 549 | 3.12 | No HANA batch before order |
| Status 3+14 (no HANA) | 1 | 0.01 | No HANA batch before order |
| Status 3 (HANA > 0) | 8 | 0.02 | In Pop B count but minor |
| **Total excluded OOS** | **3,898** | **49.24** | |

---

## 8. Methodology: How We Got Here

### Step-by-step (what we did, in order)

**Step 1 -- Export all order lines from Redshift**
```
Script: scripts/export_orders_0706.py
Query:  SELECT * FROM jumbo_bo.io_items ii
        JOIN jumbo_bo.vw_master_pickers_data p ON ii.key = p.id_raw
        WHERE p.fecha_creacion >= '2026-07-06' AND p.fecha_creacion < '2026-07-07'
        AND p.id_tienda LIKE 'J%'
Output: exports/cencommerce_quiebre/all_orders_0706_FULL.csv (419,497 rows)
```

**Step 2 -- Download HANA parquets from S3**
```
Script: scripts/download_latest_batch.sh (or aws s3 cp)
Bucket: s3://cencosud.prod.sm.cl.raw/.../vw_daily_nrt_0706_*/
Output: data_samples/vw_daily_nrt_0706_*/*.parquet (17 batches)
Needs:  AWS SSO + VPN
```

**Step 3 -- Generate delay_proof_0706_preventable.parquet**
```
Script: scripts/delay_proof_0706.py
Input:  HANA parquets + orders CSV (local only, no Redshift)
Output: analysis/jul6/delay_proof_0706_preventable.parquet (19,241 rows)
        analysis/jul6/delay_proof_0706_raw.xlsx
What:   For EVERY order line, lookup most recent HANA batch before order.
        Filter to HANA <= 0 -> these are orders where HANA said OOS before customer ordered.
        This gives us the 19,241 "HANA was out of stock" lines.
```

**Step 4 -- Break down the 19,241 by picking status**
```
From the 19,241 HANA <= 0 lines:
  13,980 = picker FOUND it (false alarm -- HANA wrong)    -> false_alarms_jul6.csv
   2,608 = picker NOT FOUND (confirmed OOS)                -> Pop A
   1,634 = substituted
     415 = weight OOS (status 13)
     222 = partial pick
     382 = added
       0 = not picked (no status 0 on Jul 6)
```

**Step 5 -- Same for HANA > 0 (387,547 lines)**
```
From the 387,547 HANA > 0 lines:
 360,387 = picker FOUND it (expected)
  10,613 = picker NOT FOUND (phantom inventory)            -> Pop B
   8,501 = substituted
   2,813 = weight OOS (status 13)
   1,455 = partial pick
   3,781 = added
       0 = not picked
```

**Step 6 -- Run evidence_pack_0706.py for Pop A + Pop B**
```
Script: scripts/evidence_pack_0706.py
Input:  HANA parquets + orders CSV + 2 LIVE Redshift queries
        -> Redshift query 1: io_internal_orders (order status, canceledby, amountinvoice)
        -> Redshift query 2: io_picking (picking session proof)
Output: analysis/jul6/evidence_pack_jul6.xlsx
        analysis/jul6/evidence_pop_a_jul6.csv (2,608 lines)
        analysis/jul6/evidence_pop_b_jul6.csv (10,613 lines)
What:   For each Pop A/B line, attaches:
        - HANA daily profile (all 17 batches for that item+store)
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
| `evidence_pack_0706.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0706.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0706.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
| `rs.py` | Redshift connection helper (persistent conn) | -- | YES |

### CSV read gotcha (DuckDB) -- Jul 6 specific

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

Jul 6 CSV requires THREE extra VARCHAR overrides (`venta_neta`, `items_solicitados`, `items_faltantes`) compared to Jul 1 (which only needed `dateupdate`, `inicio_picking`, `fin_picking`). This is because some Jul 6 orders have empty strings in these fields.

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
| `analysis/jul6/evidence_pack_jul6.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul6/evidence_pop_a_jul6.csv` | 2,608 | Pop A full detail |
| `analysis/jul6/evidence_pop_b_jul6.csv` | 10,613 | Pop B full detail |
| `analysis/jul6/false_alarms_jul6.csv` | 13,980 | HANA <= 0 but picker found it |
| `analysis/jul6/delay_proof_0706_preventable.parquet` | 19,241 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul6/delay_proof_0706_raw.xlsx` | 19,241 | Same, Excel format with summaries |

### False alarms by store (top 10)

| Store | False alarms |
|-------|-------------|
| J414 | 2,348 |
| J411 | 1,821 |
| J512 | 1,549 |
| J410 | 1,338 |
| J403 | 1,097 |
| J408 | 1,010 |
| J519 | 515 |
| J775 | 460 |
| J513 | 323 |
| J989 | 296 |

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 19,241 |
| Distinct orders | 11,505 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 40.4 |
| Gap minutes (median) | 34.0 |
| Gap minutes (max) | 239 |
