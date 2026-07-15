# MASTER REFERENCE — Jul 7-12, 2026 (6-Day OOS Analysis)

> Generated 2026-07-15. Combines daily analyses from Jul 7 through Jul 12.
> Data sources: HANA (24/24 batches per day), Redshift orders, VTEX SQS events.

---

## 1. Data Sources

| Day | Orders CSV | Lines | HANA Batches | VTEX Events | Evidence A | Evidence B |
|-----|-----------|-------|-------------|-------------|------------|------------|
| **Jul 7** (Tuesday) | `all_orders_0707_FULL.csv` | 371,553 | 24/24 | 2,400,000 | 1,978 | 8,510 |
| **Jul 8** (Wednesday) | `all_orders_0708_FULL.csv` | 355,262 | 24/24 | 2,300,000 | 1,402 | 7,235 |
| **Jul 9** (Thursday) | `all_orders_0709_FULL.csv` | 367,299 | 24/24 | 618,000 | 1,488 | 6,527 |
| **Jul 10** (Friday) | `all_orders_0710_FULL.csv` | 419,133 | 24/24 | 905,164 | 1,789 | 7,295 |
| **Jul 11** (Saturday) | `all_orders_0711_FULL.csv` | 412,697 | 24/24 | 545,332 | 1,789 | 7,295 |
| **Jul 12** (Sunday) | `all_orders_0712_FULL.csv` | 394,647 | 24/24 | 652,783 | 2,253 | 8,665 |

**6-day totals:** 2,320,591 order lines across 139,764 orders. 144 HANA batches (24×6). ~7.9M VTEX events.

---

## 2. Order-Level Overview (6-Day Aggregate)

| Metric | Jul 7 | Jul 8 | Jul 9 | Jul 10 | Jul 11 | Jul 12 | **TOTAL** |
|--------|-------|-------|-------|--------|--------|--------|-----------|
| Order lines | 371,553 | 355,262 | 367,299 | 419,133 | 412,697 | 394,647 | **2,320,591** |
| Distinct orders | 22,349 | 22,106 | 22,546 | 25,683 | 24,934 | 22,146 | **139,764** |
| OOS rate (st5) | 2.9% | 2.4% | 2.2% | 2.1% | 2.2% | 2.8% | **2.4%** |
| Orders affected | 55.9% | 51.2% | 49.2% | 48.5% | 51.2% | 57.2% | **52.1%** |
| CLP lost (OOS) | 100.2M | 90.3M | 99.4M | 109.0M | 103.8M | 97.4M | **600.2M** |

**Quantity:** 4,129,005 units ordered, 3,944,369 delivered (95.5%).
**Value:** 10.77B CLP ordered. 600.2M CLP lost to OOS.

---

## 3. Population A — Propagation Gap (HANA ≤ 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed ≤ 0 before the order was placed.
**Root cause:** HANA already knew stock was zero, but the signal never propagated to VTEX → customer ordered anyway.

### Per-day breakdown

| Day | Lines | CLP | Tier 1 | Tier 2 | Tier 3 | Tier 4 | Tier 5 |
|-----|-------|-----|--------|--------|--------|--------|--------|
| Jul 7 | 1,978 | 11.57M | 391 | 145 | 139 | 642 | 661 |
| Jul 8 | 1,402 | 8.28M | 253 | 92 | 80 | 431 | 546 |
| Jul 9 | 1,488 | 9.31M | 314 | 86 | 67 | 427 | 594 |
| Jul 10 | 1,789 | 11.03M | 387 | 81 | 63 | 591 | 667 |
| Jul 11 | 1,789 | 11.03M | 387 | 81 | 63 | 591 | 667 |
| Jul 12 | 2,253 | 13.43M | 558 | 58 | 188 | 694 | 755 |
| **TOTAL** | **10,699** | **64.66M** | **2,290** | **543** | **600** | **3,376** | **3,890** |

### 6-day confirmation tier summary

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS | 2,290 | 21.4% | 10,643,863 |
| 2 | RESTOCK LIKELY | 543 | 5.1% | 2,901,601 |
| 3 | PICKED LATER | 600 | 5.6% | 3,600,859 |
| 4 | CONFIRMED STOCKOUT | 3,376 | 31.6% | 20,409,404 |
| 5 | SINGLE ATTEMPT | 3,890 | 36.4% | 27,105,896 |

### 3-bucket summary

| Bucket | Count | CLP |
|--------|-------|-----|
| YES — someone got it at/before (Tier 1) | 2,290 | 10,643,863 |
| YES — someone got it after (Tier 2+3) | 1,143 | 6,502,460 |
| NO — nobody got it (Tier 4+5) | 7,266 | 47,515,300 |

---

## 4. Population B — Phantom Inventory (HANA > 0, picker OOS)

**Definition:** Picker marked OOS AND HANA showed > 0 before the order was placed.
**Root cause:** HANA book stock ≠ physical shelf stock. ERP says units exist, but shelf is empty.

### Per-day breakdown

| Day | Lines | CLP | Tier 1 | Tier 2 | Tier 3 | Tier 4 | Tier 5 |
|-----|-------|-----|--------|--------|--------|--------|--------|
| Jul 7 | 8,510 | 46.83M | 2602 | 419 | 612 | 2170 | 2707 |
| Jul 8 | 7,235 | 39.63M | 2052 | 302 | 570 | 1774 | 2537 |
| Jul 9 | 6,527 | 37.21M | 1827 | 289 | 401 | 1675 | 2335 |
| Jul 10 | 7,295 | 40.20M | 2532 | 167 | 413 | 1563 | 2620 |
| Jul 11 | 7,295 | 40.20M | 2532 | 167 | 413 | 1563 | 2620 |
| Jul 12 | 8,665 | 45.48M | 3090 | 61 | 843 | 1788 | 2883 |
| **TOTAL** | **45,527** | **249.54M** | **14,635** | **1,405** | **3,252** | **10,533** | **15,702** |

### 6-day confirmation tier summary

| Tier | Meaning | Count | % | CLP |
|------|---------|-------|---|-----|
| 1 | PICKER MISS | 14,635 | 32.1% | 72,854,873 |
| 2 | RESTOCK LIKELY | 1,405 | 3.1% | 6,951,324 |
| 3 | PICKED LATER | 3,252 | 7.1% | 16,604,542 |
| 4 | CONFIRMED STOCKOUT | 10,533 | 23.1% | 56,544,005 |
| 5 | SINGLE ATTEMPT | 15,702 | 34.5% | 96,587,245 |

---

## 5. Combined Summary (Pop A + Pop B)

| | Pop A (HANA ≤ 0) | Pop B (HANA > 0) | **Combined** |
|---|---|---|---|
| **Lines** | 10,699 | 45,527 | **56,226** |
| **CLP** | 64.66M | 249.54M | **314.20M** |
| YES at/before (T1) | 2,290 (21.4%) | 14,635 (32.1%) | 16,925 |
| YES after (T2+3) | 1,143 (10.7%) | 4,657 (10.2%) | 5,800 |
| NO nobody (T4+5) | 7,266 (67.9%) | 26,235 (57.6%) | 33,501 |

**6-day total OOS impact: 314.2M CLP across 56,226 order lines.**
Projected monthly: ~1571M CLP.

---

## 6. VTEX Website Blindness — Hourly Classification

For each (sku, store) pair ordered on a given day, we built an hourly timeline comparing
HANA stock level (24 hourly snapshots) vs VTEX OOS state (from SQS events). This reveals
how many items the website kept showing as "available" despite HANA confirming zero stock.

### Per-day summary

| Day | Items Hit Zero | Never Caught | Caught | Blind % | Avg Delay (caught) |
|-----|---------------|-------------|--------|---------|-------------------|
| Jul 7 (Tuesday) | 6,868 | 5,573 | 1,295 | 81% | 3.1 hrs |
| Jul 8 (Wednesday) | 6,359 | 5,225 | 1,134 | 82% | 2.7 hrs |
| Jul 9 (Thursday) | 6,304 | 5,674 | 630 | 90% | 3.6 hrs |
| Jul 10 (Friday) | 7,642 | 6,330 | 1,312 | 83% | 2.2 hrs |
| Jul 11 (Saturday) | 6,572 | 5,944 | 628 | 90% | 3.3 hrs |
| Jul 12 (Sunday) | 6,382 | 5,866 | 516 | 92% | 4.3 hrs |
| **TOTAL** | **40,127** | **34,612** | **5,515** | **86%** | |

**Key finding:** 86% of items that run out of stock are NEVER removed from the website.
The website keeps showing them as available. Customers order, pickers can't find them.

### VTEX event volume vs catch rate

| Day | VTEX Events | Catch Rate | Notes |
|-----|------------|-----------|-------|
| Jul 7 | 2,400,000 | 18.9% | PUBLICADOR spike |
| Jul 8 | 2,300,000 | 17.8% | PUBLICADOR spike |
| Jul 9 | 618,000 | 10.0% | Organic only |
| Jul 10 | 905,164 | 17.2% | Organic only |
| Jul 11 | 545,332 | 9.6% | Organic only |
| Jul 12 | 652,783 | 8.1% | Organic only |

Pearson r = 0.835 (strong positive). PUBLICADOR batch spikes roughly DOUBLE the catch rate.

---

## 7. Revenue at Risk — Orders on Never-Caught Items

For the ~5,500-6,300 never-caught (sku,store) pairs per day, how many customer orders
were placed on those exact combos?

| Day | NC Pairs | Order Lines | Picker OOS | OOS Rate | CLP Lost |
|-----|---------|------------|-----------|---------|---------|
| Jul 7 | 5,573 | 19,222 | 1,995 | 10.4% | 11,920,221 |
| Jul 8 | 5,225 | 17,300 | 1,453 | 8.4% | 8,671,356 |
| Jul 9 | 5,674 | 18,789 | 1,488 | 7.9% | 9,439,692 |
| Jul 10 | 6,330 | 22,418 | 1,925 | 8.6% | 12,248,230 |
| Jul 11 | 5,944 | 19,794 | 1,887 | 9.5% | 11,478,307 |
| Jul 12 | 5,866 | 18,796 | 2,249 | 12.0% | 13,507,014 |
| **TOTAL** | **34,612** | **116,319** | **10,997** | **9.5%** | **67,264,820** |

**116,319 order lines** placed on items the website should have hidden.
**10,997 lines** failed at picking = **67.3M CLP** lost.
Projected monthly: **~336M CLP**.

### What happened to those 116,319 order lines?

| Outcome | Lines | % |
|---------|-------|---|
| Delivered OK (picker found stock despite HANA=0) | ~105,322 | ~91% |
| Picker OOS (confirmed failure) | 10,997 | 9.5% |

Items on never-caught pairs are **~4.2x more likely** to fail at picking than the average item.

---

## 8. Store-Level Blindness Ranking

Top 15 stores by OOS volume (6-day aggregate):

| Store | Total OOS | Never Caught | Caught | Blind % |
|-------|----------|-------------|--------|---------|
| J414 | 8,513 | 8,052 | 461 | 94.6% |
| J403 | 6,829 | 6,565 | 264 | 96.1% |
| J410 | 6,478 | 6,192 | 286 | 95.6% |
| J408 | 4,361 | 4,113 | 248 | 94.3% |
| J411 | 1,165 | 594 | 571 | 51.0% |
| J512 | 1,008 | 735 | 273 | 72.9% |
| J775 | 817 | 549 | 268 | 67.2% |
| J513 | 727 | 436 | 291 | 60.0% |
| J989 | 677 | 394 | 283 | 58.2% |
| J519 | 610 | 461 | 149 | 75.6% |
| J504 | 600 | 492 | 108 | 82.0% |
| J659 | 591 | 423 | 168 | 71.6% |
| J843 | 567 | 435 | 132 | 76.7% |
| J534 | 552 | 428 | 124 | 77.5% |
| J633 | 531 | 403 | 128 | 75.9% |

**Worst stores (highest blind %):** J403, J410, J414, J408, J592 (88-96%)
**Best stores (lowest blind %):** J780, J411, J760, J989, J513 (50-60%)

---

## 9. Chronic Repeat Offenders

(sku, store) pairs that were never-caught across multiple days:

| Days Never-Caught | (sku,store) Pairs |
|-------------------|------------------|
| 1 of 6 days | 8,487 |
| 2 of 6 days | 3,752 |
| 3 of 6 days | 1,916 |
| 4 of 6 days | 1,196 |
| 5 of 6 days | 713 |
| 6 of 6 days | 754 |

**754 (sku,store) pairs were never caught across ALL 6 days.**

### SKUs never-caught at the most stores (4+ days):

| SKU | Stores | Product |
|-----|--------|---------|
| 9083 | 33 | Pan Ciabatta Granel |
| 53629 | 33 | Pan Hallulla Granel |
| 76215 | 32 | Pan Hallulla Delgada Granel |
| 96332 | 32 | Pan Marraqueta Grande Granel |
| 85469 | 31 | Pan Hallulla Integral Linaza Granel |
| 53388 | 31 | Pan Hot Dog Granel |
| 53389 | 29 | Pan de Hamburguesa Granel |
| 71734 | 29 | Piña Pelada 1 un. |
| 53390 | 29 | Pan Doblada Granel |
| 9082 | 28 | Pan Ciabatta Integral Granel |

---

## 10. Delay Proof Stats

| Day | Preventable Events | Distinct Orders | Gap Min (min) | Gap Mean (min) | Gap Max (min) |
|-----|-------------------|----------------|--------------|---------------|--------------|
| Jul 7 | 17,680 | 10,662 | 0 | 30 | 61 |
| Jul 8 | 15,551 | 9,889 | 0 | 30 | 61 |
| Jul 9 | 17,261 | 10,464 | 0 | 30 | 62 |
| Jul 10 | 19,165 | 11,810 | 0 | 30 | 61 |
| Jul 11 | 17,735 | 11,168 | 0 | 30 | 61 |
| Jul 12 | 17,825 | 10,703 | 0 | 30 | 61 |
| **TOTAL** | **105,217** | **64,696** | | | |

---

## 11. Methodology

### Pipeline (per day)

1. **Export orders** (`export_orders_MMDD.py`) — Redshift `io_items` JOIN `vw_master_pickers_data`
2. **Download HANA** — 24 hourly parquet batches from S3 (`vw_daily_nrt_MMDD_HHMM/`)
3. **Delay proof** (`delay_proof_MMDD.py`) — Cross-ref each order line with most recent HANA batch before `fecha_creacion`
4. **Evidence pack** (`evidence_pack_MMDD.py`) — Pop A/B + confirmation tiers + evidence flags + Redshift enrichment
5. **VTEX classification** (`hana_vtex_classify.py MMDD`) — Hourly HANA vs VTEX state comparison

### Confirmation Tier Logic

```sql
CASE
  WHEN got_before > 0
     THEN '1 PICKER MISS (same item picked at/before this order)'
  WHEN got_after > 0 AND hana_rose_after
     THEN '2 RESTOCK LIKELY (picked later + HANA rose)'
  WHEN got_after > 0
     THEN '3 PICKED LATER (only later, HANA flat)'
  WHEN n_orders_item > 1
     THEN '4 CONFIRMED STOCKOUT (multiple orders, all failed)'
  ELSE '5 SINGLE ATTEMPT (cause unconfirmed)'
END
```

### CSV read pattern (DuckDB)

Always apply 6 VARCHAR overrides: `dateupdate`, `inicio_picking`, `fin_picking`, `venta_neta`, `items_solicitados`, `items_faltantes`.

---

## 12. Key Output Files

| Day | Evidence Pack | Pop A CSV | Pop B CSV | Delay Proof | VTEX Classification |
|-----|-------------|-----------|-----------|-------------|-------------------|
| Jul 7 | `evidence_pack_jul07.xlsx` | `evidence_pop_a_jul07.csv` | `evidence_pop_b_jul07.csv` | `delay_proof_0707_preventable.parquet` | `hana_vtex_hourly_classification.csv` |
| Jul 8 | `evidence_pack_jul08.xlsx` | `evidence_pop_a_jul08.csv` | `evidence_pop_b_jul08.csv` | `delay_proof_0708_preventable.parquet` | `hana_vtex_hourly_classification.csv` |
| Jul 9 | `evidence_pack_jul09.xlsx` | `evidence_pop_a_jul09.csv` | `evidence_pop_b_jul09.csv` | `delay_proof_0709_preventable.parquet` | `hana_vtex_hourly_classification.csv` |
| Jul 10 | `evidence_pack_jul10.xlsx` | `evidence_pop_a_jul10.csv` | `evidence_pop_b_jul10.csv` | `delay_proof_0710_preventable.parquet` | `hana_vtex_hourly_classification.csv` |
| Jul 11 | `evidence_pack_jul11.xlsx` | `evidence_pop_a_jul11.csv` | `evidence_pop_b_jul11.csv` | `delay_proof_0711_preventable.parquet` | `hana_vtex_hourly_classification.csv` |
| Jul 12 | `evidence_pack_jul12.xlsx` | `evidence_pop_a_jul12.csv` | `evidence_pop_b_jul12.csv` | `delay_proof_0712_preventable.parquet` | `hana_vtex_hourly_classification.csv` |

All files in `analysis/julDD/`. Orders CSVs in `exports/cencommerce_quiebre/`.
HANA parquets in `data_samples/vw_daily_nrt_MMDD_HHMM/`.
