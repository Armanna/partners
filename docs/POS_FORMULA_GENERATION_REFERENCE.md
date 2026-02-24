# POS Formula Generation Architecture Reference

## Key Architecture Components

### 1. Bucket Generation Logic

**Location:** `hippo/transform/pos_formulas.py`

**Two-Phase Process:**
1. **Fundamental Matrix Generation** (`_generate_percentage_buckets`)
   - Generates ALL possible spread values from spread_min to spread_max with 1% step
   - Example: For spread 50-80%, generates [50%, 51%, 52%, ..., 80%] (31 buckets)
   - Creates comprehensive coverage for all possible profit spreads

2. **Interval Filtering** (`_filter_dataset_applicable_buckets`)
   - Selects only buckets at configured intervals
   - Formula: `actual_step = spread_step × spread_step_percentage`
   - Default spread_step_percentage = 5
   - Example: step=3, step_pct=5 → actual=15% → keeps [50, 65, 80]

**Critical Formula:**
```
actual_bucket_step = spread_step × spread_step_percentage
```

### 2. Profit Score Optimization vs Simple Spreads

**Two Different Code Paths:**

**Simple Default Spreads (meijer_gbm):**
- Uses `default_spread_override` config
- Generates simple Random spreads: `NADAC {Random min: -50 max: -80 step: -15}`
- All NDCs get same spread configuration
- Affected by bucket generation logic

**Per-NDC Profit Score Optimization (caprx, buzzrx_tpdt, rxsense_ibo):**
- Uses profit scoring algorithm
- Generates optimized spreads per-NDC: `NADAC {Random min: -53.74 max: -33.969 step: 2.479}`
- Each NDC has custom spread based on profitability analysis
- **Bypasses simple bucket logic** - not affected by bucket bug
- Overrides manual_per_ndc_overrides configurations

**How to identify:**
- Simple spreads: Round numbers aligned with config (e.g., -50, -65, -80)
- Optimized spreads: Decimal values (e.g., -53.74, -33.969, 2.479)

### 3. Data Type Handling

**NDC Format:**
- All NDCs stored as 11-digit strings with leading zeros: `'00002245780'`
- S3Downloader uses `dtype=object` to preserve leading zeros (line 74 in hippo/sources/s3.py)
- Critical for merge operations - dtype mismatch causes silent failures

**Always normalize NDCs:**
```python
df['ndc'] = df['ndc'].astype(str).str.zfill(11)
```

### 4. Common Debugging Patterns

**For Empty Output Issues:**
1. Add debug logging at each merge point:
   ```python
   log.info(f'Before merge: df1 has {len(df1)} rows, df2 has {len(df2)} rows')
   log.info(f'After merge: {len(merged)} rows')
   ```

2. Check merge indicator:
   ```python
   merged = df1.merge(df2, on='ndc', how='outer', indicator=True)
   log.info(f'Matches: {(merged["_merge"] == "both").sum()}')
   log.info(f'Left only: {(merged["_merge"] == "left_only").sum()}')
   ```

3. Check data types before merges:
   ```python
   log.info(f'df1 NDC dtype: {df1["ndc"].dtype}')
   log.info(f'df2 NDC dtype: {df2["ndc"].dtype}')
   ```

**For "Silently Passing" Tasks:**
- Beware of fixes that make empty DataFrames have correct columns - they mask root causes
- Always prefer fixing root cause over working around symptoms
- Example: PR #597 made empty DataFrame pass but didn't fix bucket generation bug

### 5. Critical Merge Points in process_pos_formulas()

**Line 291-296:** Routing filter
- Filters brand_ndc_config by pricing_client_name
- Can filter out ALL NDCs if pricing_client not in brand_channel_distribution_config

**Line 360:** First cost merge
- Merges price_config with ndc_costs_v2
- Outer join with indicator
- Logs "Costs for NDC not found" warnings

**Line 430:** Second cost merge
- Inner join: `target_ndcs.merge(ndc_costs_v2)`
- Stricter - filters out NDCs without costs

**Line 541-546:** all_drugs_all_vars creation
- Merges bucket matrix with cost data

**Line 604-608:** Final merge with price_config
- **Inner join** - critical filter point
- If all_drugs_all_vars has no matching NDCs with price_config, result is empty

**Line 610:** Bucket filter
- `_filter_dataset_applicable_buckets()` - select configured intervals
- BUG WAS HERE: filter logic didn't account for spread ranges or step_percentage

### 6. Testing Manual Overrides

**Don't just check if NDC is in output - check if FORMULA is correct:**

```python
# Bad check
if ndc in output_df['ndc'].values:
    print("NDC found")  # Could have wrong formula!

# Good check
actual_formula = output_df[output_df['ndc'] == ndc]['brands'].iloc[0]
# Extract Random values and compare with expected config
```

**Manual overrides can be overridden by:**
- Profit score optimization (takes precedence)
- Brand routing restrictions
- Missing cost data
- Bucket generation bugs (now fixed)

### 7. Spread Configuration Patterns

**Common Patterns:**
- **Fixed spread:** spread_max=None, spread_step=None (wags_finder, publix_finder)
- **Wide range:** min=30, max=90, step=6 → [30, 60, 90] (buzzrx_tpdt default, rxsense_ibo)
- **Medium range:** min=50, max=80, step=3 → [50, 65, 80] (meijer_gbm)
- **Narrow range:** min=10, max=50, step=1 → [10, 15, 20, ..., 50] (caprx configs)
- **Ultra-granular:** min=1, max=21, step=1, step_pct=1 → every 1% (Zepbound/Wegovy)

**Alignment Check:**
- step=1 → hits all values (0,1,2,3,...)
- step=2 → hits even values (0,2,4,6,...)
- step=3 → hits multiples of 3 (0,3,6,9,...) → **skips 50, 80**
- step=5 → hits multiples of 5 (0,5,10,15,...) → hits common values

**Only fails when:** spread_min/spread_max not divisible by spread_step AND not in bucket list

## Key Files

- **Formula logic:** `etl-lib/libs/py3-etl/hippo/transform/pos_formulas.py`
- **POS processor:** `etl-partners/src/partners/sources/program_overrides/processors/pos.py`
- **Orchestrator:** `etl-partners/src/partners/sources/program_overrides/orchestrator.py`
- **S3Downloader:** `etl-lib/libs/py3-etl/hippo/sources/s3.py` (dtype=object)

## Lessons Learned

1. **Treat root causes, not symptoms** - PR #597 example
2. **Profit score optimization overrides manual configs** - by design
3. **Step value determines bucket granularity** - not just filter intervals
4. **Always verify formulas, not just NDC presence** - optimization can change spreads
5. **Debug logging at merge points is critical** - reveals where data is lost
