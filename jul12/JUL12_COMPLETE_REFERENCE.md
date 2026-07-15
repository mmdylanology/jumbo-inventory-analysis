# July 12, 2026 — Complete OOS Analysis Reference

> Generated 2026-07-15. Same methodology as JUL8_COMPLETE_REFERENCE.md.

---

## 1. Data Sources

| Source | Path | Rows | Notes |
|--------|------|------|-------|
| **HANA parquets** | `data_samples/vw_daily_nrt_0712_*/*.parquet` | 24 batches | 00:40-23:41 CLT, J-stores only |
| **Orders CSV** | `exports/cencommerce_quiebre/all_orders_0712_FULL.csv` | 394,647 lines | Exported from Redshift `io_items` |
| **Redshift live** | `jumbo_bo.io_internal_orders` + `jumbo_bo.io_picking` | ~22K orders | Only 2 queries needed by evidence_pack_0712.py |
| **Evidence CSVs** | `analysis/jul12/evidence_pop_a_jul12.csv`, `evidence_pop_b_jul12.csv` | 2,253 + 8,665 | Output of evidence_pack_0712.py |
| **Evidence Excel** | `analysis/jul12/evidence_pack_jul12.xlsx` | 3 sheets | Summary + Pop A + Pop B |
| **Delay proof** | `analysis/jul12/delay_proof_0712_preventable.parquet` | 17,825 | All HANA<=0 order lines |
| **Delay proof Excel** | `analysis/jul12/delay_proof_0712_raw.xlsx` | 17,825 | Same, Excel format with summaries |

### HANA batch times (24 batches, CLT)

```
00:40  01:40  02:40  03:40  04:40  05:41  06:40  07:40  08:40  09:40  10:41  11:41
12:40  13:41  14:40  15:40  16:41  17:40  18:41  19:42  20:40  21:40  22:41  23:40
```

Full 24/24 coverage. First batch at 00:40 means all orders have a HANA snapshot.

HANA stats: 541,535,050 rows loaded, 503,180,601 zero-or-negative (92.9%).

---

## 2. Status Codes (from `jumbo_bo.io_items_status`)

### Statuses active on Jul 12 (Redshift, J-stores)

| Status | Code | Lines | % | CLP (M) | Example |
|--------|------|-------|---|---------|---------|
| **1** | Pickeado | **365,887** | 92.7% | 1569.61 | Pañales Adulto Pants Cotidian Ultra Talla G 30 un. @ J660 (159,400 CLP) |
| **5** | Sin stock (validado) | **11,023** | 2.8% | 59.59 | Aceite de Oliva Chef Extra Virgen 1 L @ J660 (106,190 CLP) |
| **4** | Sustituto | **8,904** | 2.3% | 47.11 | Filete Premium A Punto Al Vacío kg @ J414 (201,542 CLP) |
| **11** | Agregado | **4,585** | 1.2% | 22.2 | Filete cat V nacional kg @ J414 (64,265 CLP) |
| **13** | Compuesto (weight OOS) | **2,637** | 0.7% | 37.85 | Filete Premium A Punto Al Vacío kg @ J414 (403,085 CLP) |
| **12** | Faltante parcial | **1,597** | 0.4% | 14.85 | Juego de Mesa ¡Yo Lo Vi! Dactic Games @ J411 (149,900 CLP) |
| **3** | Sin stock | **7** | 0.0% | 0.04 | Pechuga de Pollo Entera kg @ J955 (21,425 CLP) |
| **0** | Sin pickear | **5** | 0.0% | 0.0 | Palta Hass Extra Chilena (2 un. Aprox) @ J520 (86 CLP) |
| **14** | Sin Stock (sala) | **2** | 0.0% | 0.0 | Saborizante Polvo Cola Cao Chocolate 400 g @ J414 (3,790 CLP) |
| | **TOTAL** | **394,647** | 100% | **1751.26** | |

Jul 12 (Sunday):
- **OOS rate 2.8%** (status 5: 11,023 lines)
- **Affected orders 57.2%** of 22,146
- Our filter `status IN (3,5,7,14)` captures relevant OOS lines

---

## 3. Order-Level Overview

### Total orders (source: local CSV `all_orders_0712_FULL.csv`)

| Metric | Count |
|--------|-------|
| **Total item lines** | **394,647** |
| **Distinct orders** | **22,146** |

### Quantity

| Metric | Value |
|--------|-------|
| Qty ordered (SUM originalquantity) | 703,681 |
| Qty delivered (SUM pickingquantity) | 669,086 (95.1%) |

### Value (CLP)

| Metric | CLP | % |
|--------|-----|---|
| **Total ordered** | **1,751,259,693** | 100% |
| Lost to OOS (status 5 + 13) | 97,443,509 | 5.6% |
| Lost to substitution (status 4) | 47,112,856 | 2.7% |

### Order impact summary

| Category | Orders | % of 22,146 |
|----------|--------|-------------|
| **Clean** (no OOS, no substitution) | **9,474** | **42.8%** |
| Had any OOS (status 5 or 13) | 8,540 | 38.6% |
| Had substitution (status 4) | 6,387 | 28.8% |
| Had OOS AND substitution | 2,742 | 12.4% |
| Had partial pick (status 12) | 1,456 | 6.6% |

**57.2% of all processed orders had at least one item affected by OOS or substitution.**
**97.4M CLP lost to OOS alone.**

### Connection: orders -> item lines -> Pop A/B

```
22,146 orders in CSV (394,647 item lines)
|- 22,146 with processed lines
|  |- 9,474 clean (no OOS, no sub)
|  |- 12,672 had OOS or sub or both
|     |-  11,032 status 3/5/14 OOS lines -> Pop A (2,253) + Pop B (8,665)  <- Sections 5-6
|     |-  2,637 status-13 weight OOS lines -> NOT in Pop A/B
|     |-  8,904 status-4 substitution lines
```

---

## 4. Full Analysis Flow (step-by-step drill-down)

### Step 1: Start with all order lines

**394,647 order lines** from `all_orders_0712_FULL.csv`

### Step 2: Look up HANA stock at most recent batch before each order

```
394,647 total order lines
|- Has HANA snapshot before order : 389,682
|- No HANA snapshot               :   4,965  (store/item not in HANA)
```

### Step 3: Split by HANA stock level

```
389,682 with HANA snapshot
|- HANA <= 0 (ERP says out of stock) :  17,825
|- HANA > 0 (ERP says in stock)      : 371,857
```

### Step 4a: HANA <= 0 branch -> 17,825 order lines

HANA said this item was out of stock BEFORE the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (false alarm -- HANA was wrong) | **13,174** | 54.75 |
| 5 | **Sin stock (validado)** (confirmed OOS) = **Pop A** | **2,252** | 13.43 |
| 4 | **Sustituto** | **1,356** | 8.8 |
| 13 | **Compuesto (weight OOS)** | **429** | 5.59 |
| 11 | **Agregado** | **382** | 1.58 |
| 12 | **Faltante parcial** | **231** | 1.65 |
| 14 | **Sin Stock (sala)** | **1** | 0.0 |

Key finding: **13,174 false alarms** -- HANA said stock was 0 but picker found it anyway.

**False alarm examples (HANA <= 0, but picker found it):**
- Estufa Infrarroja Thomas TH-IR60 Madera @ J403 — HANA=0, picker found it, 125,990 CLP
- Microondas digital Ursus Trotter UT-KROMM - 25 litros @ J403 — HANA=0, picker found it, 109,990 CLP
- Alimento Perro Adulto Champion Dog Salmón Pollo 15 kg @ J410 — HANA=0, picker found it, 95,960 CLP

### Step 4b: HANA > 0 branch -> 371,857 order lines

HANA said this item was IN stock before the customer ordered. What happened at picking?

| Status | Meaning | Lines | CLP (M) |
|--------|---------|-------|---------|
| 1 | **Pickeado** (expected -- HANA was right) | **348,092** | 1494.86 |
| 5 | **Sin stock (validado)** (phantom inventory) = **Pop B** | **8,657** | 45.44 |
| 4 | **Sustituto** | **7,439** | 37.67 |
| 11 | **Agregado** | **4,155** | 20.33 |
| 13 | **Compuesto (weight OOS)** | **2,163** | 31.71 |
| 12 | **Faltante parcial** | **1,348** | 12.96 |
| 3 | **Sin stock** | **7** | 0.04 |
| 0 | **Sin pickear** | **5** | 0.0 |
| 14 | **Sin Stock (sala)** | **1** | 0.0 |

**Phantom inventory examples (HANA > 0, but picker OOS):**
- Aceite de Oliva Chef Extra Virgen 1 L @ J660 — HANA=52, picker OOS, 106,190 CLP
- Secador Multistyler Mint 8 en 1 MHR-036 @ J534 — HANA=7, picker OOS, 89,990 CLP
- Posta Rosada Premium Natural Al Vacío kg @ J532 — HANA=21, picker OOS, 77,948 CLP

**Extreme phantoms (highest HANA qty, still OOS):**
- Atún Lomitos en Agua 91 g drenado, 140 g neto @ J504 — HANA=**7,704** units, picker OOS, 2,991 CLP
- Bebida Coca-Cola Light 1.5 L @ J414 — HANA=**5,322** units, picker OOS, 8,970 CLP

### Step 5: The two populations we analyze

```
Pop A = HANA <= 0 AND status IN (3,5,7,14) :  2,253  (13.43M CLP) -- propagation gap
Pop B = HANA >  0 AND status IN (3,5,7,14) :  8,665  (45.48M CLP) -- phantom inventory
----------------------------------------------------------------------
TOTAL analyzed                              :  10,918  (58.91M CLP)
```

---

## 5. Population A -- Propagation Gap (HANA <= 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed <= 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX -> customer ordered anyway.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **2,253** |
| Total CLP | **13,434,324** (13.43M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS (same item picked at/before this order - restock impossible) | 558 | 24.8% | 2,821,107 |
| 2 | RESTOCK LIKELY (same item picked only later + HANA rose after) | 58 | 2.6% | 351,516 |
| 3 | PICKED LATER (only later, HANA flat - probable picker miss) | 188 | 8.3% | 1,028,523 |
| 4 | CONFIRMED STOCKOUT (multiple orders, all failed) | 694 | 30.8% | 3,942,036 |
| 5 | SINGLE ATTEMPT (not delivered; cause unconfirmed) | 755 | 33.5% | 5,291,142 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 558 | 2,821,107 |
| **YES -- someone got it after** (Tier 2+3) | 246 | 1,380,039 |
| **NO -- nobody got it** (Tier 4+5) | 1,449 | 9,233,178 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 2,253 | 100.0% |
| Picker marked OOS | 2,253 | 100.0% |
| Item not delivered (pick=0) | 2,253 | 100.0% |
| Order NOT cancelled | 2,241 | 99.5% |
| Order delivered/invoiced | 2,145 | 95.2% |
| Real picking session ran | 1,787 | 79.3% |

### Sample rows -- Pop A

**Tier 1 (PICKER MISS -- strongest evidence):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_before |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233259729jmch-01 | J408 | 1462094-KG | 11475 | Filete Al Vacío kg | 39,879 | -24.46 | 11.0 |
| v233243967jmch-01 | J504 | 1462094-KG | 11475 | Filete Al Vacío kg | 39,879 | 0.0 | 1.0 |

**Tier 2 (RESTOCK LIKELY -- picked only later + HANA rose after):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233250966jmch-01 | J403 | 1964212 | 132539 | Empanada Mediana Queso 450 g 6 un. | 28,518 | 0.0 | 1.0 |
| v233233497jmch-01 | J403 | 2057880 | 162443 | Pollo Tuto Deshuesado Congelado 750 g | 27,450 | 0.0 | 1.0 |

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233223527jmch-01 | J410 | 2055038 | 159202 | Pack Whisky Buchanan"s 12 Años 40° 750 cc + Copa | 75,572 | 0.0 |
| v233281496jmch-01 | J414 | 991938-KG | 17845 | Asiento Premium Natural Al Vacío kg | 51,974 | 0.0 |

**Tier 5 (SINGLE ATTEMPT -- not delivered, cause unconfirmed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233274827jmch-01 | J403 | 2026369 | 153897 | Cafetera Nescafé Dolce Gusto Genio S White | 69,993 | 0.0 |
| v233220854jmch-01 | J843 | 1462107-KG | 11485 | Punta de Ganso Al Vacío kg | 62,958 | 0.0 |

---

## 6. Population B -- Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock != physical shelf stock. ERP says units exist, but shelf is empty.

### Numbers

| Metric | Value |
|--------|-------|
| Total lines | **8,665** |
| Total CLP | **45,480,309** (45.48M) |

### Confirmation tiers

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS (same item picked at/before this order - restock impossible) | 3,090 | 35.7% | 14,166,806 |
| 2 | RESTOCK LIKELY (same item picked only later + HANA rose after) | 61 | 0.7% | 281,916 |
| 3 | PICKED LATER (only later, HANA flat - probable picker miss) | 843 | 9.7% | 4,024,943 |
| 4 | CONFIRMED STOCKOUT (multiple orders, all failed) | 1,788 | 20.6% | 9,305,597 |
| 5 | SINGLE ATTEMPT (not delivered; cause unconfirmed) | 2,883 | 33.3% | 17,701,047 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| **YES -- someone got it at/before** (Tier 1) | 3,090 | 14,166,806 |
| **YES -- someone got it after** (Tier 2+3) | 904 | 4,306,859 |
| **NO -- nobody got it** (Tier 4+5) | 4,671 | 27,006,644 |

### Evidence flags

| Flag | Count | % |
|------|-------|---|
| HANA snapshot before order | 8,665 | 100.0% |
| Picker marked OOS | 8,665 | 100.0% |
| Item not delivered (pick=0) | 8,665 | 100.0% |
| Order NOT cancelled | 8,635 | 99.7% |
| Order delivered/invoiced | 8,283 | 95.6% |
| Real picking session ran | 6,146 | 70.9% |

### Sample rows -- Pop B

**Tier 1 (PICKER MISS -- HANA showed stock, picker missed it):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_before |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233287287jmch-01 | J411 | 1843154 | 87636 | Pack 12 un. Yogurt Batido Nestlé Berries 115 g | 58,800 | 16.0 | 2.0 |
| v233260159jmch-01 | J403 | 451845 | 1451 | Aceite de Oliva Kardámili Clásico Extra Virgen 1 L | 54,360 | 4.0 | 4.0 |

**Tier 3 (PICKED LATER -- only later, HANA flat):**

| order_id | store | refid | sku | name | CLP | HANA qty | got_after |
|----------|-------|-------|-----|------|-----|----------|------------|
| v233244104jmch-01 | J659 | 942892 | 8115 | Papel Higiénico Confort Max Una Hoja 100 m 8 un. | 31,770 | 40.0 | 1.0 |
| v233220265jmch-01 | J775 | 2072119-KG | 166623 | Pechuga Deshuesada de Pollo kg (6 a 7 un. Aprox) | 28,450 | 100.0 | 1.0 |

**Tier 4 (CONFIRMED STOCKOUT -- multiple orders, all failed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233229043jmch-01 | J660 | 1247547 | 1437 | Aceite de Oliva Chef Extra Virgen 1 L | 106,190 | 52.0 |
| v233271347jmch-01 | J660 | 1462096-KG | 11476 | Huachalomo Al Vacío kg | 50,344 | 185.03 |

**Tier 5 (SINGLE ATTEMPT -- not delivered, cause unconfirmed):**

| order_id | store | refid | sku | name | CLP | HANA qty |
|----------|-------|-------|-----|------|-----|----------|
| v233264317jmch-01 | J534 | 2064977 | 165499 | Secador Multistyler Mint 8 en 1 MHR-036 | 89,990 | 7.0 |
| v233219724jmch-01 | J532 | 993602-KG | 17853 | Posta Rosada Premium Natural Al Vacío kg | 77,948 | 20.63 |

---

## 7. Combined Summary

| | Pop A (HANA <= 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 2,253 | 8,665 | **10,918** |
| **CLP** | 13,434,324 | 45,480,309 | **58,914,633** |
| YES at/before | 558 (24.8%) | 3,090 (35.7%) | 3,648 (33.4%) |
| YES after | 246 (10.9%) | 904 (10.4%) | 1,150 (10.5%) |
| NO nobody | 1,449 (64.3%) | 4,671 (53.9%) | 6,120 (56.1%) |

**Cancelled order examples:**
- Pop A: Espumante JP Chenet 24 Carat Gold 750 cc @ J408 (order v233257960jmch-01, cancelled by oscar.silva.a@gmail.com)
- Pop B: Pellet certificado 15 kg @ J403 (order v233280975jmch-01, cancelled by mariapaz.iglesias@jumbo.cl)

### Multi-day comparison

| Metric | Jul 1 | Jul 6 | Jul 7 | Jul 8 | Jul 9 | Jul 12 |
|--------|-------|-------|-------|-------|-------|-------|
| Order lines | 370,654 | 419,497 | 366,041 | 355,262 | 363,397 | 394,647 |
| OOS rate (st5) | 2.5% | 3.3% | 2.9% | 2.4% | 2.5% | 2.8% |
| Orders affected by OOS/sub | 51.9% | 59.6% | 53.9% | 49.1% | 47.0% | 57.2% |
| CLP lost to OOS (st5+13) | 89.0M | 118.6M | 100.0M | 90.3M | 99.0M | 97.4M |
| Pop A lines | 1,482 | 2,608 | 1,978 | 1,402 | 1,488 | 2,253 |
| Pop A CLP | 8.53M | 13.88M | 11.57M | 8.28M | 9.30M | 13.43M |
| Pop B lines | 6,911 | 10,613 | 8,510 | 7,235 | 6,527 | 8,665 |
| Pop B CLP | 37.59M | 55.56M | 46.83M | 39.63M | 37.21M | 45.48M |
| Combined lines | 8,393 | 13,221 | 10,488 | 8,637 | 8,015 | 10,918 |
| Combined CLP | 46.13M | 69.44M | 58.40M | 47.91M | 46.51M | 58.91M |
| False alarms | 10,903 | 13,980 | 0 | 12,246 | 0 | 13,174 |
| HANA batches | 24 | 17 | 24 | 24 | 24 | 24 |

---

## 8. Methodology: How We Got Here

### Step 1: Export orders from Redshift
- Script: `scripts/export_orders_0712.py`
- Query: `jumbo_bo.io_items` JOIN `jumbo_bo.vw_master_pickers_data` WHERE fecha_creacion on Jul 12
- Output: `all_orders_0712_FULL.csv` (394,647 lines)

### Step 2: Download HANA parquets from S3
- 24 hourly batches from `s3://cencosud-jumbo-raw/vw_daily_nrt/`
- Each batch: 36 parquet files, ~462MB total
- Stored: `data_samples/vw_daily_nrt_0712_HHMM/`

### Step 3: Cross-reference orders × HANA (delay_proof)
- Script: `scripts/delay_proof_0712.py`
- For each order line, find most recent HANA batch BEFORE `fecha_creacion`
- If `Ending_On_Hand_Qty <= 0` → mark as preventable
- Output: `delay_proof_0712_preventable.parquet` (17,825 events)

### Step 4: Build evidence pack (Pop A/B + confirmation tiers)
- Script: `scripts/evidence_pack_0712.py`
- Pop A: HANA <= 0 AND status IN (3,5,7,14) → propagation gap
- Pop B: HANA > 0 AND status IN (3,5,7,14) → phantom inventory
- Enrichment: got_before/got_after, hana_rose_after, io_internal_orders, io_picking
- Confirmation tiers 1-5 assigned based on cross-order corroboration
- Output: `evidence_pop_a_jul12.csv` (2,253), `evidence_pop_b_jul12.csv` (8,665)

---

## 9. Scripts Reference

| Script | What it does | Data sources | Redshift needed? |
|--------|-------------|--------------|------------------|
| `evidence_pack_0712.py` | Pop A/B + confirmation tiers + evidence flags | HANA parquets + orders CSV + 2 RS queries | YES |
| `delay_proof_0712.py` | HANA lookup for all orders -> preventable parquet | HANA parquets + orders CSV | NO |
| `export_orders_0712.py` | Export all order lines from Redshift | Redshift io_items + vw_master_pickers_data | YES |
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
| `analysis/jul12/evidence_pack_jul12.xlsx` | 3 sheets | Summary + Pop A + Pop B with all evidence columns |
| `analysis/jul12/evidence_pop_a_jul12.csv` | 2,253 | Pop A full detail |
| `analysis/jul12/evidence_pop_b_jul12.csv` | 8,665 | Pop B full detail |
| `analysis/jul12/delay_proof_0712_preventable.parquet` | 17,825 | All HANA<=0 order lines (checkpoint) |
| `analysis/jul12/delay_proof_0712_raw.xlsx` | 17,825 | Same, Excel format with summaries |

### HANA batch coverage note

Jul 12 has **full 24/24 batch coverage** (00:40 through 23:41), making it a clean day for analysis.

---

## 12. Delay Proof Stats

| Metric | Value |
|--------|-------|
| Total preventable events (HANA <= 0 before order) | 17,825 |
| Distinct orders | 10,703 |
| Gap minutes (min) | 0 |
| Gap minutes (mean) | 30 |
| Gap minutes (max) | 61 |

The mean gap of **30 minutes** represents the average time between the most recent HANA batch (showing stock <= 0) and when the customer placed their order.
