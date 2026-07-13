# July 6, 2026 — VTEX Propagation Analysis Reference

> Generated 2026-07-09. Extends JUL6_COMPLETE_REFERENCE.md with VTEX SQS event layer.
> Answers: "What was VTEX telling the customer at the moment they ordered?"
> All stats use **Jul 6 CLT events only** (00:00-23:59 CLT). Jul 2 events in the JSONL files are excluded from analysis.

---

## 1. Data Sources

| Source | Path | Records | Notes |
|--------|------|---------|-------|
| **VTEX SQS events (Jul 6)** | `analysis/vtex_events/jul6/*.jsonl` | 1,417,919 Jul 6 events (2,228,589 total in files incl. Jul 2) | 7 JSONL files, drained from SQS |
| **VTEX SQS events (Jul 7)** | `analysis/vtex_events/jul7/*.jsonl` | 2,401,483 events | 2 JSONL files |
| **VTEX-SAP mapping** | `analysis/vtex_sap_mapping.csv` | 74,804 rows | From Redshift `dim_surtidos` |
| **Preventable parquet** | `analysis/jul6/delay_proof_0706_preventable.parquet` | 19,241 rows | All HANA<=0 order lines (from JUL6 analysis) |
| **Enriched output** | `analysis/jul6/preventable_with_vtex_jul6.csv` | 19,241 rows | Every line enriched with Jul 6 VTEX state |

### SQS drain details

```
Queue:    hd-schn-ordm-giv-vtex-stock-events
Account:  103413823818 (prod)
Profile:  eks-csc (AWS SSO)
Script:   scripts/drain_sqs_fast.py
Method:   Non-destructive (ReceiveMessage with VisibilityTimeout, no DeleteMessage)
Workers:  20 parallel, long-poll
```

Messages are not deleted from the queue. The drain uses VisibilityTimeout only -- messages reappear after timeout.

The script supports `KEEP_FROM_MS` / `KEEP_TO_MS` env vars to filter events by timestamp window. When set, events outside the window are received from SQS but NOT written to the JSONL file (counted as `skip_win` in terminal output). When not set (default: 0, falsy), no window filtering is applied and all events are kept.

The Jul 6 drain runs did NOT set `KEEP_FROM_MS`, so all events in the queue were written. The resulting JSONL files contain events from two dates:

| Date | Events (in files) | Events (filtered for analysis) |
|------|-------------------|-------------------------------|
| Jul 2 | 787,696 | excluded |
| Jul 6 | 1,440,893 | **1,417,919** (jumboclj stores only) |

All analysis in this document uses Jul 6 CLT events only (receivedAt between Jul 6 00:00 CLT and Jul 7 00:00 CLT, i.e. Jul 6 04:00 UTC to Jul 7 04:00 UTC).

---

## 2. VTEX Event Structure

Each SQS message is a JSON object. The relevant fields:

```json
{
  "receivedAt": 1783350000000,       // UTC epoch milliseconds (always UTC)
  "attributes": {
    "skuId": "9079",                  // VTEX SKU identifier
    "storeId": "jumboclj989",        // VTEX store identifier
    "outOfStock": false,              // boolean: true = OOS, false = in stock
    "stockLevel": "in-stock"          // string: "in-stock" or "out-of-stock"
  }
}
```

### Critical limitations

| Property | Reality |
|----------|---------|
| **Binary only** | `stockLevel` is `in-stock` or `out-of-stock`. No actual quantity. VTEX does not know how many units exist. |
| **Not per-sale** | Events fire on stock **state changes**, not on every sale. An item can sell 50 units and produce 0 events if it never crosses the threshold. |
| **Timestamps are UTC** | `receivedAt` is Unix milliseconds in UTC. Chile winter = UTC-4. All conversions in this doc subtract 4 hours. |
| **Duplicates exist** | Some events appear twice with identical timestamps (SQS at-least-once delivery). |

### Jul 6 event stats (CLT day only)

| Metric | Value |
|--------|-------|
| Total events | 1,417,919 |
| Distinct (skuId, storeId) combos | 782,858 |
| Distinct skuIds | 27,644 |
| Combos with exactly 1 event | 74.0% |
| Median events per combo | 1 |
| Mean events per combo | 1.8 |

### Hourly distribution (CLT)

Events are NOT evenly distributed. Massive spike at 16:00-20:00 CLT = PUBLICADOR batch push (not organic sales).

```
00:00-03:00  ~25K-60K/hr    (overnight, low)
04:00-08:00  ~30K-75K/hr    (store opening)
09:00-15:00  ~50K-90K/hr    (daytime organic)
16:00-20:00  ~157K-248K/hr  (PUBLICADOR batch spike)
21:00-23:00  ~40K-90K/hr    (wind-down)
```

---

## 3. Mapping: VTEX <-> HANA

### SKU mapping

| | Source | File | Rows |
|---|---|---|---|
| **Primary (used)** | Redshift `dim_surtidos` | `analysis/vtex_sap_mapping.csv` | 74,804 |
| Alternate (NOT used) | Unknown source | `analysis/vtex_sap_mapping_FULL.xlsx` | 147,347 |

**Why CSV over XLSX:** CSV has 99.91% coverage of Jul 6 VTEX events (27,647/27,673 matched). XLSX has only 85.9% coverage and contains suffixed `sku_sap` values (e.g. `1871755-KG`) that need extra parsing. CSV is strictly better.

**Mapping chain:**
```
VTEX skuId  --(vtex_sap_mapping.csv)--> sku_sap  --(zero-pad to 18 digits)--> HANA Item_Id
Example:     9079                        991863                                000000000000991863
```

**Extraction script:** `scripts/utils/extract_vtex_sap_mapping.py` -- queries Redshift `dim_surtidos`, validated against 4 known samples from Javiera.

### Store mapping

```
VTEX storeId     -->  strip 'jumboclj'  -->  add 'J' prefix  -->  HANA Location_Id
jumboclj989      -->  989               -->  J989             -->  J989
jumboclj414      -->  414               -->  J414             -->  J414
jumboclj411rincon --> 411rincon          -->  J411  (strip 'rincon')
```

### Coverage against our analysis (Jul 6 events only)

| Population | Total (item, store) pairs | Have Jul 6 VTEX event | Coverage |
|---|---|---|---|
| All HANA<=0 (19,241 lines = 5,933 unique pairs) | 5,933 | 5,840 | **98.4%** |
| Pop A (picker OOS, status 3/5/7/14) | 1,516 | ~1,500 | ~98.9% |
| Combined Pop A + Pop B | 8,163 | ~8,100 | ~99.2% |

**93 pairs with no Jul 6 VTEX event:** These items had no stock state change on Jul 6 -- their VTEX state was set on a prior day and persisted unchanged.

**3 pairs with no mapping:** Item IDs not found in `vtex_sap_mapping.csv` at all.

### Unmapped VTEX skuIds

26 out of 27,644 VTEX skuIds (Jul 6) have no match in `vtex_sap_mapping.csv`. These are a contiguous block (168850-169078) -- likely newly added products not yet in `dim_surtidos`. File: `analysis/jul6/unmapped_vtex_skus_jul6.csv`.

Only 1 of the 26 appeared in the XLSX mapping. The XLSX adds negligible value.

---

## 4. Core Analysis: VTEX State at Order Time

### Method

For each of the 19,241 order lines in `delay_proof_0706_preventable.parquet`:

1. Get `fecha_creacion` (order creation time, CLT timezone)
2. Convert to UTC epoch milliseconds (CLT = UTC-4)
3. Map `item_id_for_hana` -> VTEX `skuId` via `vtex_sap_mapping.csv`
4. Map `id_tienda` (e.g. `J989`) -> VTEX `storeId` (e.g. `jumboclj989`)
5. Build timeline of **Jul 6 CLT events only** for that (skuId, storeId)
6. Binary search for the last event with `receivedAt` < order time
7. Read the `outOfStock` field from that event = what VTEX was showing when the customer ordered

### Results

| VTEX state at order time | Lines | % | CLP |
|---|---|---|---|
| **IN_STOCK** | **14,875** | **77.3%** | **57,489,332** |
| NO_PRIOR_EVENT | 4,025 | 20.9% | 26,250,534 |
| NO_VTEX_EVENT | 261 | 1.4% | 1,119,986 |
| OOS | 77 | 0.4% | 439,456 |
| NO_MAPPING | 3 | 0.0% | 16,968 |

**77.3% of the time, a Jul 6 VTEX event confirmed "this item is available" while HANA already knew stock was at zero.**

### Breakdown by picker status

| Status | Meaning | Total | VTEX IN_STOCK | VTEX OOS | NO_PRIOR | NO_VTEX |
|--------|---------|-------|---------------|----------|----------|---------|
| 1 | Picker FOUND it (false alarm) | 13,980 | 11,163 (79.9%) | 13 | 2,665 | 139 |
| 5 | Picker NOT FOUND (Pop A) | 2,607 | **1,769** (67.9%) | 9 | 792 | 37 |
| 4 | Substituted | 1,634 | 1,188 (72.7%) | 8 | 401 | 37 |
| 13 | Weight item OOS | 415 | 362 (87.2%) | 0 | 52 | 1 |
| 11 | Added item | 382 | 202 (52.9%) | 47 | 83 | 47 |
| 12 | Partial weight pick | 222 | 190 (85.6%) | 0 | 32 | 0 |
| 3 | Sin stock (legacy) | 1 | 1 | 0 | 0 | 0 |

### The preventable core: Pop A with VTEX IN_STOCK

Of the 2,607 Pop A lines (status 5, HANA<=0, picker confirmed OOS):

```
2,607 Pop A lines
|- VTEX said IN_STOCK at order time:   1,769 (67.9%)  <- CONFIRMED PREVENTABLE
|- VTEX had NO_PRIOR_EVENT:              792 (30.4%)  <- likely also IN_STOCK (stale from prior day)
|- VTEX had NO_VTEX_EVENT:                37 ( 1.4%)  <- no Jul 6 data
|- VTEX said OOS:                          9 ( 0.3%)  <- customer ordered despite OOS signal
```

**1,769 lines where all three sources agree on the failure chain (Jul 6 data only):**
- HANA: "stock is zero" (correct)
- VTEX: "item is available" (wrong -- didn't update, confirmed by Jul 6 event)
- Picker: "item not found" (confirmed)

If VTEX had reflected HANA's zero stock, those 1,769 orders would never have been placed.

The 792 NO_PRIOR_EVENT lines had no Jul 6 VTEX event before the order -- the website was showing whatever state was set on a previous day (likely IN_STOCK since the order was allowed through). Combined with IN_STOCK, that's **2,561 (98.2%)** of Pop A lines where the customer almost certainly saw "in stock".

---

## 5. State Definitions

### What each VTEX state means

| State | Definition | Lines | What happened |
|---|---|---|---|
| **IN_STOCK** | Last Jul 6 VTEX event before order time had `outOfStock=false` | 14,875 | A Jul 6 VTEX event confirmed the item was shown as available to the customer at order time. |
| **NO_PRIOR_EVENT** | Jul 6 VTEX events exist for this (sku, store), but ALL are AFTER the order time | 4,025 | No Jul 6 VTEX update had arrived before the customer ordered. Website was showing whatever state was set on a previous day (likely IN_STOCK since the order was allowed through). |
| **NO_VTEX_EVENT** | Zero Jul 6 VTEX events for this (sku, store) | 261 | This item-store combo had no stock change signal on Jul 6. Stale state from a prior day. |
| **OOS** | Last Jul 6 VTEX event before order time had `outOfStock=true` | 77 | VTEX correctly showed OOS, but the customer ordered anyway. Likely a cached page or race condition. |
| **NO_MAPPING** | Item has no entry in `vtex_sap_mapping.csv` | 3 | Cannot look up VTEX data -- item not in `dim_surtidos`. |

### Interpreting NO_PRIOR_EVENT (4,025 lines)

These items had Jul 6 VTEX events, but all arrived AFTER the order was placed. The customer ordered before VTEX sent any update that day. The website was displaying whatever state was set on a prior day -- almost certainly IN_STOCK since the order was allowed through.

We cannot prove this from Jul 6 data alone. To confirm, we would need the prior day's last VTEX event for each item.

---

## 6. Enriched Output File

**File:** `analysis/jul6/preventable_with_vtex_jul6.csv`
**Rows:** 19,241 (every HANA<=0 order line)
**Size:** 36.0 MB
**VTEX scope:** Jul 6 CLT events only

### Column reference

| Column | Source | Description |
|--------|--------|-------------|
| `order_id` | Redshift | Cencommerce order ID (e.g. `v232739010jmch-01`) |
| `id_tienda` | Redshift | Store code (e.g. `J989`) |
| `tienda` | Redshift | Store name (e.g. `JUMBO INDEPENDENCIA`) |
| `item_id_for_hana` | Redshift/HANA | 18-digit zero-padded SAP item ID |
| `sku` | Redshift | VTEX SKU ID (numeric) |
| `name` | Redshift | Product name |
| `refid` | Redshift | Cencommerce reference ID (may have `-KG` suffix for weight items) |
| `status` | Redshift | Picker status code (1=found, 3/5/14=sin stock, 4=sustituto, etc.) |
| `originalquantity` | Redshift | Units customer ordered |
| `pickingquantity` | Redshift | Units picker actually picked (0 = OOS confirmed) |
| `monto_clp` | Redshift | Line value in CLP |
| `fecha_creacion` | Redshift | Order creation time (**CLT**, Chile local, UTC-4 winter) |
| `inicio_picking` | Redshift | Picker start time (CLT) |
| `fin_picking` | Redshift | Picker end time (CLT) |
| `hana_last_batch_ts` | HANA parquet | Timestamp of last HANA batch before order (CLT) |
| `hana_qty_at_last_batch` | HANA parquet | `Ending_On_Hand_Qty` at that batch (always <= 0 in this file) |
| `gap_minutes` | Computed | Minutes between HANA batch and order creation |
| `vtex_sku` | Mapping | Mapped VTEX skuId (from `vtex_sap_mapping.csv`) |
| `vtex_store` | Mapping | Mapped VTEX storeId (e.g. `jumboclj989`) |
| `vtex_state_at_order` | VTEX events | What VTEX was showing at order time (Jul 6 events only): `IN_STOCK`, `OOS`, `NO_PRIOR_EVENT`, `NO_VTEX_EVENT`, `NO_MAPPING` |
| `vtex_last_event_utc` | VTEX events | Timestamp of last Jul 6 VTEX event before order (**UTC**) |
| `vtex_last_event_chile` | VTEX events | Same timestamp converted to **CLT** |
| `vtex_events_count_jul6` | VTEX events | Total number of Jul 6 VTEX events for this (sku, store) |
| `vtex_all_events_jul6` | VTEX events | Full Jul 6 timeline: `09:21:54CLT=IN \| 15:43:35CLT=OOS \| ...` |

### Timezone handling

| Field | Timezone | How we know |
|-------|----------|-------------|
| `receivedAt` (VTEX SQS) | **UTC** | Unix epoch milliseconds are always UTC by definition |
| `fecha_creacion` (Redshift) | **CLT** (UTC-4) | Confirmed by order hour distribution matching Chilean business hours |
| `hana_last_batch_ts` (HANA) | **CLT** | Confirmed in `analysis/findings/TIMEZONE_PROOF.md` |
| `vtex_last_event_utc` | **UTC** | Converted from `receivedAt` |
| `vtex_last_event_chile` | **CLT** | `receivedAt` converted to America/Santiago |
| `vtex_all_events_jul6` | **CLT** | All times shown in CLT for readability |

To match VTEX events against order time: convert `fecha_creacion` (CLT) to UTC epoch ms, then binary search the VTEX timeline (which is in UTC epoch ms). This is what the enrichment script does.

---

## 7. Example Cases

### Case 1: POP A + VTEX IN_STOCK (the preventable OOS) -- 1,769 lines

**Pan Campo 550g at Jumbo Independencia**

```
order_id              = v232739010jmch-01
id_tienda             = J989 (JUMBO INDEPENDENCIA)
item_id_for_hana      = 000000000000991863
name                  = Pan Campo 550 g
status                = 5 (picker: SIN STOCK)
originalquantity      = 1.0
pickingquantity       = 0.0 (picked nothing)
monto_clp             = 2,490
fecha_creacion        = 2026-07-06 05:01:45 CLT
inicio_picking        = 2026-07-06 11:24:58 CLT
fin_picking           = 2026-07-06 12:08:48 CLT
hana_last_batch_ts    = 2026-07-06 04:40:49 CLT
hana_qty_at_last_batch = 0.0
gap_minutes           = 21
vtex_sku              = 9079
vtex_store            = jumboclj989
vtex_state_at_order   = IN_STOCK
vtex_last_event_chile = 2026-07-06 00:19:47 CLT
vtex_events_count     = 3
vtex_all_events       = 00:19:47CLT=IN | 05:01:52CLT=IN | 18:48:56CLT=IN
```

**Timeline:**
```
00:19  VTEX event: IN_STOCK (Jul 6)
04:40  HANA batch: qty = 0  (HANA knows stock is gone)
05:01  Customer orders (VTEX showing IN_STOCK -- gap_minutes=21 since HANA knew)
05:01  VTEX event: IN_STOCK (7 seconds after order -- still saying "in stock")
11:24  Picker starts
12:08  Picker finishes -- marked SIN STOCK, picked 0 units
```

**Story:** HANA knew stock was 0 at 04:40. VTEX sent a Jul 6 event at 00:19 saying "in stock". Customer ordered at 05:01 seeing "in stock". Picker confirmed OOS 6 hours later. The order should never have been allowed.

---

### Case 2: FALSE ALARM + VTEX IN_STOCK (HANA wrong, picker found it) -- 11,163 lines

**Tarta Banoffee at Jumbo La Reina**

```
order_id              = v232741505jmch-01
id_tienda             = J512 (JUMBO LA REINA)
item_id_for_hana      = 000000000002008697
name                  = Tarta Banoffee 1 un.
status                = 1 (picker: FOUND IT)
originalquantity      = 1.0
pickingquantity       = 1.0 (picked full amount)
monto_clp             = 9,990
fecha_creacion        = 2026-07-06 08:27:38 CLT
hana_last_batch_ts    = 2026-07-06 07:40:20 CLT
hana_qty_at_last_batch = 0.0
gap_minutes           = 47
vtex_state_at_order   = IN_STOCK
vtex_last_event_chile = 2026-07-06 08:27:31 CLT  (7 seconds before order!)
vtex_events_count     = 5
vtex_all_events       = 08:27:31CLT=IN | 08:27:43CLT=IN | 11:06:24CLT=IN | 13:53:33CLT=IN | 16:38:42CLT=IN
```

**Story:** HANA said 0 stock at 07:40, but VTEX sent an IN_STOCK event at 08:27:31 -- just 7 seconds before the order at 08:27:38. Picker walked to the shelf and found it. HANA's book stock was wrong -- physical stock existed. VTEX happened to be correct.

---

### Case 3: SUSTITUTO + VTEX IN_STOCK (customer got substitute) -- 1,188 lines

**Toalla de Papel at Jumbo Antofagasta**

```
order_id              = v232757770jmch-01
id_tienda             = J534 (JUMBO ANTOFAGASTA ANGAMOS)
name                  = Toalla de Papel Favorita Estilo 24 m 2 un.
status                = 4 (SUSTITUTO)
pickingquantity       = 0.0 (original not picked)
monto_clp             = 2,190
fecha_creacion        = 2026-07-06 11:40:51 CLT
hana_qty_at_last_batch = -2.0
gap_minutes           = 60
vtex_state_at_order   = IN_STOCK
vtex_last_event_chile = 2026-07-06 10:50:17 CLT
vtex_events_count     = 5
vtex_all_events       = 00:20:40CLT=OOS | 10:50:17CLT=IN | 11:40:55CLT=IN | 12:49:38CLT=OOS | 20:24:50CLT=OOS
```

**Story:** This item flipped states throughout Jul 6: OOS at midnight, back to IN at 10:50, the customer ordered at 11:40 (seeing IN_STOCK), then flipped back to OOS at 12:49. HANA said -2 the whole time. Picker couldn't find it, offered a substitute. The VTEX timeline shows the signal DID eventually arrive (OOS at 12:49) -- but 1 hour too late.

---

### Case 4: POP A + VTEX OOS (VTEX correct, order went through anyway) -- 9 lines

**Croqueta Espanola at Jumbo Nunoa**

```
order_id              = v232802910jmch-01
id_tienda             = J775 (JUMBO NUNOA)
name                  = Croqueta Espanola de Jamon La Culinaria 240 g
status                = 5 (SIN STOCK)
monto_clp             = 2,495
fecha_creacion        = 2026-07-06 18:56:05 CLT
hana_qty_at_last_batch = 0.0
gap_minutes           = 75
vtex_state_at_order   = OOS
vtex_last_event_chile = 2026-07-06 16:47:51 CLT
vtex_events_count     = 1
vtex_all_events       = 16:47:51CLT=OOS
```

**Story:** VTEX correctly sent an OOS event at 16:47. Customer ordered at 18:56 -- 2 hours after VTEX said OOS. The order went through anyway -- cached page, race condition, or website not enforcing the OOS flag. Only 9 Pop A lines (77 across all statuses) have this pattern; it's very rare.

---

### Case 5: FOUND + NO_PRIOR_EVENT (no Jul 6 VTEX update before order) -- 2,665 lines

**Jamon Pierna at Jumbo Independencia**

```
order_id              = v232739010jmch-01
id_tienda             = J989 (JUMBO INDEPENDENCIA)
name                  = Jamon Pierna Acaramelado Al Vacio 250 g
status                = 1 (FOUND IT)
originalquantity      = 0.25
pickingquantity       = 0.238 (picked ~full amount)
monto_clp             = 3,990
fecha_creacion        = 2026-07-06 05:01:45 CLT
hana_qty_at_last_batch = -6.848
vtex_state_at_order   = NO_PRIOR_EVENT
vtex_events_count     = 2
vtex_all_events       = 05:01:52CLT=IN | 19:25:10CLT=IN
```

**Story:** Order at 05:01:45, first Jul 6 VTEX event at 05:01:52 -- 7 seconds AFTER. No Jul 6 VTEX event existed before the order. Website was showing whatever state was set on a previous day (almost certainly IN_STOCK since the order was allowed). Picker found it despite HANA saying -6.8 -- false alarm.

---

### Case 6: POP A + NO_VTEX_EVENT (zero Jul 6 VTEX data) -- 261 lines

**Leche Surlat at Darkstore Costanera**

```
order_id              = v232739668jmch-01
id_tienda             = J411 (DARKSTORE COSTANERA)
name                  = Leche Surlat Proteina Descremada Sin Lactosa 1 L
status                = 5 (SIN STOCK)
originalquantity      = 6.0
pickingquantity       = 0.0
monto_clp             = 15,540
hana_qty_at_last_batch = -2.0
vtex_state_at_order   = NO_VTEX_EVENT
vtex_sku              = 121065 (note: Redshift sku=121063 -- possible mapping mismatch)
vtex_events_count     = 0
```

**Story:** Zero Jul 6 VTEX events for this item-store. The mapping gave VTEX sku 121065 but Redshift sku is 121063 -- a mismatch that explains the miss. These 261 lines are a blind spot where we cannot determine VTEX state from Jul 6 data.

---

## 8. Methodology: How We Got Here

### Step 1 -- Drain VTEX SQS events

```
Script:   scripts/drain_sqs_fast.py
Queue:    hd-schn-ordm-giv-vtex-stock-events (account 103413823818)
Method:   Non-destructive ReceiveMessage (20 workers, long-poll)
Output:   analysis/vtex_events/jul6/*.jsonl (7 files, 2.2M events total, 1.4M Jul 6 CLT)
          analysis/vtex_events/jul7/*.jsonl (2 files, 2.4M events)
Needs:    AWS SSO (eks-csc profile) + VPN
```

Messages are not deleted from the queue. The drain uses VisibilityTimeout only -- messages reappear after timeout. This means the same events can be drained again in future sessions.

### Step 2 -- Build VTEX-SAP mapping

```
Script:   scripts/utils/extract_vtex_sap_mapping.py
Source:   Redshift dim_surtidos table
Output:   analysis/vtex_sap_mapping.csv (74,804 rows)
Columns:  sku_vtex, sku_sap, sku_sap_raw, sku_ean
```

Maps VTEX skuId to SAP item number. Zero-pad `sku_sap` to 18 digits = HANA `Item_Id`.

### Step 3 -- Verify coverage

```
Tool:     DuckDB (ad-hoc queries in conversation)
Check 1:  27,644 distinct Jul 6 VTEX skuIds, 99.91% matched in mapping
Check 2:  5,840/5,933 HANA<=0 pairs have Jul 6 VTEX events (98.4%)
Output:   analysis/jul6/unmapped_vtex_skus_jul6.csv (26 unmatched)
```

### Step 4 -- Enrich every preventable line with VTEX state

```
Tool:     Python script (in-conversation, not saved as standalone)
Input:    delay_proof_0706_preventable.parquet (19,241 rows)
        + analysis/vtex_events/jul6/*.jsonl (filtered to Jul 6 CLT only: 1,417,919 events)
        + analysis/vtex_sap_mapping.csv
Filter:   receivedAt >= Jul 6 00:00 CLT AND < Jul 7 00:00 CLT
Method:   For each order line:
          1. Map item_id_for_hana -> vtex_sku via mapping CSV
          2. Map id_tienda (J989) -> vtex_store (jumboclj989)
          3. Build timeline of Jul 6 CLT VTEX events for (sku, store)
          4. Binary search for last event BEFORE fecha_creacion (converted CLT->UTC ms)
          5. Read outOfStock field = vtex_state_at_order
Output:   analysis/jul6/preventable_with_vtex_jul6.csv (19,241 rows, 36.0 MB)
```

---

## 9. Summary: The Three-Source Verdict

For each of the 19,241 order lines where HANA <= 0:

```
Three sources, one question: should this order have been allowed?

HANA (ERP):     "Stock is zero"       (knew before the order)
VTEX (website): "Item is available"   (told the customer to order)
PICKER (human): "Item not found"      (confirmed on the shelf -- for Pop A)
                "Item found"          (contradicted HANA -- for false alarms)
```

### The failure chain (Pop A, 1,769 VTEX-confirmed preventable lines):

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

The breakpoint is step 2. HANA knew. The signal never propagated. VTEX kept showing "in stock". The customer ordered a ghost.

### By the numbers

| What we measured | Count | CLP |
|---|---|---|
| All HANA<=0 order lines | 19,241 | 85,316,276 |
| VTEX showed IN_STOCK at order time (Jul 6 event) | 14,875 (77.3%) | 57,489,332 |
| Of those, picker confirmed OOS (Pop A) | 1,769 | ~9.4M |
| Of those, picker found it anyway (false alarm) | 11,163 | ~43.0M |
| VTEX showed OOS but order went through | 77 (0.4%) | 439,456 |
| No Jul 6 VTEX event before order (stale prior-day state) | 4,025 (20.9%) | 26,250,534 |

---

## 10. Limitations and Caveats

| Limitation | Impact |
|---|---|
| **VTEX events are binary** | We can only say "in stock" or "out of stock", not quantity. Cannot measure partial depletion. |
| **SQS drain is a snapshot** | We captured whatever was in the queue at drain time. Events older than retention period are lost. |
| **Jul 6 only scope** | Analysis uses only Jul 6 CLT events. The JSONL files also contain Jul 2 events (787K) which are excluded. Prior-day state is unknown for 4,025 NO_PRIOR_EVENT lines. |
| **NO_PRIOR_EVENT is ambiguous** | 4,025 lines had no Jul 6 VTEX event before order time. The state was set on a prior day -- almost certainly IN_STOCK (order was allowed), but not provable from Jul 6 data alone. |
| **PUBLICADOR contaminates** | The 16-20h CLT spike is PUBLICADOR batch-pushing stock states, not organic sales. VTEX state changes during this window may not reflect real physical stock changes. |
| **Mapping imperfect** | 26 VTEX skuIds unmapped. 3 HANA item_ids unmapped. Some sku mismatches observed (e.g. 121063 vs 121065). |
| **Deduplication not applied** | Some SQS events appear twice (at-least-once delivery). Does not affect `vtex_state_at_order` (binary search still correct). |

---

## 11. Output Files

| File | Rows | Description |
|------|------|-------------|
| `analysis/jul6/preventable_with_vtex_jul6.csv` | 19,241 | Every HANA<=0 order line enriched with Jul 6 VTEX state, timestamps, and full event timeline |
| `analysis/jul6/unmapped_vtex_skus_jul6.csv` | 26 | VTEX skuIds not in mapping file |
| `analysis/jul6/vtex_sample_events.csv` | ~8 SKUs | Raw VTEX event samples used for exploration |
| `analysis/vtex_sap_mapping.csv` | 74,804 | Authoritative VTEX-SAP mapping from dim_surtidos |

---

## 12. Connection to JUL6_COMPLETE_REFERENCE.md

This document extends Section 4a of the Jul 6 reference (the 19,241 HANA<=0 branch) by adding the VTEX dimension. The drill-down now looks like:

```
419,497 order lines
|- 406,788 with HANA snapshot
|  |- 19,241 HANA <= 0
|  |  |- 14,875 VTEX said IN_STOCK at order time  (77.3%)    <- THIS DOC
|  |  |  |- 11,163 picker FOUND it (false alarm)
|  |  |  |- 1,769 picker NOT FOUND (VTEX-confirmed preventable Pop A)
|  |  |  |- 1,188 substituted
|  |  |  |- remainder: weight OOS, partial, added
|  |  |- 4,025 NO_PRIOR_EVENT  (20.9%)
|  |  |- 261 NO_VTEX_EVENT  (1.4%)
|  |  |- 77 VTEX said OOS  (0.4%)
|  |  |- 3 NO_MAPPING  (0.0%)
|  |- 387,547 HANA > 0
|- 12,709 no HANA snapshot
```

The evidence pack (Pop A/B confirmation tiers, 3-bucket summary, evidence flags) remains in the main reference doc. This doc adds the "what was the website showing?" layer on Jul 6.
