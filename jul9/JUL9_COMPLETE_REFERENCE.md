# July 9, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-11. Same methodology as JUL1_COMPLETE_REFERENCE.md and JUL6_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0709_*/*.parquet` | 24 batches | 00:41–23:42 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0709_FULL.csv` | 367,299 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~22K orders | Only 2 queries needed by evidence_pack_0709.py |
| **Evidence CSVs** | `analysis/jul9/evidence_pop_a_jul9.csv`, `evidence_pop_b_jul9.csv` | 1,488 + 6,527 | Output of evidence_pack_0709.py |
| **Evidence Excel** | `analysis/jul9/evidence_pack_jul9.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul9/delay_proof_0709_preventable.parquet` | 17,261 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul9/delay_proof_0709_raw.xlsx` | 17,261 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:41  01:40  02:40  03:40  04:41  05:40  06:41  07:40  08:41  09:40  10:40  11:41
12:41  13:41  14:41  15:40  16:41  17:41  18:41  19:41  20:40  21:41  22:40  23:42
```

Full 24/24 coverage — every hour of the day represented, including overnight batches.
No gaps in coverage (contrast with Jul 6's 4-hour gap from 17:41 to 21:40).

HANA stats: 541,334,228 rows loaded, 502,801,995 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 9 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) |
|--------|------|-------|---|---------|
| **1** | Pickeado | **344,333** (93.7%) | 93.7% | 1,578.12 |
| **5** | Sin stock (OOS) | **8,142** (2.2%) | 2.2% | 47.32 |
| **4** | Sustituto | **6,806** (1.9%) | 1.9% | 37.83 |
| **11** | Agregado | **3,940** (1.1%) | 1.1% | 19.50 |
| **13** | Compuesto (weight OOS) | **2,731** (0.7%) | 0.7% | 52.08 |
| **12** | Faltante parcial | **1,341** (0.4%) | 0.4% | 14.09 |
| **0** | Sin pickear | **5** (0.0%) | 0.0% | 0.00 |
| **14** | Sin Stock (sala) | **1** (0.0%) | 0.0% | 0.00 |
| | **TOTAL** | **367,299** | 100% | **1,748.95** |

Jul 9 differences from prior days:
- **OOS rate 2.2%** — lowest of all five days measured (Jul 1=2.5%, Jul 6=3.3%, Jul 7=2.9%, Jul 8=2.4%)
- **Status 0 has only 5 lines** (effectively all orders processed by export time)
- **Status 14 has 1 line** (sala deletion — captured by our `status IN (3,5,7,14)` filter)
- Our filter `status IN (3,5,7,14)` captures: st5 (8,142) + st14 (1) = 8,143 lines at status level

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0709_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **367,299** |
| **Distinct orders** | **22,546** |
| Qty ordered (SUM originalquantity) | 654,907 |
| Qty delivered (SUM pickingquantity) | 626,933 (95.7%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,748,948,384** | 100% |
| Lost to OOS (status 5 + 13) | 99,398,978 | 5.7% |
| Lost to substitution (status 4) | 37,831,136 | 2.2% |

### Order impact summary

| Category | Orders | % of 22,546 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **11,958** | **53.0%** |
| Had any OOS (status 5 or 13) | 7,312 | 32.4% |
| Had substitution (status 4) | 5,237 | 23.2% |
| Had OOS AND substitution | 1,961 | 8.7% |
| Had partial pick (status 12) | 1,227 | 5.4% |

**47.0% of all processed orders had at least one item affected by OOS or substitution.**
**99.4M CLP lost to OOS alone (st5 + st13).**

### Connection: orders -> item lines -> Pop A/B

```
22,546 orders in CSV (367,299 item lines)
|- 22,546 with processed lines
|  |- 11,958 clean (no OOS, no sub)
|  |- 10,588 had OOS or sub or both
|     |- 8,013 status 5/14 OOS lines -> Pop A (1,486) + Pop B (6,527)  <- Sections 5-6
|     |-  2,731 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  6,806 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**367,299 order lines** from `all_orders_0709_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

For each order line, find the most recent HANA parquet batch BEFORE `fecha_creacion` and get `Ending_On_Hand_Qty` for that (item, store) pair.

```
367,299 total order lines
|- Has HANA snapshot before order : 362,735
|- No HANA snapshot              :   4,564  (order before 00:41 CLT or store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
362,735 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  17,261
|- HANA > 0 (ERP says in stock)      : 345,474
```

### Step 4a: HANA <= 0 branch -> 17,261 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (false alarm -- HANA was wrong) | **13,819** | 60.16 |
| 5 | **Picker NOT FOUND** (confirmed OOS) = **Pop A** | **1,485** | 9.30 |
| 4 | Substituted | 956 | 5.94 |
| 13 | Weight item OOS | 389 | 5.28 |
| 11 | Added item | 389 | 1.66 |
| 12 | Partial weight pick | 222 | 1.76 |
| 14 | Sin Stock (sala) | 1 | 0.00 |

Key finding: **13,819 false alarms** -- HANA said stock was 0 but picker found it anyway. These are cases where HANA underreported stock (ERP book < physical shelf).

Note: Pop A is 1,485 from delay_proof (st5 only) vs 1,486 from stats query vs 1,488 from evidence_pack (st5 + st14 combined via `status IN (3,5,7,14)`, plus 2 additional lines from st3/st14 inclusion). The 2-line delta is from st3/st14 inclusion in the evidence_pack filter.

### Step 4b: HANA > 0 branch -> 345,474 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (expected -- HANA was right) | **326,226** | 1,498.09 |
| 5 | **Picker NOT FOUND** (phantom inventory) = **Pop B** | **6,527** | 37.21 |
| 4 | Substituted | 5,793 | 31.55 |
| 13 | Weight item OOS | 2,312 | 46.40 |
| 11 | Added item | 3,512 | 17.65 |
| 12 | Partial weight pick | 1,099 | 12.09 |
| 0 | Not picked (CSV export artifact) | 5 | 0.00 |

Key finding: **6,527 phantom inventory** at status-5 level -- HANA said stock existed but picker found nothing. ERP book stock != physical shelf. Pop B is captured in full by evidence_pack (6,527 lines).

### Step 4c: No HANA snapshot -> 4,564 order lines

Orders placed before the first HANA batch (00:41 CLT) or for items/stores not in HANA.
This is the smallest no-snapshot pool of all five days analyzed, due to full 24/24 batch coverage.

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | Picker FOUND it | 4,288 | 19.87 |
| 5 | Picker NOT FOUND (OOS) | 130 | 0.81 |
| 4 | Substituted | 57 | 0.34 |
| 13 | Weight item OOS | 30 | 0.40 |
| 11 | Added item | 39 | 0.19 |
| 12 | Partial weight pick | 20 | 0.23 |

### Step 5: The two populations we analyze

From the drill-down above:

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  1,486  ( 9.30M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) :  6,527  (37.21M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              :  8,013  (46.51M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **1,488** (evidence_pack count, includes st3/st14) |
| Total CLP | **9,300,994** (9.30M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 314 | 21.1% | 1,475,217 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 86 | 5.8% | 463,314 |
| 3 | PICKED LATER -- only later, HANA flat | 67 | 4.5% | 480,516 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 427 | 28.7% | 2,557,539 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 594 | 39.9% | 4,330,790 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 314 | 1,475,217 |
| **YES -- someone got it after** (Tier 2+3) | 153 | 943,830 |
| **NO -- nobody got it** (Tier 4+5) | 1,021 | 6,888,329 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 1,488 | 100.0% |
| Picker marked OOS | 1,488 | 100.0% |
| Item not delivered (pick=0) | 1,488 | 100.0% |
| Order NOT cancelled | 1,484 | 99.7% |
| Order delivered/invoiced | 1,403 | 94.3% |
| Real picking session ran | 1,181 | 79.4% |

### Sample rows -- Pop A

**Tier 1 (PICKER MISS -- strongest evidence):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| Example Jul 9 Tier-1 row | J403 | -- | Item picked by other orders same slot | -- | <= 0 | >= 1 |

> Interpretation: HANA said 0 (or negative), picker marked OOS, but other orders picked the same item at/before this order. The stock was there -- picker missed it.

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | name | CLP | HANA qty | orders_tried |
|----------|-------|-------|------|-----|----------|-------------|
| Example Jul 9 Tier-4 row | J414 | -- | Item tried by multiple orders, all OOS | -- | 0 | >= 2 |

> Interpretation: HANA said 0, multiple orders tried this item all day, ALL failed. Real stockout confirmed by both HANA and picker across multiple independent attempts.

---

## 6. Population B -- Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock != physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **6,527** |
| Total CLP | **37,211,211** (37.21M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS -- same item picked at/before this order | 1,827 | 28.0% | 9,806,768 |
| 2 | RESTOCK LIKELY -- picked only later + HANA rose after | 289 | 4.4% | 1,675,751 |
| 3 | PICKED LATER -- only later, HANA flat | 401 | 6.1% | 2,114,241 |
| 4 | CONFIRMED STOCKOUT -- multiple orders, all failed | 1,675 | 25.7% | 8,978,350 |
| 5 | SINGLE ATTEMPT -- not delivered, cause unconfirmed | 2,335 | 35.8% | 14,636,101 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 1,827 | 9,806,768 |
| **YES -- someone got it after** (Tier 2+3) | 690 | 3,789,992 |
| **NO -- nobody got it** (Tier 4+5) | 4,010 | 23,614,451 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 6,527 | 100.0% |
| Picker marked OOS | 6,527 | 100.0% |
| Item not delivered (pick=0) | 6,527 | 100.0% |
| Order NOT cancelled | 6,508 | 99.7% |
| Order delivered/invoiced | 6,168 | 94.5% |
| Real picking session ran | 5,143 | 78.8% |

### Sample rows -- Pop B

**Tier 1 (PICKER MISS -- HANA showed stock, picker missed it):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| Example Jul 9 Tier-1 row | J411 | -- | Item with HANA > 0 picked by other order | -- | > 0 | >= 1 |

> Interpretation: HANA said stock > 0, picker marked OOS, but other orders successfully picked the same item. Classic phantom inventory -- the stock exists but the picker couldn't find it or chose not to.

---

## 7. Combined Summary

| | Pop A (HANA <= 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 1,486 (1,488 in evidence_pack) | 6,527 | **8,013** |
| **CLP** | 9,300,994 | 37,211,211 | **46,512,205** |
| YES at/before | 314 (21.1%) | 1,827 (28.0%) | 2,141 (26.7%) |
| YES after | 153 (10.3%) | 690 (10.6%) | 843 (10.5%) |
| NO nobody | 1,021 (68.6%) | 4,010 (61.4%) | 5,031 (62.8%) |

### Five-day comparison: Jul 1 through Jul 9

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul 8 | Jul 9 |
|--------|-------|-------|-------|-------|-------|
| Order lines | 370,654 | 419,497 | ~390K | ~370K | 367,299 |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | 2.4% | **2.2%** |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | 49.1% | **47.0%** |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | -- | -- | 99.4M |
| Pop A lines | 1,482 | 2,608 | 1,978 | 1,402 | **1,486** |
| Pop A CLP | 8.53M | 13.88M | 11.57M | 8.28M | 9.30M |
| Pop B lines | 6,911 | 10,613 | 8,510 | 7,235 | **6,527** |
| Pop B CLP | 37.59M | 55.56M | 46.83M | 39.63M | 37.21M |
| Combined lines | 8,393 | 13,221 | 10,488 | 8,637 | **8,013** |
| Combined CLP | 46.13M | 69.44M | 58.40M | 47.91M | 46.51M |
| False alarms (HANA<=0, picker found) | 10,903 | 13,980 | -- | -- | 13,819 |
| HANA batches | 18 | 17 | 24 | 24 | 24 |

**Jul 9 is the best day measured so far.** Lowest OOS rate (2.2%), lowest affected-order % (47.0%), fewest Pop B cases (6,527), and smallest combined CLP (46.5M). Still, 46.5M CLP in daily preventable OOS impact represents a material, ongoing operational loss.

### What's NOT in Pop A/B

| Category | Lines | CLP (M) | Why excluded |
|----------|-------|---------|--------------|
| Status 13 weight OOS (HANA <= 0) | 389 | 5.28 | Filter is `status IN (3,5,7,14)` -- misses 13 |
| Status 13 weight OOS (HANA > 0) | 2,312 | 46.40 | Same |
| Status 13 weight OOS (no HANA) | 30 | 0.40 | Same |
| Status 5 OOS (no HANA snapshot) | 130 | 0.81 | No HANA batch before order |
| Status 14 (no HANA snapshot) | -- | -- | Captured in evidence_pack via filter |
| **Total excluded OOS** | **2,861** | **52.89** | |

---

## 8. Methodology: How We Got Here

### Step-by-step (what we did, in order)

**Step 1 -- Export all order lines from Redshift**
```
Script: scripts/export_orders_0709.py
Query:  SELECT * FROM jumbo_bo.io_items ii
        JOIN jumbo_bo.vw_master_pickers_data p ON ii.key = p.id_raw
        WHERE p.fecha_creacion >= '2026-07-09' AND p.fecha_creacion < '2026-07-10'
        AND p.id_tienda LIKE 'J%'
Output: exports/cencommerce_quiebre/all_orders_0709_FULL.csv (367,299 rows)
```

**Step 2 -- Download HANA parquets from S3**
```
Script: scripts/download_latest_batch.sh (or aws s3 cp)
Bucket: s3://cencosud.prod.sm.cl.raw/.../vw_daily_nrt_0709_*/
Output: data_samples/vw_daily_nrt_0709_*/*.parquet (24 batches)
Needs:  AWS SSO + VPN
```

**Step 3 -- Generate delay_proof_0709_preventable.parquet**
```
Script: scripts/delay_proof_0709.py
Input:  HANA parquets + orders CSV (local only, no Redshift)
Output: analysis/jul9/delay_proof_0709_preventable.parquet (17,261 rows)
        analysis/jul9/delay_proof_0709_raw.xlsx
What:   For EVERY order line, lookup most recent HANA batch before order.
        Filter to HANA <= 0 -> these are orders where HANA said OOS before customer ordered.
        This gives us the 17,261 "HANA was out of stock" lines.
```

**Step 4 -- Break down the 17,261 by picking status**
```
From the 17,261 HANA <= 0 lines:
  13,819 = picker FOUND it (false alarm -- HANA wrong)
   1,485 = picker NOT FOUND (confirmed OOS)              -> Pop A (1,486/1,488 with st14 inclusion)
     956 = substituted
     389 = weight OOS (status 13)
     222 = partial pick
     389 = added
       1 = sin stock sala (status 14)
```

**Step 5 -- Same for HANA > 0 (345,474 lines)**
```
From the 345,474 HANA > 0 lines:
 326,226 = picker FOUND it (expected)
   6,527 = picker NOT FOUND (phantom inventory)          -> Pop B
   5,793 = substituted
   2,312 = weight OOS (status 13)
   1,099 = partial pick
   3,512 = added
       5 = not picked (status 0, CSV artifact)
```

**Step 6 -- Run evidence_pack_0709.py for Pop A + Pop B**
```
Script: scripts/evidence_pack_0709.py
Input:  HANA parquets + orders CSV + 2 LIVE Redshift queries
        -> Redshift query 1: io_internal_orders (order status, canceledby, amountinvoice)
        -> Redshift query 2: io_picking (picking session proof)
Output: analysis/jul9/evidence_pack_jul9.xlsx
        analysis/jul9/evidence_pop_a_jul9.csv (1,488 lines)
        analysis/jul9/evidence_pop_b_jul9.csv (6,527 lines)
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
| `evidence_pack_0709.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0709.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0709.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
| `rs.py` | Redshift connection helper (persistent conn) | -- | YES |

### CSV read gotcha (DuckDB) -- Jul 9 specific

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

Same six VARCHAR overrides as Jul 6 and Jul 7/8. The three extra fields (`venta_neta`, `items_solicitados`, `items_faltantes`) contain empty strings for some orders and will cause DuckDB type inference to fail without explicit VARCHAR casting.

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

**Cross-order corroboration:** For each OOS line, check if the SAME (store, item) was picked successfully in ANY other order that day.
- `got_before > 0` -> stock was on shelf while THIS order was being picked -> picker miss (Tier 1)
- `got_after > 0` -> stock appeared later -> restock or picker miss (Tier 2/3)
- Nobody got it -> all orders for this item failed -> real stockout or single-attempt (Tier 4/5)

---

## 11. Key Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul9/evidence_pack_jul9.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul9/evidence_pop_a_jul9.csv` | 1,488 | Pop A full detail |
| `analysis/jul9/evidence_pop_b_jul9.csv` | 6,527 | Pop B full detail |
| `analysis/jul9/delay_proof_0709_preventable.parquet` | 17,261 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul9/delay_proof_0709_raw.xlsx` | 17,261 | Same, Excel format with summaries |

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 17,261 |
| Distinct orders | 10,464 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 62 |

**Comparison to prior days:**

| Day | Total preventable | Distinct orders | Gap mean (min) | Gap max (min) |
|-----|-----------------|-----------------|----------------|---------------|
| Jul 1 | 14,523 | -- | -- | -- |
| Jul 6 | 19,241 | 11,505 | 40.4 | 239 |
| Jul 7 | -- | -- | -- | -- |
| Jul 8 | -- | -- | -- | -- |
| Jul 9 | **17,261** | **10,464** | **30** | **62** |

Jul 9 shows a substantially tighter gap window (max 62 min vs Jul 6's 239 min), meaning the HANA-to-VTEX propagation delay is smaller on this day. The mean gap of 30 minutes is the lowest recorded.
