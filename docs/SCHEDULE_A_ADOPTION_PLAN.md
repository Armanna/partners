# Schedule A Pricing Optimization — Adoption Plan

## Why Schedule A Is Different

Schedule B drugs (buydown_offset) have straightforward economics: spread × conversion = profit score. Schedule A (buydown_price / fixed_buydown) has three additional complications:

1. **Eligibility restrictions** — Age, gender restrictions limit which patients can fill. State is known at lookup time (via NPI → state correlation), but age/gender are not. Hippo prices at bid, but ineligible fills get reversed (draining money). A drug with 60% eligibility and 25% fill-attempt conversion is actually converting at ~40% among eligible patients.

2. **Higher conversion expectations** — Schedule A drugs are buydown-based and fill more aggressively. Dynamic target: `max(50, spread+5)` capped at 100% (vs Schedule B: `max(30, spread-5)`).

3. **Generic drugs don't need optimization** — For branded drugs with WAC/AWP basis, if `multi_source_code = 'Y'` the drug went generic and almost always costs less than buydown_price. No point optimizing spread on these. Note: NADAC-basis drugs may still be classified as branded even when medispan says generic (or vice versa) — per-contract classification is needed for accuracy (see P5).

4. **Fixed buydown economics** — Drugs like Lantus have payout that scales by days_supply, not quantity. Diminishing returns at high spreads since payout is flat.

5. **Fill attempts ≠ actual fills** — Reconciliation counts fill_attempts (claims-authz calls). Reversals inflate apparent conversion at aggressive spreads. Need fill-through rate for accurate economics.

## What's Done (PR #748)

### P1: Data-Driven Eligibility Rate
- New SQL in `pricing_data_collector.py` queries `brand_claims_router.brands_claim_eligibility` vs `reconciliation.brand_rebatable_fill_attempts_correlation_ids`
- Sliding window: uses the 100 most recent non-reversed fill attempts per NDC (constant `ELIGIBILITY_MIN_FILL_ATTEMPTS` in `constants.py`)
- Intra-day reversals (`valid_from::date = valid_to::date`) excluded from fill_attempts denominator for eligibility calculation. Note: these reversals should still participate in future reversal rate economics (P4) — they are only excluded from eligibility, not from fill-through analysis.
- Eligibility discount applied in `classification.py`: `eff_target *= eligibility_pct / 100`, floored at `ELIGIBILITY_TARGET_FLOOR` (10%)
- Restriction columns (`min_age`, `max_age`, `gender_restriction`, `state_restriction`) pulled from `brand_channel_ndc_config_v2_history`

### P2: Restriction Annotations
- `comment_formatter.py` shows: `Eligibility: 62% (age:18+, gender:F) -> target 31-37%`
- `eff_target_low` and `eff_target_high` included in classification result dict

### Schedule-Aware Dynamic Targets
- `--schedule a` flag: `max(50, spread+5)` capped at 100
- `--schedule b` flag: `max(30, spread-5)`, capped at 100. If profit spread >= 90%, target conversion is 100%.
- No flag: legacy `max(20, spread-5)` preserved

### Grouping
- Restricted (<95% eligibility) and unrestricted NDCs no longer pooled together

## Next Steps (Priority Order)

### P2.5: Generic NDC Filtering (High Impact, Low Effort)

**Problem:** Generic drugs (`multi_source_code = 'Y'`) cost less than buydown_price ~99% of the time. Running eligibility analysis and pricing optimization on these is wasted effort.

**Approach:**
- Add `multi_source_code` to the eligible NDCs SQL (already joins `medications_v1.drug_ndcs_with_rxnorm` for drug_name)
- Flag `is_generic` in metadata output
- Skip generic NDCs in eligibility query and classifier
- Log count of skipped generics in summary

**Key nuance:** Some NDCs are brand for one contract but generic for another. `multi_source_code = 'Y'` from medispan is a good-enough proxy for now. Per-contract classification requires `ndcs_v2_history` and contract-specific `historic_constants.py` — deferred.

**Tables:**
- Current: `medications_v1.drug_ndcs_with_rxnorm` — column `multi_source_code` ('Y' = generic)
- Historical: `historical_data_formulas_costs_bin_<bin>.ndcs_v2_history` — has `multi_source_code` and possibly `nadac_is_generic` (SCD2 with `valid_from`/`valid_to`)

### P2.6: Cost-Change-Based Eligibility Window (Medium Impact, Medium Effort)

**Problem:** The 100-fill sliding window ignores cost changes. If drug costs shift >10%, old eligibility data reflects different economics.

**Approach:**
```
eligibility_window_start = max(last_cost_change_>10%, current_date - 180 days)
```

Use `historical_data_bin_019901.ndc_costs_v2_history` (SCD2) with `LAG()` to find the most recent >10% change in the relevant cost column.

**Complication:** Which cost column (NADAC vs WAC vs AWP) depends on the contract's basis source:
- NADAC-based contracts: check NADAC changes
- WAC-based: check WAC
- AWP-based: check AWP

This info comes from `historic_constants.py` per contract. For a first pass, could just check all three (any >10% change resets the window).

### P3: Fixed Buydown Awareness (Medium Impact, Low Effort)

**Problem:** `fixed_buydown` NDCs (e.g., Lantus) have flat payout regardless of quantity. Aggressive EXTEND has diminishing returns since payout doesn't scale with higher spreads.

**Approach:**
- `payout_type` already in `_ClassifyCtx` metadata
- Apply lower target conversion multiplier for `fixed_buydown`
- Suppress aggressive EXTEND — cap at current best rather than pushing higher
- Flag in notes: "fixed_buydown: flat payout, volume-optimized"

### P4: Fill-Through Rate (High Impact, High Effort — Deferred)

**Problem:** Fill attempts that pass eligibility can still get reversed (14-day lag). Conversion data overestimates actual fills.

**Score calculation shift:** Currently score = `profit_spread × conversion_rate` where `conversion_rate = fill_attempts / lookups`. The corrected formula should be `profit_spread × (non-reversed fills / lookups)`. However, reversal data is only reliable 14+ days after the fill, so this analysis runs on a delayed basis.

**Effective conversion = fill_attempt_conversion × eligibility_rate × fill_through_rate**

**Approach:** Per-NDC reversal rate from `reporting.claims` SCD2 pattern (note: intra-day reversals excluded from eligibility in P1 should still participate here):
- `valid_to > current_timestamp` = fill stuck
- `valid_to < current_timestamp` = reversed
- Data must be 14+ days old for reversals to settle
- Reversal rate is time-agnostic — use wide window

**Alternative:** Use `financial_claims.profit_report` (actual `profit_to_hippo_usd` per claim) to build profit-per-spread-bucket directly, replacing conversion proxy with real P&L signals. The debit monitor already does this for drug-level aggregation.

**Cross-check: etl-data-monitor draining drugs.** The debit monitor's actual profit calculations for draining drugs provide a ground-truth signal that can validate both eligibility rates and fill-through rates. If a drug shows high eligibility but negative actual profit, the fill-through rate is likely the missing factor. Incorporating draining drug P&L as a cross-check would strengthen both the eligibility (P1) and fill-rate (P4) optimization threads — drugs flagged as draining should correlate with low eligibility or high reversal rates.

### P5: Per-Contract Brand/Generic Classification (Low Priority)

**Problem:** `multi_source_code = 'Y'` is a global flag. Some NDCs are brand-priced for one contract but generic for another.

**Source:** Contract `historic_constants.py` files define `generic_flag` expressions using combinations of:
- `nadac_classification_for_rate_setting` (from nadac.csv, 'G'/'B')
- `multi_source_code` (from medispan, 'Y'/'O')
- `nadac_is_generic` (derived flag)

**Approach:** Query `ndcs_v2_history` for the contract-relevant classification. Only matters for edge cases where a drug is brand for the target contract but generic globally.

## Caveats

### Fixed Price Overrides — Exclude from Optimization

Some NDCs have **manual fixed price overrides** defined in a separate CSV alongside the profit-spread overrides:
`buzzrx_tpdt-cvs_tpdt-pos_schedule_a_fixed_price_overrides.csv`

These NDCs use hardcoded `min_fixed_price_usd` / `max_fixed_price_usd` rather than spread-based formulas. They should **not be targeted** by the pricing optimization workflow by default, since their pricing is manually managed and spread changes won't affect them.

Example: Premarin (`00046110281`) has fixed prices of $135 (qty 30) and $370 (qty 90) — yet generates the longest formula string in the export (7,766 chars) due to quantity + state + DS branching.

Check for fixed price override files when selecting optimization candidates:
```
src/partners/sources/program_overrides/{partner}/pricing_clients/{pc}/{pc}-{chain}-pos_schedule_{a|b}_fixed_price_overrides.csv
```

## Pilot Strategy

**First run:** Pick a chain/pricing_client pair that already has Schedule B optimization running, so we can compare behavior. Candidates:
- `caprx` / `cvs` or `walgreens_integrated` — high volume, well-understood
- `meijer_gbm` — lower volume but already has extensive optimization history

**Validation steps:**
1. Run data collector with `--schedule a` — check eligible NDC count, eligibility rates, generic count
2. Run classifier — compare classifications against current GIVE UP drugs in debit monitor
3. Drugs flagged as "draining" in debit monitor should get conservative classification (low eligibility → discounted targets)
4. Unrestricted brand drugs should classify similarly to Schedule B behavior (just higher targets)
5. Generic drugs should be filtered out entirely

## Key Tables Reference

| Table | Use | Key Columns |
|-------|-----|-------------|
| `brand_claims_router.brands_claim_eligibility` | Eligibility rate numerator | `ndc`, `correlation_id`, `brand_claim_eligibility`, `valid_from`, `valid_to` |
| `reconciliation.brand_rebatable_fill_attempts_correlation_ids` | Eligibility rate denominator | `ndc`, `correlation_id`, `pricing_client_name`, `chain`, `drug_type`, `date` |
| `medications_v1.drug_ndcs_with_rxnorm` | Generic flag (current) | `ndc`, `multi_source_code`, `drug_descriptor_id` |
| `historical_data_formulas_costs_bin_<bin>.ndcs_v2_history` | Generic flag (historical) | `ndc`, `multi_source_code`, `nadac_is_generic`, `valid_from`, `valid_to` |
| `historical_data_bin_019901.ndc_costs_v2_history` | Cost changes | `ndc`, `nadac`, `wac`, `awp`, `valid_from`, `valid_to` |
| `reporting.claims` | Fill-through rate (P4) | `correlation_id`, `valid_to`, `fill_status` |
| `financial_claims.profit_report` | P&L per claim (P4 alt) | `correlation_id`, `profit_to_hippo_usd` |
