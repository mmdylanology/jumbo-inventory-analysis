# Jumbo Inventory Analysis — Reference Docs

OOS (Out-of-Stock) and VTEX propagation delay analysis for Cencosud Jumbo ecommerce (Jul 2026).

## Structure

```
jul1/   Complete OOS reference (18 HANA batches, no VTEX layer)
jul6/   Complete OOS reference + VTEX propagation analysis
jul7/   Complete OOS reference + VTEX delay classification (A/B/C/D)
jul8/   Complete OOS reference + VTEX delay classification
jul9/   Complete OOS reference + VTEX delay classification (organic baseline)
```

## Document Types

- **`JULX_COMPLETE_REFERENCE.md`** — Full OOS analysis: HANA × Redshift orders, Pop A (propagation gap) + Pop B (phantom inventory), confirmation tiers, evidence flags, CLP impact.
- **`VTEX_ANALYSIS_JULX.md`** — VTEX SQS event layer: propagation delay stats, A/B/C/D case classification, PUBLICADOR impact, cross-day comparison.

## Summary (Jul 1–9)

| Day | Pop A | Pop A CLP | Pop B | Pop B CLP | Combined CLP |
|-----|-------|-----------|-------|-----------|-------------|
| Jul 1 | 1,482 | 8.53M | 6,911 | 37.59M | 46.13M |
| Jul 6 | 2,608 | 13.88M | 10,613 | 55.56M | 69.44M |
| Jul 7 | 1,978 | 11.57M | 8,510 | 46.83M | 58.40M |
| Jul 8 | 1,402 | 8.28M | 7,235 | 39.63M | 47.91M |
| Jul 9 | 1,488 | 9.30M | 6,527 | 37.21M | 46.51M |

## Data Sources

- **HANA**: `vw_daily_nrt` parquets from S3 (hourly snapshots, CLT timezone)
- **Redshift**: `io_items` + `io_internal_orders` + `io_picking` (Cencommerce orders)
- **VTEX SQS**: Stock state change events drained from `hd-schn-ordm-giv-vtex-stock-events`
- **Mapping**: `vtex_sap_mapping.csv` from Redshift `dim_surtidos` (74,804 SKUs)
