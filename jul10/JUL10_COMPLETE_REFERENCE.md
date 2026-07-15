# July 10, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-15. Same methodology as JUL8_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0710_*/*.parquet` | 24 batches | 00:40-23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0710_FULL.csv` | 419,133 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~25K orders | Only 2 queries needed by evidence_pack_0710.py |
| **Evidence CSVs** | `analysis/jul10/evidence_pop_a_jul10.csv`, `evidence_pop_b_jul10.csv` | 1,789 + 7,295 | Output of evidence_pack_0710.py |
| **Evidence Excel** | `analysis/jul10/evidence_pack_jul10.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul10/delay_proof_0710_preventable.parquet` | 19,165 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul10/delay_proof_0710_raw.xlsx` | 19,165 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:40  01:41  02:40  03:40  04:40  05:41  06:40  07:40  08:40  09:40  10:41  11:41
12:41  13:42  14:40  15:41  16:41  17:40  18:40  19:40  20:41  21:41  22:40  23:40
```

Full 24/24 coverage. First batch at 00:40 means all orders have a HANA snapshot.

HANA stats: 541,448,395 rows loaded, 502,941,454 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 10 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) | Example |
|--------|------|-------|---|---------|---------|
| **1** | Pickeado | **393,540** | 93.9% | 1787.54 | Whisky Johnnie Walker Black Label 12 Años 1 L @ J411 (179,940 CLP) |
| **5** | Sin stock (validado) | **8,814** | 2.1% | 50.65 | Cafetera Nespresso Essenza White Single - Paris Electro @ J414 (129,990 CLP) |
| **4** | Sustituto | **7,586** | 1.8% | 42.3 | Pack Whisky Buchanan"s 12 Años 40° 750 cc + Copa @ J512 (113,358 CLP) |
| **11** | Agregado | **4,520** | 1.1% | 22.84 | Camarón Cocido Sin Cáscara M 500 g @ J534 (85,520 CLP) |
| **13** | Compuesto (weight OOS) | **3,088** | 0.7% | 58.36 | Filete Al Vacío kg @ J660 (418,730 CLP) |
| **12** | Faltante parcial | **1,584** | 0.4% | 16.61 | Bolsa de Basura Virutex Carga Pesada 110 x 120 cm 5 un. @ J411 (134,820 CLP) |
| **14** | Sin Stock (sala) | **1** | 0.0% | 0.0 | Hielo Campana Cuisine & Co 2.5 kg @ J414 (2,170 CLP) |
| | **TOTAL** | **419,133** | 100% | **1978.32** | |

Jul 10 (Friday):
- **OOS rate 2.1%** (status 5: 8,814 lines)
- **Affected orders 48.5%** of 25,683
- Our filter `status IN (3,5,7,14)` captures relevant OOS lines

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0710_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **419,133** |
| **Distinct orders** | **25,683** |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 736,184 |
| Qty delivered (SUM pickingquantity) | 706,791 (96.0%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,978,316,117** | 100% |
| Lost to OOS (status 5 + 13) | 109,011,813 | 5.5% |
| Lost to substitution (status 4) | 42,302,522 | 2.1% |

### Order impact summary

| Category | Orders | % of 25,683 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **13,220** | **51.5%** |
| Had any OOS (status 5 or 13) | 8,087 | 31.5% |
| Had substitution (status 4) | 5,883 | 22.9% |
| Had OOS AND substitution | 2,102 | 8.2% |
| Had partial pick (status 12) | 1,460 | 5.7% |

**48.5% of all processed orders had at least one item affected by OOS or substitution.**
**109.0M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
25,683 orders in CSV (419,133 item lines)
|- 25,683 with processed lines
|  |- 13,220 clean (no OOS, no sub)
|  |- 12,463 had OOS or sub or both
|     |-  8,815 status 3/5/14 OOS lines -> Pop A (1,789) + Pop B (7,295)  <- Sections 5-6
|     |-  3,088 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  7,586 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**419,133 order lines** from `all_orders_0710_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

```
419,133 total order lines
|- Has HANA snapshot before order : 413,915
|- No HANA snapshot               :   5,218  (store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
413,915 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  19,165
|- HANA > 0 (ERP says in stock)      : 394,750
```

### Step 4a: HANA <= 0 branch -> 19,165 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (false alarm -- HANA was wrong) | **15,058** | 66.04 |
| 5 | **Sin stock (validado)** (confirmed OOS) = **Pop A** | **1,752** | 11.11 |
| 4 | **Sustituto** | **1,199** | 7.13 |
| 13 | **Compuesto (weight OOS)** | **468** | 7.94 |
| 11 | **Agregado** | **447** | 2.05 |
| 12 | **Faltante parcial** | **241** | 1.91 |

Key finding: **15,058 false alarms** -- HANA said stock was 0 but picker found it anyway.

**False alarm examples (HANA <= 0, but picker found it):**
- Microondas Thomas TH-20S01 20 Litros @ J410 — HANA=0, picker found it, 111,986 CLP
- Cepillo de Dientes Eléctrico Oral-B 1 Mango + Cabezal iO2 @ J410 — HANA=0, picker found it, 92,386 CLP
- Locos Cocidos en Conserva Alimex 1 kg @ J414 — HANA=0, picker found it, 89,980 CLP

### Step 4b: HANA > 0 branch -> 394,750 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (expected -- HANA was right) | **373,624** | 1699.8 |
| 5 | **Sin stock (validado)** (phantom inventory) = **Pop B** | **6,963** | 39.06 |
| 4 | **Sustituto** | **6,287** | 34.62 |
| 11 | **Agregado** | **3,993** | 20.43 |
| 13 | **Compuesto (weight OOS)** | **2,582** | 49.9 |
| 12 | **Faltante parcial** | **1,321** | 14.5 |

**Phantom inventory examples (HANA > 0, but picker OOS):**
- Café Grano Molido Señor K Manizales 250 g @ J408 — HANA=14, picker OOS, 140,220 CLP
- Secador Multistyler Mint 8 en 1 MHR-036 @ J512 — HANA=21, picker OOS, 89,990 CLP
- Cafetera Nescafé Dolce Gusto Genio S White @ J512 — HANA=4, picker OOS, 69,993 CLP

**Extreme phantoms (highest HANA qty, still OOS):**
- Jugo en Polvo Livean Papaya 7 g @ J512 — HANA=**5,469** units, picker OOS, 199 CLP
- Yogurt Loncoleche Proteína Natural Endulzado 140 g @ J411 — HANA=**4,569** units, picker OOS, 3,000 CLP

### Step 5: The two populations we analyze

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  1,789  (11.03M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) :  7,295  (40.20M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              :  9,084  (51.23M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **1,789** |
| Total CLP | **11,033,823** (11.03M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS (same item picked at/before this order - restock impossible) | 387 | 21.6% | 1,753,237 |
| 2 | RESTOCK LIKELY (same item picked only later + HANA rose after) | 81 | 4.5% | 445,449 |
| 3 | PICKED LATER (only later, HANA flat - probable picker miss) | 63 | 3.5% | 398,892 |
| 4 | CONFIRMED STOCKOUT (multiple orders, all failed) | 591 | 33.0% | 3,738,607 |
| 5 | SINGLE ATTEMPT (not delivered; cause unconfirmed) | 667 | 37.3% | 4,697,638 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 387 | 1,753,237 |
| **YES -- someone got it after** (Tier 2+3) | 144 | 844,341 |
| **NO -- nobody got it** (Tier 4+5) | 1,258 | 8,436,245 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 1,789 | 100.0% |
| Picker marked OOS | 1,789 | 100.0% |
| Item not delivered (pick=0) | 1,789 | 100.0% |
| Order NOT cancelled | 1,786 | 99.8% |
| Order delivered/invoiced | 1,736 | 97.0% |
| Real picking session ran | 1,499 | 83.8% |

### Sample rows -- Pop A

**Tier 1 (PICKER MISS -- strongest evidence):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_before |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233194224jmch-01 | J519 | 691432-KG | 11430 | Tártaro 3% Grasa kg | 33,780 | -13.442 | 3.0 |
| v233161434jmch-01 | J989 | 1462097-KG | 23 | Lomo Liso Al Vacío kg | 30,583 | 0.0 | 2.0 |

**Tier 2 (RESTOCK LIKELY -- picked only later + HANA rose after):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233177829jmch-01 | J403 | 1404539 | 9293 | Torta Florencia: Panqueque, Crema Chocolate y Naranja 15 Porciones | 29,990 | 0.0 | 1.0 |
| v233144352jmch-01 | J512 | 1692053 | 26202 | Torta Hoja, Crema Pastelera y Manjar 15 Porciones | 17,990 | 0.0 | 1.0 |

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233211877jmch-01 | J410 | 2055038 | 159202 | Pack Whisky Buchanan"s 12 Años 40° 750 cc + Copa | 113,358 | 0.0 |
| v233139698jmch-01 | J410 | 2055038 | 159202 | Pack Whisky Buchanan"s 12 Años 40° 750 cc + Copa | 56,679 | 0.0 |

**Tier 5 (SINGLE ATTEMPT -- not delivered, cause unconfirmed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233170327jmch-01 | J414 | 2037339 | 150220 | Pisco Bou Barroeta Marias Transparente 40° 750 cc | 59,980 | 0.0 |
| v233142126jmch-01 | J748 | 2005801 | 141064 | Prieta con Arroz 500 g | 47,920 | 0.0 |

---

## 6. Population B -- Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock != physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **7,295** |
| Total CLP | **40,198,521** (40.20M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS (same item picked at/before this order - restock impossible) | 2,532 | 34.7% | 12,646,951 |
| 2 | RESTOCK LIKELY (same item picked only later + HANA rose after) | 167 | 2.3% | 756,281 |
| 3 | PICKED LATER (only later, HANA flat - probable picker miss) | 413 | 5.7% | 1,966,842 |
| 4 | CONFIRMED STOCKOUT (multiple orders, all failed) | 1,563 | 21.4% | 8,749,959 |
| 5 | SINGLE ATTEMPT (not delivered; cause unconfirmed) | 2,620 | 35.9% | 16,078,488 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 2,532 | 12,646,951 |
| **YES -- someone got it after** (Tier 2+3) | 580 | 2,723,123 |
| **NO -- nobody got it** (Tier 4+5) | 4,183 | 24,828,447 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 7,295 | 100.0% |
| Picker marked OOS | 7,295 | 100.0% |
| Item not delivered (pick=0) | 7,295 | 100.0% |
| Order NOT cancelled | 7,273 | 99.7% |
| Order delivered/invoiced | 7,011 | 96.1% |
| Real picking session ran | 5,739 | 78.7% |

### Sample rows -- Pop B

**Tier 1 (PICKER MISS -- HANA showed stock, picker missed it):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_before |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233213623jmch-01 | J408 | 1935306 | 118071 | Café Grano Molido Señor K Manizales 250 g | 140,220 | 14.0 | 3.0 |
| v233191599jmch-01 | J780 | 1561251 | 11502 | Malaya de Cerdo Super Cerdo 900 g | 68,050 | 9.0 | 1.0 |

**Tier 3 (PICKED LATER -- only later, HANA flat):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233161361jmch-01 | J775 | 1997502-KG | 139011 | Punta de Ganso Americana Farmers Al Vacío kg | 64,974 | 37.32 | 1.0 |
| v233143320jmch-01 | J592 | 355161 | 432 | Bebida Sprite 1.5 L | 35,880 | 96.0 | 2.0 |

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233152733jmch-01 | J512 | 2019838-KG | 146344 | Asado de Tira Bandera EEUU-V Farmers kg | 47,980 | 14.5 |
| v233152647jmch-01 | J512 | 2019838-KG | 146344 | Asado de Tira Bandera EEUU-V Farmers kg | 47,980 | 14.5 |

**Tier 5 (SINGLE ATTEMPT -- not delivered, cause unconfirmed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233212097jmch-01 | J512 | 2064977 | 165499 | Secador Multistyler Mint 8 en 1 MHR-036 | 89,990 | 21.0 |
| v233150505jmch-01 | J512 | 2026369 | 153897 | Cafetera Nescafé Dolce Gusto Genio S White | 69,993 | 4.0 |

---

## 7. Combined Summary

| | Pop A (HANA <= 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 1,789 | 7,295 | **9,084** |
| **CLP** | 11,033,823 | 40,198,521 | **51,232,344** |
| YES at/before | 387 (21.6%) | 2,532 (34.7%) | 2,919 (32.1%) |
| YES after | 144 (8.0%) | 580 (8.0%) | 724 (8.0%) |
| NO nobody | 1,258 (70.3%) | 4,183 (57.3%) | 5,441 (59.9%) |

**Cancelled order examples:**
- Pop A: Torta Vegana Mix Tropical un. @ J989 (order v233157229jmch-01, cancelled by marco.chauran@jumbo.cl)
- Pop B: Pack 12 un. Cerveza Kross Golden Ale 5.3° 330 cc @ J414 (order v233208646jmch-01, cancelled by carolinatomic@yahoo.com)

### Multi-day comparison

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul 8 | Jul 9 | Jul 10 |
|--------|-------|-------|-------|-------|-------|-------|
| Order lines | 370,654 | 419,497 | 366,041 | 355,262 | 363,397 | 419,133 |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | 2.4% | 2.5% | 2.1% |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | 49.1% | 47.0% | 48.5% |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | 100.0M | 90.3M | 99.0M | 109.0M |
| Pop A lines | 1,482 | 2,608 | 1,978 | 1,402 | 1,488 | 1,789 |
| Pop A CLP | 8.53M | 13.88M | 11.57M | 8.28M | 9.30M | 11.03M |
| Pop B lines | 6,911 | 10,613 | 8,510 | 7,235 | 6,527 | 7,295 |
| Pop B CLP | 37.59M | 55.56M | 46.83M | 39.63M | 37.21M | 40.20M |
| Combined lines | 8,393 | 13,221 | 10,488 | 8,637 | 8,015 | 9,084 |
| Combined CLP | 46.13M | 69.44M | 58.40M | 47.91M | 46.51M | 51.23M |
| False alarms | 10,903 | 13,980 | 0 | 12,246 | 0 | 15,058 |
| HANA batches | 24 | 17 | 24 | 24 | 24 | 24 |

---

## 8. Methodology: How We Got Here

### Step 1: Export orders from Redshift
- Script: `scripts/export_orders_0710.py`
- Query: `jumbo_bo.io_items` JOIN `jumbo_bo.vw_master_pickers_data` WHERE fecha_creacion on Jul 10
- Output: `all_orders_0710_FULL.csv` (419,133 lines)

### Step 2: Download HANA parquets from S3
- 24 hourly batches from `s3://cencosud-jumbo-raw/vw_daily_nrt/`
- Each batch: 36 parquet files, ~462MB total
- Stored: `data_samples/vw_daily_nrt_0710_HHMM/`

### Step 3: Cross-reference orders × HANA (delay_proof)
- Script: `scripts/delay_proof_0710.py`
- For each order line, find most recent HANA batch BEFORE `fecha_creacion`
- If `Ending_On_Hand_Qty <= 0` → mark as preventable
- Output: `delay_proof_0710_preventable.parquet` (19,165 events)

### Step 4: Build evidence pack (Pop A/B + confirmation tiers)
- Script: `scripts/evidence_pack_0710.py`
- Pop A: HANA <= 0 AND status IN (3,5,7,14) → propagation gap
- Pop B: HANA > 0 AND status IN (3,5,7,14) → phantom inventory
- Enrichment: got_before/got_after, hana_rose_after, io_internal_orders, io_picking
- Confirmation tiers 1-5 assigned based on cross-order corroboration
- Output: `evidence_pop_a_jul10.csv` (1,789), `evidence_pop_b_jul10.csv` (7,295)

---

## 9. Scripts Reference

| Script | What it does | Data sources | Redshift needed? |
|--------|-------------|--------------|------------------|
| `evidence_pack_0710.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0710.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0710.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
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

Always apply these 6 VARCHAR overrides when reading any `all_orders_07XX_FULL.csv` via DuckDB.

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
| `analysis/jul10/evidence_pack_jul10.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul10/evidence_pop_a_jul10.csv` | 1,789 | Pop A full detail |
| `analysis/jul10/evidence_pop_b_jul10.csv` | 7,295 | Pop B full detail |
| `analysis/jul10/delay_proof_0710_preventable.parquet` | 19,165 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul10/delay_proof_0710_raw.xlsx` | 19,165 | Same, Excel format with summaries |

### HANA batch coverage note

Jul 10 has **full 24/24 batch coverage** (00:40 through 23:41), making it a clean day for analysis.

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 19,165 |
| Distinct orders | 11,810 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 61 |

The mean gap of **30 minutes** represents the average time between the most recent HANA batch (showing stock <= 0) and when the customer placed their order.
