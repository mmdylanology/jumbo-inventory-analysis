# July 11, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-15. Same methodology as JUL8_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0711_*/*.parquet` | 24 batches | 00:40-23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0711_FULL.csv` | 412,697 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~24K orders | Only 2 queries needed by evidence_pack_0711.py |
| **Evidence CSVs** | `analysis/jul11/evidence_pop_a_jul11.csv`, `evidence_pop_b_jul11.csv` | 1,789 + 7,295 | Output of evidence_pack_0711.py |
| **Evidence Excel** | `analysis/jul11/evidence_pack_jul11.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul11/delay_proof_0711_preventable.parquet` | 17,735 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul11/delay_proof_0711_raw.xlsx` | 17,735 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:40  01:40  02:41  03:40  04:42  05:42  06:41  07:40  08:40  09:40  10:41  11:40
12:41  13:41  14:41  15:40  16:41  17:40  18:41  19:40  20:40  21:40  22:41  23:41
```

Full 24/24 coverage. First batch at 00:40 means all orders have a HANA snapshot.

HANA stats: 541,534,462 rows loaded, 503,112,396 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 11 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) | Example |
|--------|------|-------|---|---------|---------|
| **1** | Pickeado | **385,603** | 93.4% | 1701.82 | Whisky Johnnie Walker 18 Años 750 cc @ J408 (230,500 CLP) |
| **5** | Sin stock (validado) | **9,148** | 2.2% | 51.36 | Café Grano Molido Señor K Manizales 250 g @ J408 (140,220 CLP) |
| **4** | Sustituto | **8,048** | 2.0% | 46.0 | Alimento Perro Adulto Purina One Medianos y Grandes Carne 12 kg @ J403 (135,960 CLP) |
| **11** | Agregado | **5,253** | 1.3% | 26.13 | Congrio Dorado Filete Fresco Granel @ J619 (62,046 CLP) |
| **13** | Compuesto (weight OOS) | **3,004** | 0.7% | 52.4 | Filete Al Vacío kg @ J414 (418,730 CLP) |
| **12** | Faltante parcial | **1,618** | 0.4% | 15.27 | Shampoo Herbal Essences Bio Renew Argan Oil 865 ml @ J513 (125,082 CLP) |
| **3** | Sin stock | **19** | 0.0% | 0.23 | Trutro Largo de Pollo Sin Marinar 1 kg @ J506 (27,560 CLP) |
| **14** | Sin Stock (sala) | **3** | 0.0% | 0.02 | Caja Galleta Costa Mini Chips Chocolate 35 g @ J410 (7,500 CLP) |
| **15** | Compuesto (v2) | **1** | 0.0% | 0.0 | Pera Abate Fetel Granel @ J592 (548 CLP) |
| | **TOTAL** | **412,697** | 100% | **1893.23** | |

Jul 11 (Saturday):
- **OOS rate 2.2%** (status 5: 9,148 lines)
- **Affected orders 51.2%** of 24,934
- Our filter `status IN (3,5,7,14)` captures relevant OOS lines

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0711_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **412,697** |
| **Distinct orders** | **24,934** |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 718,401 |
| Qty delivered (SUM pickingquantity) | 688,324 (95.8%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,893,231,325** | 100% |
| Lost to OOS (status 5 + 13) | 103,762,738 | 5.5% |
| Lost to substitution (status 4) | 46,001,575 | 2.4% |

### Order impact summary

| Category | Orders | % of 24,934 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **12,161** | **48.8%** |
| Had any OOS (status 5 or 13) | 8,256 | 33.1% |
| Had substitution (status 4) | 6,171 | 24.7% |
| Had OOS AND substitution | 2,311 | 9.3% |
| Had partial pick (status 12) | 1,479 | 5.9% |

**51.2% of all processed orders had at least one item affected by OOS or substitution.**
**103.8M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
24,934 orders in CSV (412,697 item lines)
|- 24,934 with processed lines
|  |- 12,161 clean (no OOS, no sub)
|  |- 12,773 had OOS or sub or both
|     |-  9,170 status 3/5/14 OOS lines -> Pop A (1,789) + Pop B (7,295)  <- Sections 5-6
|     |-  3,004 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  8,048 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**412,697 order lines** from `all_orders_0711_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

```
412,697 total order lines
|- Has HANA snapshot before order : 408,276
|- No HANA snapshot               :   4,421  (store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
408,276 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  17,735
|- HANA > 0 (ERP says in stock)      : 390,541
```

### Step 4a: HANA <= 0 branch -> 17,735 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (false alarm -- HANA was wrong) | **13,666** | 58.12 |
| 5 | **Sin stock (validado)** (confirmed OOS) = **Pop A** | **1,788** | 11.01 |
| 4 | **Sustituto** | **1,161** | 7.52 |
| 13 | **Compuesto (weight OOS)** | **442** | 6.97 |
| 11 | **Agregado** | **433** | 2.11 |
| 12 | **Faltante parcial** | **244** | 1.95 |
| 3 | **Sin stock** | **1** | 0.03 |

Key finding: **13,666 false alarms** -- HANA said stock was 0 but picker found it anyway.

**False alarm examples (HANA <= 0, but picker found it):**
- Alimento Perro Adulto Purina One Medianos y Grandes Carne 12 kg @ J403 — HANA=0, picker found it, 135,960 CLP
- Alimento Perro Adulto Purina One Medianos y Grandes Carne 12 kg @ J403 — HANA=0, picker found it, 135,960 CLP
- Secador Multistyler Mint 8 en 1 MHR-036 @ J403 — HANA=0, picker found it, 89,990 CLP

### Step 4b: HANA > 0 branch -> 390,541 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (expected -- HANA was right) | **367,758** | 1625.07 |
| 5 | **Sin stock (validado)** (phantom inventory) = **Pop B** | **7,272** | 39.93 |
| 4 | **Sustituto** | **6,826** | 38.15 |
| 11 | **Agregado** | **4,771** | 23.78 |
| 13 | **Compuesto (weight OOS)** | **2,534** | 45.03 |
| 12 | **Faltante parcial** | **1,365** | 13.29 |
| 3 | **Sin stock** | **18** | 0.21 |
| 14 | **Sin Stock (sala)** | **3** | 0.02 |
| 15 | **Compuesto (v2)** | **1** | 0.0 |

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
| v233194224jmch-01 | J519 | 691432-KG | 11430 | Tártaro 3% Grasa kg | 33,780 | -13.442 | 3 |
| v233161434jmch-01 | J989 | 1462097-KG | 23 | Lomo Liso Al Vacío kg | 30,583 | 0.0 | 2 |

**Tier 2 (RESTOCK LIKELY -- picked only later + HANA rose after):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233177829jmch-01 | J403 | 1404539 | 9293 | Torta Florencia: Panqueque, Crema Chocolate y Naranja 15 Porciones | 29,990 | 0.0 | 1 |
| v233144352jmch-01 | J512 | 1692053 | 26202 | Torta Hoja, Crema Pastelera y Manjar 15 Porciones | 17,990 | 0.0 | 1 |

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
| v233213623jmch-01 | J408 | 1935306 | 118071 | Café Grano Molido Señor K Manizales 250 g | 140,220 | 14.0 | 3 |
| v233191599jmch-01 | J780 | 1561251 | 11502 | Malaya de Cerdo Super Cerdo 900 g | 68,050 | 9.0 | 1 |

**Tier 3 (PICKED LATER -- only later, HANA flat):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233161361jmch-01 | J775 | 1997502-KG | 139011 | Punta de Ganso Americana Farmers Al Vacío kg | 64,974 | 37.32 | 1 |
| v233143320jmch-01 | J592 | 355161 | 432 | Bebida Sprite 1.5 L | 35,880 | 96.0 | 2 |

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

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul 8 | Jul 9 | Jul 11 |
|--------|-------|-------|-------|-------|-------|-------|
| Order lines | 370,654 | 419,497 | 366,041 | 355,262 | 363,397 | 412,697 |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | 2.4% | 2.5% | 2.2% |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | 49.1% | 47.0% | 51.2% |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | 100.0M | 90.3M | 99.0M | 103.8M |
| Pop A lines | 1,482 | 2,608 | 1,978 | 1,402 | 1,488 | 1,789 |
| Pop A CLP | 8.53M | 13.88M | 11.57M | 8.28M | 9.30M | 11.03M |
| Pop B lines | 6,911 | 10,613 | 8,510 | 7,235 | 6,527 | 7,295 |
| Pop B CLP | 37.59M | 55.56M | 46.83M | 39.63M | 37.21M | 40.20M |
| Combined lines | 8,393 | 13,221 | 10,488 | 8,637 | 8,015 | 9,084 |
| Combined CLP | 46.13M | 69.44M | 58.40M | 47.91M | 46.51M | 51.23M |
| False alarms | 10,903 | 13,980 | 0 | 12,246 | 0 | 13,666 |
| HANA batches | 24 | 17 | 24 | 24 | 24 | 24 |

---

## 8. Methodology: How We Got Here

### Step 1: Export orders from Redshift
- Script: `scripts/export_orders_0711.py`
- Query: `jumbo_bo.io_items` JOIN `jumbo_bo.vw_master_pickers_data` WHERE fecha_creacion on Jul 11
- Output: `all_orders_0711_FULL.csv` (412,697 lines)

### Step 2: Download HANA parquets from S3
- 24 hourly batches from `s3://cencosud-jumbo-raw/vw_daily_nrt/`
- Each batch: 36 parquet files, ~462MB total
- Stored: `data_samples/vw_daily_nrt_0711_HHMM/`

### Step 3: Cross-reference orders × HANA (delay_proof)
- Script: `scripts/delay_proof_0711.py`
- For each order line, find most recent HANA batch BEFORE `fecha_creacion`
- If `Ending_On_Hand_Qty <= 0` → mark as preventable
- Output: `delay_proof_0711_preventable.parquet` (17,735 events)

### Step 4: Build evidence pack (Pop A/B + confirmation tiers)
- Script: `scripts/evidence_pack_0711.py`
- Pop A: HANA <= 0 AND status IN (3,5,7,14) → propagation gap
- Pop B: HANA > 0 AND status IN (3,5,7,14) → phantom inventory
- Enrichment: got_before/got_after, hana_rose_after, io_internal_orders, io_picking
- Confirmation tiers 1-5 assigned based on cross-order corroboration
- Output: `evidence_pop_a_jul11.csv` (1,789), `evidence_pop_b_jul11.csv` (7,295)

---

## 9. Scripts Reference

| Script | What it does | Data sources | Redshift needed? |
|--------|-------------|--------------|------------------|
| `evidence_pack_0711.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0711.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0711.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
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
| `analysis/jul11/evidence_pack_jul11.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul11/evidence_pop_a_jul11.csv` | 1,789 | Pop A full detail |
| `analysis/jul11/evidence_pop_b_jul11.csv` | 7,295 | Pop B full detail |
| `analysis/jul11/delay_proof_0711_preventable.parquet` | 17,735 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul11/delay_proof_0711_raw.xlsx` | 17,735 | Same, Excel format with summaries |

### HANA batch coverage note

Jul 11 has **full 24/24 batch coverage** (00:40 through 23:41), making it a clean day for analysis.

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 17,735 |
| Distinct orders | 11,168 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 61 |

The mean gap of **30 minutes** represents the average time between the most recent HANA batch (showing stock <= 0) and when the customer placed their order.
