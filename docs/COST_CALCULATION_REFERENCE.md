# Cost Calculation Reference

**Date:** 2026-02-05
**Analysis:** POS profit spread optimization (any chain)

## Cost Components (All values in USD unless specified)

### 1. AWP/WAC/NADAC (Unit Cost)
- **Stored in:** `historical_data_bin_019901.ndc_costs_v2_history`
- **Columns:** `ndc, awp, wac, nadac, gpi_nadac, valid_from, valid_to`
- **Units:** CENTS per unit (per tablet/capsule/mL)
- **Conversion:** `cost_usd = cost_cents / 100`
- **Time-varying** — must join on lookup timestamp: `pl.timestamp::date >= nc.valid_from::date AND pl.timestamp::date < nc.valid_to::date`
- **NOT in prices_lookup** — prices_lookup has NO awp/wac columns, you must JOIN this table

### 2. Hippo Cost (chain-dependent)
```sql
-- General formula:
hippo_cost_usd = COALESCE(
    (nadac_usd * (1 + nadac_coeff)) * quantity,
    (wac_usd * (1 + wac_coeff)) * quantity,
    (awp_usd * (1 + awp_coeff)) * quantity
) + disp_fee
```

**Coefficients vary by chain — don't hardcode, look them up from source:**

- **Contract rates** (base, coefficient, disp fee, fallbacks):
  ```
  etl-pbm-hippo/src/contracts/<contract_name>/v1/programs/regular/historic_constants.py
  ```

### 3. Patient Paid
```sql
patient_paid_usd = (price_cents - flat_sales_tax_cents) / 100
```
- From `prices_api_pipe.prices_lookup.price` (in cents)
- Minus any flat sales tax
- Convert to dollars

### 4. Buydown Price
Depends on payout_type from the config history table:

**Schedule A** config: `historical_data_partner_goodrx.brand_channel_ndc_config_v2_history`

**Type: buydown_price** (scales with quantity)
```sql
buydown_price_usd = (payout_price_usd / default_quantity) * actual_quantity
```

**Type: fixed_buydown** (does NOT scale)
```sql
buydown_price_usd = payout_price_usd
```

**Schedule B** config: `historical_data_partner_buzzrx.brand_channel_ndc_config_v2_history`

**Type: buydown_offset** (scales with quantity, but different profit formula)

Payout scales by quantity the same way as `buydown_price` (RPU × actual_quantity stays
constant, total payout grows with quantity):
```sql
offset_usd = (payout_price_usd / default_quantity) * actual_quantity
```

But the profit spread formula differs — the effective floor price for the formula is
`hippo_cost - offset`, so the denominator is just the offset, not `hippo_cost - payout`:
```
profit_spread_pct = ((patient_paid - (hippo_cost - offset)) / offset) * 100
```

Config lives in `partner_buzzrx` schema (vs `partner_goodrx` for Schedule A).
Schedule is determined entirely by `payout_type`: Schedule B ↔ `buydown_offset`,
Schedule A ↔ `buydown_price` / `fixed_buydown`.

Where:
- `payout_price_usd = payout_price_cents / 100`
- `payout_price_cents` from config table

### 5. Profit Spread Percentage
```sql
profit_spread_pct = ((patient_paid_usd - buydown_price_usd) /
                     (hippo_cost_usd - buydown_price_usd)) * 100
```

## Example

To calculate profit spread for a specific drug, look up the chain's contract rates
from `etl-pbm-hippo` (see § 2 above), then apply:

```
hippo_cost_usd = (unit_cost * (1 + coefficient)) * quantity + disp_fee
buydown_price_usd = payout_price (or scaled by quantity if buydown_price type)
profit_spread_pct = ((patient_paid - buydown_price) / (hippo_cost - buydown_price)) * 100
```

## Common Pitfalls

1. **AWP/WAC are in cents** — Must divide by 100. They are per-unit, so multiply by quantity AFTER converting to dollars.
2. **Price is in cents** — `prices_api_pipe.prices_lookup.price` is in cents, divide by 100
3. **Buydown price scaling** — Check payout_type: `buydown_price` and `buydown_offset` scale with quantity (`payout_price / default_quantity * actual_quantity`), `fixed_buydown` does NOT scale
4. **Dispensing fee** — Chain-specific (e.g. Walgreens=$2.00, Albertsons=$10.00, CVS=$18.30/$27.30) — check chain_pricing_config. Also in cents in the contract config.
5. **Negative spreads** — Can occur when patient_paid < buydown_price (losing money!)
6. **AWP/WAC are NOT in prices_lookup** — Must JOIN `historical_data_bin_019901.ndc_costs_v2_history` on NDC + timestamp range
7. **`default_quantity` is varchar** in some history tables — cast to float explicitly: `bcnc.default_quantity::float`
8. **`default_quantity` changes alongside payout** — When computing RPU (payout_price / default_quantity), ALWAYS use the quantity from the SAME history row. Do NOT use the current config's default_quantity for historical RPU. Example: Eliquis payout went 7500→20 but qty went 60→1, so RPU = $1.25→$0.20 (not $75→$0.20)
8. **NDCs can switch payout types** — e.g. Trintellix switched from `buydown_offset` (Sched B) to `buydown_price` (Sched A) on 2026-01-08. Always check current payout_type, not historical.

## Contract Rate Lookup

Contract-specific formulas (AWP/NADAC/WAC percentages, dispensing fees, margins) live in etl-pbm-hippo:
```
etl-pbm-hippo/src/contracts/<contract_name>/v1/programs/regular/historic_constants.py
```
Look for `base` (AWP/NADAC/WAC/UNC), `value` (discount %), and `DISP_FEE`.

## Validation Queries

**Check AWP for a drug:**
```sql
SELECT ndc, awp, valid_from, valid_to
FROM historical_data_bin_019901.ndc_costs_v2_history
WHERE ndc = '59467070027'
AND valid_from <= '2026-01-15'
AND valid_to > '2026-01-15';
```

**Check payout config:**
```sql
SELECT ndc, payout_type, payout_price, default_quantity,
       brand_channel_name, valid_from, valid_to
FROM historical_data_partner_goodrx.brand_channel_ndc_config_v2_history
WHERE ndc = '59467070027'
AND payout_type IN ('buydown_price', 'fixed_buydown');
```
