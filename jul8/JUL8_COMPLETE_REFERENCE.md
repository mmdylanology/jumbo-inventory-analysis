# July 8, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-11. Same methodology as JUL1_COMPLETE_REFERENCE.md and JUL6_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0708_*/*.parquet` | 24 batches | 00:40-23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0708_FULL.csv` | 355,262 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~22K orders | Only 2 queries needed by evidence_pack_0708.py |
| **Evidence CSVs** | `analysis/jul8/evidence_pop_a_jul8.csv`, `evidence_pop_b_jul8.csv` | 1,402 + 7,235 | Output of evidence_pack_0708.py |
| **Evidence Excel** | `analysis/jul8/evidence_pack_jul8.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul8/delay_proof_0708_preventable.parquet` | 15,551 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul8/delay_proof_0708_raw.xlsx` | 15,551 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:40  01:41  02:40  03:41  04:41  05:40  06:40  07:40  08:41  09:40  10:40  11:41
12:41  13:40  14:40  15:41  16:41  17:40  18:41  19:40  20:41  21:40  22:40  23:41
```

Full 24/24 coverage — best coverage of any day analysed so far. First batch at 00:40 means all orders have a HANA snapshot.

HANA stats: 541,089,638 rows loaded, 502,519,697 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 8 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) |
|--------|------|-------|---|---------|
| **1** | Pickeado | **331,465** | 93.3% | 1,505.92 |
| **5** | Sin stock (OOS) | **8,688** | 2.4% | 48.29 |
| **4** | Sustituto | **7,293** | 2.1% | 39.78 |
| **11** | Agregado | **3,829** | 1.1% | 18.13 |
| **13** | Compuesto (weight OOS) | **2,608** | 0.7% | 42.04 |
| **12** | Faltante parcial | **1,329** | 0.4% | 13.25 |
| **3** | Sin stock (legacy) | **46** | 0.0% | 0.15 |
| **14** | Sin Stock (sala) | **4** | 0.0% | 0.02 |
| | **TOTAL** | **355,262** | 100% | **1,667.57** |

Jul 8 differences from Jul 6:
- **Lower volume** — 355,262 lines vs Jul 6's 419,497 (Tuesday vs Saturday effect)
- **OOS rate 2.4%** vs Jul 6's 3.3% — meaningfully better, consistent with Jul 1 (2.5%)
- **Affected orders 49.1%** vs Jul 6's 59.6% — back toward normal operating range
- Our filter `status IN (3,5,7,14)` captures: st5 (8,688) + st3 (46) + st14 (4) = 8,738 lines

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0708_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **355,262** |
| **Distinct orders** | **22,106** |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 641,659 |
| Qty delivered (SUM pickingquantity) | 613,024 (95.5%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,667,570,673** | 100% |
| Lost to OOS (status 5 + 13) | 90,325,196 | 5.4% |
| Lost to substitution (status 4) | 39,775,379 | 2.4% |

### Order impact summary

| Category | Orders | % of 22,106 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **11,255** | **50.9%** |
| Had any OOS (status 5 or 13) | 7,488 | 33.9% |
| Had substitution (status 4) | 5,505 | 24.9% |
| Had OOS AND substitution | 2,142 | 9.7% |
| Had partial pick (status 12) | 1,232 | 5.6% |

**49.1% of all processed orders had at least one item affected by OOS or substitution.**
**90.3M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
22,106 orders in CSV (355,262 item lines)
|- 22,106 with processed lines
|  |- 11,255 clean (no OOS, no sub)
|  |- 10,851 had OOS or sub or both
|     |-  8,637 status 3/5/14 OOS lines -> Pop A (1,402) + Pop B (7,235)  <- Sections 5-6
|     |-  2,608 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  7,293 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**355,262 order lines** from `all_orders_0708_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

For each order line, find the most recent HANA parquet batch BEFORE `fecha_creacion` and get `Ending_On_Hand_Qty` for that (item, store) pair.

```
355,262 total order lines
|- Has HANA snapshot before order : 351,430
|- No HANA snapshot               :   3,832  (store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
351,430 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  15,551
|- HANA > 0 (ERP says in stock)      : 335,879
```

### Step 4a: HANA <= 0 branch -> 15,551 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (false alarm -- HANA was wrong) | **12,246** | 52.39 |
| 5 | **Picker NOT FOUND** (confirmed OOS) = **Pop A** | **1,400** | 8.28 |
| 4 | Substituted | 961 | 5.87 |
| 11 | Added item | 403 | 1.75 |
| 13 | Weight item OOS | 350 | 5.77 |
| 12 | Partial weight pick | 189 | 1.11 |
| 3 | Sin stock (legacy) | 1 | 0.00 |
| 14 | Sin Stock (sala) | 1 | 0.00 |

Key finding: **12,246 false alarms** -- HANA said stock was 0 but picker found it anyway. These are cases where HANA underreported stock (ERP book < physical shelf).

Note: Pop A is 1,400 from delay_proof (st5 only) vs 1,402 from evidence_pack (st5 + st3 + st14 combined via `status IN (3,5,7,14)`). The extra 2 lines = status 3 (1) + status 14 (1).

### Step 4b: HANA > 0 branch -> 335,879 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (expected -- HANA was right) | **315,650** | 1,436.97 |
| 5 | **Picker NOT FOUND** (phantom inventory) = **Pop B** | **7,187** | 39.46 |
| 4 | Substituted | 6,245 | 33.48 |
| 11 | Added item | 3,388 | 16.19 |
| 13 | Weight item OOS | 2,236 | 35.93 |
| 12 | Partial weight pick | 1,125 | 12.04 |
| 3 | Sin stock (legacy) | 45 | 0.15 |
| 14 | Sin Stock (sala) | 3 | 0.02 |

Key finding: **7,187 phantom inventory** at status-5 level. Pop B (via evidence_pack filter `status IN (3,5,7,14)`) captures 7,235 (includes st3=45 + st14=3 + non-st5 matches from join).

### Step 4c: No HANA snapshot -> 3,832 order lines

Orders for items/stores not present in any HANA batch on Jul 8.

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | Picker FOUND it | 3,569 | 16.56 |
| 5 | Picker NOT FOUND (OOS) | 101 | 0.55 |
| 4 | Substituted | 87 | 0.43 |
| 11 | Added item | 38 | 0.19 |
| 13 | Weight item OOS | 22 | 0.34 |
| 12 | Partial weight pick | 15 | 0.11 |

### Step 5: The two populations we analyze

From the drill-down above:

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  1,402  (8.28M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) :  7,235  (39.63M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              :  8,637  (47.91M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **1,402** |
| Total CLP | **8,281,472** (8.28M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 253 | 18.0% | 1,135,443 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 92 | 6.6% | 412,376 |
| 3 | PICKED LATER -- only later, HANA flat | 80 | 5.7% | 476,334 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 431 | 30.7% | 2,598,425 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 546 | 38.9% | 3,658,894 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 253 | 1,135,443 |
| **YES -- someone got it after** (Tier 2+3) | 172 | 888,710 |
| **NO -- nobody got it** (Tier 4+5) | 977 | 6,257,319 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 1,402 | 100.0% |
| Picker marked OOS | 1,402 | 100.0% |
| Item not delivered (pick=0) | 1,402 | 100.0% |
| Order NOT cancelled | 1,392 | 99.3% |
| Order delivered/invoiced | 1,347 | 96.1% |
| Real picking session ran | 1,178 | 84.0% |

### Sample rows -- Pop A

**Tier 1 (PICKER MISS -- strongest evidence):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| Example A1 | J414 | 780636-KG | Trucha Filete Fresca Granel | 41,980 | -5.2 | 1 |
| Example A2 | J408 | 1462094-KG | Filete Al Vacio kg | 39,879 | -12.1 | 3 |
| Example A3 | J411 | 1996518-KG | Lomo Liso Desgrasado Bandeja 900 g | 35,990 | 0.0 | 2 |

> Interpretation: HANA said 0 (or negative), picker marked OOS, but 1+ OTHER orders picked the same item at/before this order. The stock was there -- picker missed it.

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | name | CLP | HANA qty | orders_tried |
|----------|-------|-------|------|-----|----------|-------------|
| Example A4 | J410 | 2002939 | Alimento Perro Amity Premium | 39,941 | 0 | 2 |
| Example A5 | J403 | 2003780 | Estufa Simil Fuego Nex | 38,994 | 0 | 3 |
| Example A6 | J403 | 1995001 | Televisor LED 50" | 299,990 | 0 | 2 |

> Interpretation: HANA said 0, multiple orders tried this item all day, ALL failed. Real stockout confirmed by both HANA and picker, across multiple independent attempts.

---

## 6. Population B -- Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock != physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **7,235** |
| Total CLP | **39,626,116** (39.63M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 2,052 | 28.4% | 10,048,194 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 302 | 4.2% | 1,534,994 |
| 3 | PICKED LATER -- only later, HANA flat | 570 | 7.9% | 3,734,392 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 1,774 | 24.5% | 9,063,185 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 2,537 | 35.1% | 15,245,351 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 2,052 | 10,048,194 |
| **YES -- someone got it after** (Tier 2+3) | 872 | 5,269,386 |
| **NO -- nobody got it** (Tier 4+5) | 4,311 | 24,308,536 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 7,235 | 100.0% |
| Picker marked OOS | 7,235 | 100.0% |
| Item not delivered (pick=0) | 7,235 | 100.0% |
| Order NOT cancelled | 7,228 | 99.9% |
| Order delivered/invoiced | 6,952 | 96.1% |
| Real picking session ran | 5,768 | 79.7% |

### Sample rows -- Pop B

**Tier 1 (PICKER MISS -- HANA showed stock, picker missed it):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| Example B1 | J403 | 2029237 | Alimento Perro Doko Adulto | 89,990 | 14 | 2 |
| Example B2 | J408 | 2006303 | Horno Electrico Ursus Trotter | 84,990 | 3 | 1 |
| Example B3 | J411 | 1996518-KG | Lomo Liso Desgrasado Bandeja 900 g | 70,713 | 6.5 | 5 |

> Interpretation: HANA said stock > 0, picker marked OOS, but other orders successfully picked the same item. Classic phantom inventory -- the stock exists but the picker couldn't locate it or chose a substitution.

---

## 7. Combined Summary

| | Pop A (HANA <= 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 1,402 | 7,235 | **8,637** |
| **CLP** | 8,281,472 | 39,626,116 | **47,907,588** |
| YES at/before | 253 (18.0%) | 2,052 (28.4%) | 2,305 (26.7%) |
| YES after | 172 (12.3%) | 872 (12.1%) | 1,044 (12.1%) |
| NO nobody | 977 (69.7%) | 4,311 (59.6%) | 5,288 (61.2%) |

### Multi-day comparison: Jul 1 / Jul 6 / Jul 7 / Jul 8

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul 8 |
|--------|-------|-------|-------|-------|
| Order lines | 370,654 | 419,497 | — | 355,262 |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | 2.4% |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | 49.1% |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | — | 90.3M |
| Pop A lines | 1,482 | 2,608 | 1,978 | 1,402 |
| Pop A CLP | 8.53M | 13.88M | 11.57M | 8.28M |
| Pop B lines | 6,911 | 10,613 | 8,510 | 7,235 |
| Pop B CLP | 37.59M | 55.56M | 46.83M | 39.63M |
| Combined lines | 8,393 | 13,221 | 10,488 | 8,637 |
| Combined CLP | 46.13M | 69.44M | 58.40M | 47.91M |
| False alarms | 10,903 | 13,980 | — | 12,246 |
| HANA batches | 24 | 17 | — | 24 |

Jul 8 is the best weekday in the series (tied near Jul 1). Jul 6 Saturday remains the worst day by all metrics. The Jul 6 spike (+50% vs Jul 1) appears to be a weekend/Saturday effect. Jul 8 confirms the weekday baseline has not deteriorated from Jul 1 levels.

### What's NOT in Pop A/B

| Category | Lines | CLP (M) | Why excluded |
|----------|-------|---------|--------------|
| Status 13 weight OOS (HANA <= 0) | 350 | 5.77 | Filter is `status IN (3,5,7,14)` -- misses 13 |
| Status 13 weight OOS (HANA > 0) | 2,236 | 35.93 | Same |
| Status 13 weight OOS (no HANA) | 22 | 0.34 | Same |
| Status 5 OOS (no HANA snapshot) | 101 | 0.55 | No HANA batch before order |
| Status 3+14 (no HANA) | 0 | 0.00 | No HANA batch before order |
| Status 3 (HANA > 0) | 45 | 0.15 | In Pop B count but minor |
| **Total excluded OOS** | **2,754** | **42.74** | |

---

## 8. Methodology: How We Got Here

### Step-by-step (what we did, in order)

**Step 1 -- Export all order lines from Redshift**
```
Script: scripts/export_orders_0708.py
Query:  SELECT * FROM jumbo_bo.io_items ii
        JOIN jumbo_bo.vw_master_pickers_data p ON ii.key = p.id_raw
        WHERE p.fecha_creacion >= '2026-07-08' AND p.fecha_creacion < '2026-07-09'
        AND p.id_tienda LIKE 'J%'
Output: exports/cencommerce_quiebre/all_orders_0708_FULL.csv (355,262 rows)
```

**Step 2 -- Download HANA parquets from S3**
```
Script: scripts/download_latest_batch.sh (or overnight_downloader.sh)
Bucket: s3://cencosud.prod.sm.cl.raw/.../vw_daily_nrt_0708_*/
Output: data_samples/vw_daily_nrt_0708_*/*.parquet (24 batches)
Needs:  AWS SSO + VPN
```

**Step 3 -- Generate delay_proof_0708_preventable.parquet**
```
Script: scripts/delay_proof_0708.py
Input:  HANA parquets + orders CSV (local only, no Redshift)
Output: analysis/jul8/delay_proof_0708_preventable.parquet (15,551 rows)
        analysis/jul8/delay_proof_0708_raw.xlsx
What:   For EVERY order line, lookup most recent HANA batch before order.
        Filter to HANA <= 0 -> these are orders where HANA said OOS before customer ordered.
        This gives us the 15,551 "HANA was out of stock" lines.
```

**Step 4 -- Break down the 15,551 by picking status**
```
From the 15,551 HANA <= 0 lines:
  12,246 = picker FOUND it (false alarm -- HANA wrong)
   1,400 = picker NOT FOUND st5 (confirmed OOS)              -> Pop A core
       1 = picker NOT FOUND st3 (legacy OOS)                 -> Pop A (included in filter)
       1 = picker NOT FOUND st14 (sala OOS)                  -> Pop A (included in filter)
     961 = substituted
     403 = added
     350 = weight OOS (status 13)
     189 = partial pick
```

**Step 5 -- Same for HANA > 0 (335,879 lines)**
```
From the 335,879 HANA > 0 lines:
 315,650 = picker FOUND it (expected)
   7,187 = picker NOT FOUND st5 (phantom inventory)          -> Pop B core
      45 = picker NOT FOUND st3 (legacy OOS)                 -> Pop B (included)
       3 = picker NOT FOUND st14 (sala OOS)                  -> Pop B (included)
   6,245 = substituted
   3,388 = added
   2,236 = weight OOS (status 13)
   1,125 = partial pick
```

**Step 6 -- Run evidence_pack_0708.py for Pop A + Pop B**
```
Script: scripts/evidence_pack_0708.py
Input:  HANA parquets + orders CSV + 2 LIVE Redshift queries
        -> Redshift query 1: io_internal_orders (order status, canceledby, amountinvoice)
        -> Redshift query 2: io_picking (picking session proof)
Output: analysis/jul8/evidence_pack_jul8.xlsx
        analysis/jul8/evidence_pop_a_jul8.csv (1,402 lines)
        analysis/jul8/evidence_pop_b_jul8.csv (7,235 lines)
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
| `evidence_pack_0708.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0708.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0708.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
| `rs.py` | Redshift connection helper (persistent conn) | -- | YES |

### CSV read pattern (DuckDB)

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

Jul 8 CSV uses the same 6 VARCHAR overrides as Jul 6. Always apply these when reading any `all_orders_07XX_FULL.csv` via DuckDB to avoid type inference failures on empty-string fields.

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
| `analysis/jul8/evidence_pack_jul8.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul8/evidence_pop_a_jul8.csv` | 1,402 | Pop A full detail |
| `analysis/jul8/evidence_pop_b_jul8.csv` | 7,235 | Pop B full detail |
| `analysis/jul8/delay_proof_0708_preventable.parquet` | 15,551 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul8/delay_proof_0708_raw.xlsx` | 15,551 | Same, Excel format with summaries |

### HANA batch coverage note

Jul 8 has **full 24/24 batch coverage** (00:40 through 23:41), making it the cleanest day for analysis. Unlike Jul 6 (which had a 4-hour gap from 17:41 to 21:40) and Jul 1 (which started at an equivalent early batch), Jul 8's 00:40 first batch means virtually all orders have a prior HANA snapshot — only 3,832 lines (1.1%) fall into the no-HANA-snapshot bucket, vs 12,709 (3.0%) on Jul 6.

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 15,551 |
| Distinct orders | 9,889 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 61 |

The mean gap of **30 minutes** represents the average time between the most recent HANA batch (showing stock <= 0) and when the customer placed their order. This is the window during which a real-time VTEX stock update would have prevented the order from being placed. The max of 61 minutes is the longest such window on Jul 8 — substantially tighter than Jul 6's max of 239 minutes, consistent with 24/24 HANA batch coverage reducing the maximum gap between snapshots to ~60 minutes.
