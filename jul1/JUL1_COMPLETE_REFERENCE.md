# July 1, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-08. Use as template when repeating for Jul 6 / Jul 7.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0701_*/*.parquet` | 18 batches | 03:41–23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0701_FULL.csv` | 370,654 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~25K orders | Only 2 queries needed by evidence_pack.py |
| **VTEX SQS events** | `analysis/vtex_events_*.jsonl` | ~70 files | For delay_proof scripts only, NOT for Pop A/B |
| **Evidence CSVs** | `analysis/evidence_1273.csv`, `evidence_5870.csv` | 1,482 + 6,911 | Output of evidence_pack.py |
| **False alarms CSV** | `analysis/evidence_false_alarms_10903_v2.csv` | 10,903 | HANA ≤ 0 but picker found it |
| **Evidence Excel** | `analysis/evidence_pack_jul1.xlsx` | 3 sheets | Summary + Pop A + Pop B |

### Redshift ground truth (io_items, J-stores, Jul 1)

Total order lines from Redshift `io_items` (J-stores, Jul 1): **396,217**
CSV export has **370,654** lines — difference is export scope/timing.

### HANA batch times (18 batches, CLT)

```
03:41  04:40  05:41  06:40  07:40  08:41  09:41  10:40  11:41
12:41  13:40  14:40  15:41  16:43  17:40  18:40  19:40  23:41
```

Gap: no batch between 19:40 and 23:41 (stores closed overnight, 0 stock changes 03:41–06:40).
First batch at 03:41 → orders before this have no HANA snapshot.

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Full lookup table (16 statuses)

| ID | Code | Spanish | Description |
|----|------|---------|-------------|
| 0 | NO_PICKING | Sin pickear | Not picked yet |
| 1 | ADD | Pickeado | Picked successfully |
| 2 | SUSTITUTE | Sustituto | Substituted (legacy) |
| 3 | NO_STOCK | Sin stock | OOS (legacy — 0 rows on Jul 1) |
| 4 | VALIDATE_SUSTITUTE | Sustituto | Validated substitution |
| 5 | VALIDATE_NO_STOCK | Sin stock | Validated OOS — **main OOS status** |
| 6 | RE_PICKING_VALIDATE_SUSTITUTE | Sustituto | Re-pick substitution (legacy) |
| 7 | RE_PICKING_VALIDATE_NO_STOCK | Sin stock | Re-pick OOS (legacy — 0 rows on Jul 1) |
| 8 | PARTIAL_ADD | Faltante parcial | Partial pick (legacy) |
| 9 | FREE_PICKING | Compuesto | Composite (legacy) |
| 10 | CLIENT_DELITION | Eliminado cliente | Customer deleted item |
| 11 | NEW_ADD | Agregado por cliente | Customer added item |
| 12 | PARTIAL_ADD_VALIDATE | Faltante parcial | Partial weight pick |
| 13 | FREE_PICKING_VALIDATE | Compuesto | Weight/composite OOS — **NOT in Pop A/B** |
| 14 | ROOM_DELETION | Sin Stock | Room deletion OOS (0 rows on Jul 1) |
| 15 | PICK_VALIDATE | Compuesto | Validated composite (0 rows on Jul 1) |

### Statuses active on Jul 1 (Redshift, J-stores)

| Status | Code | Lines | CLP (M) | pickingquantity |
|--------|------|-------|---------|-----------------|
| **1** | Pickeado | **369,227** (93.2%) | 1,686.85 | > 0 always |
| **5** | Sin stock (OOS) | **9,844** (2.5%) | 55.02 | = 0 always |
| **4** | Sustituto | **8,621** (2.2%) | 46.02 | = 0 (replaced) |
| **11** | Agregado | **4,021** (1.0%) | 21.11 | > 0 always |
| **13** | Compuesto (weight OOS) | **3,054** (0.8%) | 48.01 | = 0 always |
| **12** | Faltante parcial | **1,450** (0.4%) | 14.11 | > 0 (partial) |
| | **TOTAL** | **396,217** | **1,871.12** | |

Statuses 0, 2, 3, 6, 7, 8, 9, 10, 14, 15 = **0 rows** on Jul 1. Our filter `status IN (3,5,7,14)` effectively = `WHERE status = 5`.

Grafana found rate formula excludes SUSTITUTO (4), SIN STOCK (5), and COMPUESTO (13) from numerator — so Grafana treats status 13 as OOS but our evidence pack does not.

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0701_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **370,654** |
| **Distinct orders** | **21,635** |
| Orders with processed lines (excl status 0) | 20,784 |
| Orders with only status 0 lines (not yet picked at export) | 851 |

### Cancellations (source: Redshift `jumbo_bo.io_internal_orders`)

337 orders cancelled on Jul 1 (status 41):

| Cancelled by | Count | % |
|-------------|-------|---|
| Customer | 275 | 81.6% |
| Jumbo staff (@jumbo.cl) | 55 | 16.3% |
| Backoffice | 4 | 1.2% |
| Cencosud staff (@cencosud.cl / @cencosud.com) | 3 | 0.9% |

Of 337 cancelled orders:
- **299** cancelled before picking (no item lines in CSV)
- **38** had item lines in CSV (cancelled after partial/full pick)
- Of those 38: 15 had OOS items — but all 15 were 100% wipeouts (every line OOS)
- **OOS is NOT driving cancellations** — customers cancel for other reasons

### Order-level fulfillment (source: local CSV, 20,784 orders with processed lines)

**Item lines (excl status 0):**

| Status | Meaning | Lines | % | CLP (M) |
|--------|---------|-------|---|---------|
| 1 | Picked OK | 329,598 | 93.2% | 1,500.1 |
| 5 | OOS | 8,700 | 2.5% | 48.0 |
| 4 | Substituted | 7,662 | 2.2% | 40.8 |
| 11 | Added by customer | 3,763 | 1.1% | 19.7 |
| 13 | Weight OOS | 2,657 | 0.8% | 41.0 |
| 12 | Partial pick | 1,329 | 0.4% | 12.2 |
| | **Total** | **353,709** | **100%** | **1,661.7** |

**Quantity:**

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 648,759 |
| Qty delivered (SUM pickingquantity) | 619,013 (95.4%) |

**Value (CLP):**

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,661,729,669** | 100% |
| Lost to OOS (status 5 + 13) | 88,995,906 | 5.4% |
| Lost to substitution (status 4) | 40,805,047 | 2.5% |

> Note: Substituted items (st4) have pickingquantity = 0 for the ORIGINAL item. The customer receives a substitute (st11 added line). Both the original and substitute lines exist in the data. Weight items (e.g., chicken, fish) have fractional originalquantity (e.g., 0.25 kg).

### Order impact summary

| Category | Orders | % of 20,784 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **10,003** | **48.1%** |
| Had any OOS (status 5 or 13) | 7,318 | 35.2% |
| Had substitution (status 4) | 5,605 | 27.0% |
| Had OOS AND substitution | 2,142 | 10.3% |
| Had partial pick (status 12) | 1,239 | 6.0% |

**51.9% of all processed orders had at least one item affected by OOS or substitution.**
**89M CLP lost to OOS alone.**

### Connection: orders → item lines → Pop A/B

```
21,635 orders in CSV (370,654 item lines)
├─ 20,784 with processed lines (excl status 0)
│  ├─ 10,003 clean (no OOS, no sub)
│  └─ 10,781 had OOS or sub or both
│     ├─ 8,393 status-5 OOS lines → Pop A (1,482) + Pop B (6,911)  ← Sections 5–6
│     ├─ 2,657 status-13 weight OOS lines → NOT in Pop A/B
│     └─ 7,662 status-4 substitution lines
└─ 851 orders with only status 0 (not yet picked at export)
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**370,654 order lines** from `all_orders_0701_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

For each order line, find the most recent HANA parquet batch BEFORE `fecha_creacion` and get `Ending_On_Hand_Qty` for that (item, store) pair.

```
370,654 total order lines
├─ Has HANA snapshot before order : 357,718
└─ No HANA snapshot              :  12,936  (order before 03:41 CLT or store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
357,718 with HANA snapshot
├─ HANA ≤ 0 (ERP says out of stock) :  14,523
└─ HANA > 0 (ERP says in stock)     : 343,195
```

### Step 4a: HANA ≤ 0 branch → 14,523 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (false alarm — HANA was wrong) | **10,903** | 47.25 |
| 5 | **Picker NOT FOUND** (confirmed OOS) = **Pop A** | **1,482** | 8.53 |
| 4 | Substituted | 1,058 | 6.13 |
| 13 | Weight item OOS | 296 | 4.40 |
| 11 | Added item | 297 | 1.41 |
| 12 | Partial weight pick | 142 | 1.02 |
| 0 | Not picked (CSV export artifact) | 345 | 0.02 |

Key finding: **10,903 false alarms** — HANA said stock was 0 but picker found it anyway. These are cases where HANA underreported stock (ERP book < physical shelf). File: `analysis/evidence_false_alarms_10903_v2.csv`

Sample OOS line (status 5):
> `v232195165jmch-01 | J403 | 1860109 | Yogurt Artesanal Mermelada Damasco 200 g | qty=4 | pick=0 | 6,360 CLP | 2026-07-01 09:53:48`
> Customer ordered 4 yogurts, picker found zero on shelf, HANA already showed ≤ 0 before the order.

### Step 4b: HANA > 0 branch → 343,195 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Picker FOUND it** (expected — HANA was right) | **306,607** | 1,398.49 |
| 5 | **Picker NOT FOUND** (phantom inventory) = **Pop B** | **6,911** | 37.59 |
| 4 | Substituted | 6,343 | 33.37 |
| 13 | Weight item OOS | 2,263 | 35.05 |
| 11 | Added item | 3,340 | 17.57 |
| 12 | Partial weight pick | 1,146 | 10.79 |
| 0 | Not picked (CSV export artifact) | 16,585 | 0.88 |

Key finding: **6,911 phantom inventory** — HANA said stock existed but picker found nothing. ERP book stock ≠ physical shelf.

### Step 4c: No HANA snapshot → 12,936 order lines

Orders placed before the first HANA batch (03:41 CLT) or for items/stores not in HANA.

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | Picker FOUND it | 12,088 | 54.33 |
| 5 | Picker NOT FOUND (OOS) | 307 | 1.87 |
| 4 | Substituted | 261 | 1.30 |
| 13 | Weight item OOS | 98 | 1.55 |
| 11 | Added item | 126 | 0.67 |
| 12 | Partial weight pick | 41 | 0.40 |
| 0 | Not picked (CSV artifact) | 15 | 0.00 |

### Step 5: The two populations we analyze

From the drill-down above:

```
Pop A = HANA ≤ 0 AND status 5 :  1,482  ( 8.53M CLP) — propagation gap
Pop B = HANA > 0 AND status 5 :  6,911  (37.59M CLP) — phantom inventory
───────────────────────────────────────────────────────
TOTAL analyzed                 :  8,393  (46.13M CLP)
```

### Status 0 note

Status 0 (`NO_PICKING` = "sin pickear") appears in the CSV (16,945 lines) but has **0 rows in Redshift** when queried later. These are orders that were not yet picked at CSV export time — they all got processed eventually. Status 0 exists only as a CSV timing artifact and is irrelevant to the analysis.

---

## 5. Population A — Propagation Gap (HANA ≤ 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed ≤ 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX → customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **1,482** |
| Total CLP | **8,531,584** (8.53M) |

### Confirmation tiers

| Tier | Meaning | Count | CLP |
|------|---------|-------|-----|
| 1 | PICKER MISS — same item picked at/before this order | 305 | 1,471,378 |
| 2 | RESTOCK LIKELY — picked only later + HANA rose after | 70 | 363,875 |
| 3 | PICKED LATER — only later, HANA flat | 76 | 424,745 |
| 4 | CONFIRMED STOCKOUT — multiple orders, all failed | 432 | 2,589,199 |
| 5 | SINGLE ATTEMPT — not delivered, cause unconfirmed | 599 | 3,682,387 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES — someone got it at/before** (Tier 1) | 305 | 1,471,378 |
| **YES — someone got it after** (Tier 2+3) | 146 | 788,620 |
| **NO — nobody got it** (Tier 4+5) | 1,031 | 6,271,586 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 1,482 | 100.0% |
| Picker marked OOS | 1,482 | 100.0% |
| Item not delivered (pick=0) | 1,482 | 100.0% |
| Order NOT cancelled | 1,480 | 99.9% |
| Order delivered/invoiced | 1,438 | 97.0% |
| Real picking session ran | 1,272 | 85.8% |
| Stock existed elsewhere (any order) | 451 | 30.4% |
| Stock present at pick time (before) | 305 | 20.6% |

### Sample rows — Pop A

**Tier 1 (PICKER MISS — strongest evidence):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| v232237296jmch-01 | J403 | 2052786 | Chocolates Mars Bolsa Mix 508 g | 7,122 | -1 | 3 |
| v232239068jmch-01 | J403 | 2071569 | Plantillas Hokkairo para Zapatos | 4,990 | 0 | 1 |
| v232233562jmch-01 | J403 | 2076048 | Lavaloza Quix Ultra Concentrado | 3,000 | 0 | 1 |

> Interpretation: HANA said 0 (or negative), picker marked OOS, but 1–3 OTHER orders picked the same item at/before this order. The stock was there — picker missed it.

**Tier 4 (CONFIRMED STOCKOUT — multiple orders, all failed):**

| order_id | store | refid | name | CLP | HANA qty | orders_tried |
|----------|-------|-------|------|-----|----------|-------------|
| v232233562jmch-01 | J403 | 1835024 | Cortador de Vegetales Spiralizer | 2,394 | 0 | 2 |
| v232209209jmch-01 | J403 | 1983863 | Mantel PVC Protector Redondo | 2,994 | -1 | 2 |
| v232266673jmch-01 | J403 | 1907741 | Tetera Vidrio 800 ml | 3,594 | -7 | 4 |

> Interpretation: HANA said 0 (or negative), multiple orders tried this item all day, ALL failed. Real stockout confirmed by both HANA and picker, across multiple independent attempts.

---

## 6. Population B — Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock ≠ physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **6,911** |
| Total CLP | **37,594,544** (37.60M) |

### Confirmation tiers

| Tier | Meaning | Count | CLP |
|------|---------|-------|-----|
| 1 | PICKER MISS — same item picked at/before this order | 2,147 | 10,616,064 |
| 2 | RESTOCK LIKELY — picked only later + HANA rose after | 253 | 1,360,641 |
| 3 | PICKED LATER — only later, HANA flat | 466 | 2,528,479 |
| 4 | CONFIRMED STOCKOUT — multiple orders, all failed | 1,769 | 9,559,227 |
| 5 | SINGLE ATTEMPT — not delivered, cause unconfirmed | 2,276 | 13,530,133 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES — someone got it at/before** (Tier 1) | 2,147 | 10,616,064 |
| **YES — someone got it after** (Tier 2+3) | 719 | 3,889,120 |
| **NO — nobody got it** (Tier 4+5) | 4,045 | 23,089,360 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 6,911 | 100.0% |
| Picker marked OOS | 6,911 | 100.0% |
| Item not delivered (pick=0) | 6,911 | 100.0% |
| Order NOT cancelled | 6,896 | 99.8% |
| Order delivered/invoiced | 6,707 | 97.0% |
| Real picking session ran | 5,860 | 84.8% |
| Stock existed elsewhere (any order) | 2,866 | 41.5% |
| Stock present at pick time (before) | 2,147 | 31.1% |

### Sample rows — Pop B

**Tier 1 (PICKER MISS — HANA showed stock, picker missed it):**

| order_id | store | refid | name | CLP | HANA qty | got_before |
|----------|-------|-------|------|-----|----------|------------|
| v232214199jmch-01 | J403 | 1995694-KG | Bistec de Asiento Bandeja 300 g | 6,228 | 0.67 | 2 |
| v232230070jmch-01 | J403 | 2044469 | Cinnamon Bread Jumbo un. | 3,490 | 6 | 5 |
| v232238985jmch-01 | J403 | 255541 | Blanqueador Ropa Klären Ultra 250 g | 3,290 | 13 | 1 |

> Interpretation: HANA said stock > 0 (even 13 units in one case), picker marked OOS, but other orders successfully picked the same item. Classic phantom inventory — the stock exists but the picker couldn't find it or chose not to.

---

## 7. Combined Summary

| | Pop A (HANA ≤ 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 1,482 | 6,911 | **8,393** |
| **CLP** | 8,531,584 | 37,594,544 | **46,126,128** |
| YES at/before | 305 (20.6%) | 2,147 (31.1%) | 2,452 (29.2%) |
| YES after | 146 (9.9%) | 719 (10.4%) | 865 (10.3%) |
| NO nobody | 1,031 (69.6%) | 4,045 (58.5%) | 5,076 (60.5%) |

### What's NOT in Pop A/B

| Category | Lines | CLP (M) | Why excluded |
|----------|-------|---------|--------------|
| Status 13 weight OOS (HANA ≤ 0) | 296 | 4.40 | Filter is `status IN (3,5,7,14)` — misses 13 |
| Status 13 weight OOS (HANA > 0) | 2,263 | 35.05 | Same |
| Status 13 weight OOS (no HANA) | 98 | 1.55 | Same |
| Status 5 OOS (no HANA snapshot) | 307 | 1.87 | No HANA batch before order |
| **Total excluded OOS** | **2,964** | **42.87** | |

---

## 8. Methodology: How We Got Here

### Step-by-step (what we did, in order)

**Step 1 — Export all order lines from Redshift**
```
Script: scripts/export_orders_0701.py
Query:  SELECT * FROM jumbo_bo.io_items ii
        JOIN jumbo_bo.io_internal_orders io ON ii.key = io.idorder
        WHERE ii.dateupdate >= '2026-07-01' AND ii.dateupdate < '2026-07-02'
        AND io.storename LIKE 'J%'
Output: exports/cencommerce_quiebre/all_orders_0701_FULL.csv (370,654 rows)
```

**Step 2 — Download HANA parquets from S3**
```
Script: scripts/download_latest_batch.sh (or aws s3 cp)
Bucket: s3://cencosud.prod.sm.cl.raw/.../vw_daily_nrt_0701_*/
Output: data_samples/vw_daily_nrt_0701_*/*.parquet (18 batches)
Needs:  AWS SSO + VPN
```

**Step 3 — Generate delay_proof_0701_preventable_v2.parquet**
```
Script: scripts/delay_proof_0701.py
Input:  HANA parquets + orders CSV (local only, no Redshift)
Output: analysis/delay_proof_0701_preventable_v2.parquet (14,523 rows)
What:   For EVERY order line, lookup most recent HANA batch before order.
        Filter to HANA ≤ 0 → these are orders where HANA said OOS before customer ordered.
        This gives us the 14,523 "HANA was out of stock" lines.
```

**Step 4 — Break down the 14,523 by picking status**
```
From the 14,523 HANA ≤ 0 lines:
  10,903 = picker FOUND it (false alarm — HANA wrong)  → evidence_false_alarms_10903_v2.csv
   1,482 = picker NOT FOUND (confirmed OOS)             → Pop A
   1,058 = substituted
     296 = weight OOS (status 13)
     142 = partial pick
     297 = added
     345 = not picked (CSV artifact)
```

**Step 5 — Same for HANA > 0 (343,195 lines)**
```
From the 343,195 HANA > 0 lines:
 306,607 = picker FOUND it (expected)
   6,911 = picker NOT FOUND (phantom inventory)         → Pop B
   6,343 = substituted
   2,263 = weight OOS (status 13)
   1,146 = partial pick
   3,340 = added
  16,585 = not picked (CSV artifact)
```

**Step 6 — Run evidence_pack.py for Pop A + Pop B**
```
Script: scripts/evidence_pack.py
Input:  HANA parquets + orders CSV + 2 LIVE Redshift queries
        → Redshift query 1: io_internal_orders (order status, canceledby, amountinvoice)
        → Redshift query 2: io_picking (picking session proof — did a real picker work this?)
Output: analysis/evidence_pack_jul1.xlsx
        analysis/evidence_1273.csv (Pop A = 1,482 lines)
        analysis/evidence_5870.csv (Pop B = 6,911 lines)
What:   For each Pop A/B line, attaches:
        - HANA daily profile (all 18 batches for that item+store)
        - Cross-order corroboration (did another order pick the same item?)
        - Timing corroboration (before or after this order?)
        - Restock signal (did HANA qty rise after?)
        - Redshift order status (delivered? invoiced? cancelled?)
        - Redshift picking session (real picker or system?)
        → Assigns confirmation tier (1-5) based on evidence
```

**Step 7 — Compute 3-bucket summary**
```
From confirmation tiers:
  Tier 1                        = YES, someone got it AT/BEFORE this order
  Tier 2 + Tier 3               = YES, someone got it AFTER this order
  Tier 4 + Tier 5               = NO, nobody got this item all day
```

---

## 9. Scripts Reference

| Script | What it does | Data sources | Redshift needed? |
|--------|-------------|--------------|------------------|
| `evidence_pack.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0701.py` | HANA lookup for all orders → preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0701.py` | Export all order lines from Redshift | Redshift io_items + io_internal_orders | YES |
| `rs.py` | Redshift connection helper (persistent conn) | — | YES |
| `drain_sqs_all.py` | SQS non-destructive drain for VTEX events | AWS SQS | NO |

### CSV read gotcha (DuckDB)

```python
read_csv_auto('path.csv',
    nullstr='None',
    types={'dateupdate': 'VARCHAR',
           'inicio_picking': 'VARCHAR',
           'fin_picking': 'VARCHAR'})
```
Without this, DuckDB auto-detects dateupdate as TIMESTAMP and chokes on 'None' strings.

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
- `got_before > 0` → stock was on shelf while THIS order was being picked → picker miss (Tier 1)
- `got_after > 0` → stock appeared later → restock or picker miss (Tier 2/3)
- Nobody got it → all orders for this item failed → real stockout or single-attempt (Tier 4/5)

---

## 11. Open Questions / Decisions

1. **Status 13 (Compuesto, weight OOS):** 2,657 lines in CSV / 3,054 in Redshift, ~48M CLP total. All weight/deli items (chicken, fish, beef, pasta by kg). Not in Pop A/B because filter is `status IN (3,5,7,14)`. Grafana already treats them as OOS. Decision: include as separate analysis or keep out of scope?

2. **"Nobody got it" bucket (5,076 lines, 60.5%):** Not necessarily genuine stockouts — could be low-demand items where only one order tried. Tier 5 (single attempt) is weakest evidence; Tier 4 (multiple orders all failed) is stronger.

3. **10,903 false alarms:** HANA said 0 but picker found it. This is the inverse problem — HANA UNDER-reporting. Not OOS, but shows HANA inaccuracy in both directions.

---

## 12. Reproducing for Jul 6 / Jul 7

### Current status (as of 2026-07-08)

| Resource | Jul 6 | Jul 7 |
|----------|-------|-------|
| **HANA parquets** | ✅ 17 batches (04:40–23:41) | ✅ 24 batches (00:41–23:41) |
| **Orders CSV** | ❌ Not exported | ❌ Not exported |
| **Export script** | ✅ `scripts/export_orders_0706.py` | ❌ Need to create |
| **VTEX SQS** | ✅ `vtex_all_20260706_204454.jsonl` (556M) | ✅ `vtex_all_20260707_192702.jsonl` (929M) |
| **evidence_pack.py** | Needs date parameters changed | Same |

### HANA batch times

**Jul 6 (17 batches):**
```
04:40  05:41  06:40  07:40  08:40  09:40  10:40  11:40  12:41
13:40  14:40  15:41  16:41  17:41  21:40  22:40  23:41
```
Note: Gap from 17:41 to 21:40 (3 hours missing vs Jul 1's 4-hour gap). No 03:41 batch (Jul 1 had one).

**Jul 7 (24 batches):**
```
00:41  01:41  02:40  03:41  04:40  05:41  06:40  07:40  08:40  09:41  10:41  11:40
12:40  13:40  14:40  15:40  16:41  17:41  18:41  19:40  20:41  21:40  22:41  23:41
```
Note: Full 24-hour coverage including overnight batches. Most complete day we have.

### Steps to reproduce

```bash
# ────────────────────────────────────────────────
# STEP 1: Export orders CSV (needs VPN + Redshift)
# ────────────────────────────────────────────────

# Jul 6 (script exists)
python3.11 scripts/export_orders_0706.py

# Jul 7 (copy and modify export_orders_0706.py)
# Change: '2026-07-06' → '2026-07-07', '2026-07-07' → '2026-07-08'
# Change: output filename to all_orders_0707_FULL.csv
cp scripts/export_orders_0706.py scripts/export_orders_0707.py
# Then edit dates and output path
python3.11 scripts/export_orders_0707.py

# ────────────────────────────────────────────────
# STEP 2: Verify HANA parquets
# ────────────────────────────────────────────────
ls data_samples/vw_daily_nrt_0706_*/   # should show 17 folders
ls data_samples/vw_daily_nrt_0707_*/   # should show 24 folders

# ────────────────────────────────────────────────
# STEP 3: Run delay_proof (local only, no Redshift)
# ────────────────────────────────────────────────
# Modify scripts/delay_proof_0701.py:
#   - HANA glob: vw_daily_nrt_0706_* (or 0707)
#   - CSV path: all_orders_0706_FULL.csv (or 0707)
#   - Output: delay_proof_0706_preventable_v2.parquet (or 0707)
python3.11 scripts/delay_proof_0706.py

# ────────────────────────────────────────────────
# STEP 4: Run evidence_pack (needs VPN for Redshift)
# ────────────────────────────────────────────────
# Modify scripts/evidence_pack.py:
#   - HANA glob: vw_daily_nrt_0706_*
#   - CSV path: all_orders_0706_FULL.csv
#   - Redshift date filter: '2026-07-06'
#   - Output: evidence_pack_jul6.xlsx
python3.11 scripts/evidence_pack.py

# ────────────────────────────────────────────────
# STEP 5: Verify outputs
# ────────────────────────────────────────────────
ls -la analysis/evidence_pack_jul6.xlsx
wc -l analysis/evidence_*.csv
```

### Key files per date

| File | Jul 1 | Jul 6 | Jul 7 |
|------|-------|-------|-------|
| Orders CSV | `all_orders_0701_FULL.csv` | ❌ not exported | ❌ not exported |
| HANA parquets | `vw_daily_nrt_0701_*` (18 batches) | `vw_daily_nrt_0706_*` (17 batches) | `vw_daily_nrt_0707_*` (24 batches) |
| VTEX SQS | `vtex_events_*.jsonl` (70 files) | `vtex_all_20260706_204454.jsonl` (556M) | `vtex_all_20260707_192702.jsonl` (929M) |
| Export script | `export_orders_0701.py` | `export_orders_0706.py` | ❌ need to create |
| Evidence Excel | `evidence_pack_jul1.xlsx` | ❌ pending | ❌ pending |
| Pop A CSV | `evidence_1273.csv` (1,482) | ❌ pending | ❌ pending |
| Pop B CSV | `evidence_5870.csv` (6,911) | ❌ pending | ❌ pending |
| False alarms | `evidence_false_alarms_10903_v2.csv` | ❌ pending | ❌ pending |
| Preventable parquet | `delay_proof_0701_preventable_v2.parquet` | ❌ pending | ❌ pending |

### Blockers

1. **Orders CSV export** — needs VPN + Redshift connection. Run `export_orders_0706.py` first.
2. **evidence_pack.py** — needs VPN for 2 live Redshift queries (io_internal_orders + io_picking).
3. **Jul 7 export script** — copy from 0706 and change dates.

### DuckDB read gotcha (same for all dates)

```python
read_csv_auto('path.csv',
    nullstr='None',
    types={'dateupdate': 'VARCHAR',
           'inicio_picking': 'VARCHAR',
           'fin_picking': 'VARCHAR'})
```
