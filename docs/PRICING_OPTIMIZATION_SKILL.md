---
name: optimize-pricing
description: Run chain pricing optimization analysis. Use when the user says "optimize pricing", "review chain pricing", "run profit score analysis", or "pricing review for <chain>".
argument-hint: "<chain> <pricing_client_name> [schedule_a|schedule_b] [max_recency_days]"
---

# Chain Pricing Optimization — Execution Guide

**Arguments:** `$ARGUMENTS` → parsed as `<chain> <pricing_client_name> [drug_type] [max_recency_days]`
- `chain` (required): e.g., `cvs`, `walgreens`, `meijer`
- `pricing_client_name` (required): e.g., `caprx`, `buzzrx_tpdt`, `meijer_gbm`
- `drug_type` (optional): `schedule_a`, `schedule_b`, or omit for both
- `max_recency_days` (optional): integer — cap adaptive recency at N days. Only NDCs with enough signal (>=3 reliable buckets) within N days are included; the rest are excluded. Implies `--adaptive-recency`. Example: `3` = only try 1d, 2d, 3d windows.

---

## Lookup Tables

### Partner → Pricing Client → Contract Mapping

| Partner (dir) | Pricing Client | Contract Name | Chains |
|---------------|---------------|---------------|--------|
| `caprx` | `caprx` | `hippo_cvs_integration_card` | CVS |
| `caprx` | `caprx` | `hippo_walgreens_integrated` | Walgreens |
| `caprx` | `caprx` | `hippo_kroger_integrated` | Kroger |
| `caprx` | `caprx` | `hippo_health_mart_atlas` | Health Mart Atlas |
| `caprx` | `caprx` | `hippo_albertsons` | Albertsons |
| `caprx` | `caprx` | `hippo_harris_teeter_integrated` | Harris Teeter |
| `caprx` | `caprx` | `hippo_meijer` | Meijer (integration) |
| `caprx` | `caprx` | `hippo_publix` | Publix |
| `buzzrx` | `buzzrx_tpdt` | `hippo_cvs_tpdt` | CVS |
| `singlecare` | `meijer_gbm` | `hippo_meijer_globalbin` | Meijer |
| `singlecare` | `wags_finder` | — | Walgreens |
| `singlecare` | `rxsense_ibo` | — | — |

**How to find the `partner` dir:** Look up `pricing_client_name` in the table above → the `Partner (dir)` column is `{partner}`.

### Competitive Environment Thresholds

| Pricing Client | Type | Min L/B per Bucket | Preferred L/B | Target Conv |
|---------------|------|---------|---------------|---------------|
| `caprx` | Integration Partner (vs Insurance) | 100 | 200+ | 0.5% |
| `rxsense_ibo` | Integration Partner (vs Insurance) | 100 | 200+ | 0.5% |
| `rxpartner` | Integration Partner (vs Insurance) | 100 | 200+ | 0.5% |
| `buzzrx_tpdt` | API Auction (vs Discount Cards) | 20 | 50+ | 40-50% |
| `meijer_gbm` | API Auction (vs Discount Cards) | 20 | 50+ | 40-50% |

**CRITICAL — Target Conversion for API Auction:**
- Default target is **40-50% conversion**, NOT maximum conversion
- At low/mid spreads, 100% conversion = underpriced (leaving money on the table) → EXTEND higher
- At ≥90% spread, 94-100% conversion is expected and fine — the drug is uncontested
- Goal: find the spread where conv ≈ target% and hold there
- If a drug shows 100% conv at current spread AND spread < 90% → it needs HIGHER spread (EXTEND)
- **Dynamic target tiers** (per-NDC, continuous formula based on best profit_score bucket spread):
  `eff_target_low = max(20, best_spread − 5)`, `eff_target_high = eff_target_low + 10`
  - Best score at 25% spread → target [20–30%] conv
  - Best score at 50% spread → target [45–55%] conv
  - Best score at 70% spread → target [65–75%] conv
  - Best score at ≥90% spread → target [85–95%] conv (uncontested, 94-100% conv is fine)

**CRITICAL — Per-Bucket Minimum:**
- The "Min L/B per Bucket" column is the minimum lookups **per individual bucket** to trust that bucket's data
- A bucket with <20 lookups (API Auction) or <100 lookups (Integration) is **noise, not signal**
- NEVER make classification decisions based on buckets below this threshold
- If no bucket meets the threshold, classify as SCAN (need more data)

**CRITICAL — Conversion Consistency (Noise Defense):**
- In auction pricing, lower spread = lower price = higher conversion
- If a bucket shows fills but NO lower-spread bucket also shows fills, the signal is likely noise (isolated spike)
- The classifier zeroes out such buckets before classification (raw data preserved in `bucket_str` comments)
- Processed low-to-high so noise can't corroborate other noise (cascading)
- Example: 1 fill at 40% with zero fills at 10-35% → noise → zeroed → GIVE UP

### Schedule → Schema Mapping

| Schedule | History Schema | Current Config Schema | Payout Types |
|----------|---------------|----------------------|-------------|
| Schedule A | `historical_data_partner_goodrx` | `partner_goodrx` | `buydown_price`, `fixed_buydown` |
| Schedule B | `historical_data_partner_buzzrx` | `partner_buzzrx` | `buydown_offset` |

---

## Schedule A: NOT YET SUPPORTED

**Schedule A optimization is under development.** If the user requests `schedule_a` or omits the drug_type (which defaults to both), **STOP and inform them:**

> Schedule A optimization is not yet ready for production use. See `docs/SCHEDULE_A_ADOPTION_PLAN.md` for the roadmap. Only `schedule_b` is currently supported.

Only proceed with `schedule_b`. Do not run the pipeline with `--schedule a`.

---

## Phase 0: Regression Tests (main agent)

Run the pricing script test suite before any analysis:
```bash
source ~/repositories/etl-testenv/bin/activate && python -m pytest .claude/scripts/tests/ -v
```
If any test fails, **STOP** and fix the issue before proceeding. Do not run optimization on broken scripts.

---

## Phase 1: Prerequisites (main agent)

1. **AWS check** — verify credentials are active
2. **DB tunnel** — port 5440+ (5439 reserved for user), source db_env.sh, verify with SELECT 1. See `.claude/scripts/setup_tunnel.sh` for commands.
3. **Assess worthiness** — check `docs/CHAIN_OPTIMIZATION_WORTHINESS.md` for "Not Ready" chains

---

## Pipeline Architecture

When optimizing **both schedules**, the full pipeline runs in parallel per schedule:

```
  Phase 2A → Phase 3A → Phase 3bA → Phase 3cA ─┐
                                                ├─→ Phase 4 (main agent: review & generate)
  Phase 2B → Phase 3B → Phase 3bB → Phase 3cB ─┘
```

Each schedule's pipeline is **completely independent** — different override CSVs,
different bucket/summary/daily data files, different eligible NDC sets, no shared
state. This means Schedule A's classifier can start as soon as its data collection
finishes, without waiting for Schedule B's data collection.

**Execution:** Launch **two Bash subagents in parallel**, each running the combined
pipeline script:
```bash
bash .claude/scripts/run_schedule_pipeline.sh \
  --chain <chain> --pricing-client <pc> --schedule a|b \
  --override-csv <path_to_override_csv> \
  --min-lb <20|100> --default-min <N> --default-max <N> --default-step <N> \
  --chain-label "<label>" [--max-recency-days N] [--cost-type nadac] \
  [--session-id <unique_id>]
```

**Session caching (`--session-id`):** When provided, Phase 2 SQL outputs are cached
to `/tmp/` with the session ID. Re-runs within the same session skip Redshift entirely
and reuse cached data. Use any unique-per-session value (e.g., short UUID or timestamp).
Always pass `--session-id` when iterating on classifier logic within a single session.
Cache is validated by an args fingerprint — if collector parameters change (e.g.,
`--min-lb`, `--cost-type`, `--max-recency-days`), the cache is automatically invalidated.

The script chains Phases 2+3+3b+3c sequentially within a single subagent, failing
fast if any step fails. Phase 3c runs `merge_overrides.py` to produce the final
`{PREFIX}_merged.csv`. All output goes to `/tmp/{chain}_{pc}_sched_{a|b}_*`.
The main agent waits for both subagents, then proceeds to Phase 4 with all files.

When optimizing a **single schedule**, launch one subagent with the same script.

**Multi-client chains**: For chains with multiple pricing clients (e.g., CVS runs both
`caprx` and `buzzrx_tpdt`), all four pipelines (2 clients × 2 schedules) can run
simultaneously — launch four subagents in parallel.

---

## Phases 2+3+3b+3c: Per-Schedule Pipeline (SUBAGENT — Bash type)

The per-schedule pipeline script (`.claude/scripts/run_schedule_pipeline.sh`)
runs four phases in sequence. Each phase is documented below for reference,
but in practice they run as a single subagent invocation.

### Phase 2: Data Collection

Runs `.claude/scripts/pricing_data_collector.py` internally.

**Parameter lookup (forwarded via pipeline script args):**
- `{schedule_letter}`: `a` or `b` (from drug_type)
- `--all-time-start`: defaults to 42 days ago
- `--min-lb`: 20 (API Auction) or 100 (Integration) — used by adaptive recency
- `--max-recency-days N`: pass when `max_recency_days` argument is provided (e.g., `--max-recency-days 3`)
- `--cost-type`: `nadac` (default), `wac`, `awp`, or `auto` (COALESCE WAC>NADAC>AWP). Look up chain's Primary Cost in table above. Used for NDC grouping metadata.

#### Adaptive Recency Windowing (default: on)

Adaptive recency is **on by default**. It uses the shortest recent data window per NDC
that still has sufficient statistical signal (>=3 reliable buckets at >=min_lb lookups each).
Use `--no-adaptive-recency` to disable.

**How it works:** For each NDC, tries progressively wider windows (Fibonacci steps:
1, 2, 3, 5, 8, 13, 21, 34, 55, 89 days) and stops at the first window with enough signal.
RPU boundaries are always respected — data before a payout change is never used.
For grouped NDCs, recency is applied to the **pooled** daily data (requires `--daily-buckets`
in the classifier).

**Additional flags (forwarded via pipeline script):**
- `--recency-steps 1,2,3,5,8,13,21,34,55,89`: customize window steps (comma-separated days)
- `--min-lb 20`: min lookups per bucket to count as reliable (default: 20)
- `--max-recency-days N`: cap adaptive window at N days. NDCs that don't reach >=3 reliable buckets within N days are **excluded** from output (not fallen back to all-time). Use this to focus analysis on high-volume drugs with fresh signal.
- `--no-adaptive-recency`: disable recency, use all-time data for all NDCs
- `--deployment-dates PATH`: deployment date file (`ndc|quantity|deployment_date`). When provided, `analysis_start` is floored at the deployment date per NDC — data collected under the old formula is excluded. Also consumed by Phase 3b (quantity analyzer) for per-quantity deployment floors.

#### Deployment-Aware Windowing

When an override CSV exists, the pipeline computes per-NDC and per-(NDC, quantity)
deployment dates using `git blame` on `origin/release` (pre-Phase 2 step). Each CSV
line is blamed to the exact commit that last modified it — no commit-count limitation,
every NDC gets a date. For NDCs with multiple quantity rows, the NDC-level date is the
earliest across all quantities (`min()`). Output format: `ndc|quantity|deployment_date`.
This ensures only post-deployment lookups are analyzed — data collected under the old
formula is noise for evaluating the current range.

**9am UTC rule:** Airflow deploys daily ~9am UTC. Commits before 9am on day D →
deployment date = D. Commits after 9am → deployment date = D+1.

**Windowing priority:** `analysis_start = max(rpu_start, deployment_date)`. Both
RPU boundaries and deployment dates are respected — the later of the two wins.

**Output:** Same `_buckets_raw.txt` and `_summary_raw.txt` format.
Additionally writes `_recency_diagnostics.txt` showing per-NDC window selection details.
Always writes `_ndc_metadata.txt` (ndc|drug_name|rpu_dollars|cost_per_unit) for NDC grouping.
Always writes `_daily_buckets.txt` (ndc|drug_name|date|spread|lookups|fills) for classifier group recency.

### Phase 3: Classification

Runs `.claude/scripts/classification_history.py` (git history check) and
`.claude/scripts/auction_classifier.py` internally.

**Key parameters from Competitive Environment table (forwarded via pipeline script):**
- `--min-lb`: 20 (API Auction) or 100 (Integration)
- `--default-min`, `--default-max`: chain's current default spread
- `--chain-label`: human-readable label for CSV header
- `--default-step`: current default step size
- `--ndc-metadata PATH`: path to `_ndc_metadata.txt` from data collector. Enables NDC grouping — pools lookups across NDCs with same drug_name and similar RPU/cost to reach statistical significance.
- `--grouping-tolerance N`: max % difference for RPU and cost matching (default: 5). Only relevant when `--ndc-metadata` is provided.
- `--daily-buckets PATH`: path to `_daily_buckets.txt` from data collector. Enables recency windowing for grouped NDCs — pools daily data then applies Fibonacci-step recency to find shortest recent window with enough signal.
- `--groups-output PATH`: path to write group definitions JSON for per-quantity analyzer.
- `--recency-steps 1,2,3,5,8,13,21,34,55,89`: customize window steps (comma-separated days)
- `--override-csv PATH`: path to the actual override CSV in the repo. Enables **history-aware dual-window analysis** — uses git history to find when each NDC's spread range last changed, then compares pre-change vs post-change conversion data. Requires `--daily-buckets`.

#### History-Aware Dual-Window Analysis

When `--override-csv` is provided, the classifier enriches each result with
historical context by splitting recon data at the config change boundary:

- **Short window** (post-change): data since the spread range was last modified.
  This is what the classifier already uses for its primary classification.
- **Long window** (pre-change): data from before the change — shows what conversion
  looked like under the OLD spread pricing.

The comparison detects:
- **Regime changes**: conversion at the same spread passes a two-proportion
  z-test (z > 1.96, α=5%) between windows. This means the market changed
  (new PBM bidders, seasonal demand). The z-score and direction appear in
  CSV comments: e.g., "regime shift at 40%: conv 10%→60% (up, z=4.06)".
- **Tested territory**: the long window may cover spreads that the classifier
  wants to SHIFT into — if prior data shows 0% conversion there, the shift
  is riskier than the classifier alone would indicate.
- **Prior round performance**: shows what score/conversion the old range achieved,
  giving context for whether the current range is an improvement.

### Phase 3b: Per-Quantity Analysis

Runs `.claude/scripts/quantity_analyzer.py` internally.

Per-quantity analysis detects divergent
competitive dynamics across quantities (e.g., qty=30 converts at 80% while qty=90
converts at 10%). When per-quantity data reaches statistical significance, it is
**preferred over the aggregate** — merge qualifying per-quantity rows into the final
override CSV, replacing the single aggregate row.

**Excludes `fixed_buydown` NDCs** — payout depends on days supply which is missing
from recon data.

**Qualification criteria (all must hold):**
1. Quantity diversity: >=2 distinct quantities AND each candidate quantity meets the volume threshold: **>= 15% of total lookups OR >= 200 absolute lookups** (and the remaining pool also meets the >= 15% threshold)
2. Per-quantity viability (either path):
   a. Standard: quantity has >=3 reliable buckets (same min_lb as general classification)
   b. Unanimous signal bypass: >=2 core buckets with >=min_lb total lookups AND each bucket >= 40% of min_lb (e.g., 8 L/B when min_lb=20), and all bucket conversions within 5pp delta (max - min <= 5). Clear signal without needing 3 buckets — works for any level (100%, 0%, 50%, etc.)

**Deployment-aware per-quantity windowing:**
When `--deployment-dates` is provided, per-quantity `analysis_start` is floored at
the deployment date for that specific (NDC, quantity). Cascading lookup: `(ndc, qty)`
→ `(ndc, '')` aggregate → NDC-level → no floor. This prevents per-quantity analysis
from including data collected under a previous formula for that quantity.

**History-aware enrichment for per-quantity results:**
When `--override-csv` is provided, the quantity analyzer enriches each qualifying
per-quantity result with dual-window history context (same approach as Phase 3).
Uses **cascading lookup**: prefers `(ndc, quantity)` history when the CSV historically
had quantity-specific rows for that NDC+qty, falls back to `(ndc, '')` aggregate
history otherwise. This means every per-quantity result gets history if any history
exists for that NDC — no hard separation.

**Output design — no overlap between per-quantity and aggregated:**
- Qualifying quantities get quantity-specific CSV rows
- The aggregated fallback row (blank quantity) EXCLUDES qualifying quantities
- Non-qualifying quantities fall through to the blank-quantity fallback
- If only 1 quantity exists total, per-quantity adds nothing — use aggregated

**Interaction with NDC grouping:** Per-quantity runs on BOTH individual NDCs and
grouped NDCs. The correct flow is: group first (Phase 3), then per-quantity
(Phase 3b) on both paths:
- **Individual NDCs:** high-volume NDCs analyzed for per-quantity on their own data
- **Grouped NDCs:** when `--groups-json` is provided, daily rows from all NDCs
  in each group are pooled, then per-quantity analysis runs on the pooled data.
  This combines two enrichment techniques: grouping (more rows from similar NDCs)
  and per-quantity (quantity-specific spreads).

NDC grouping is a data enrichment step — it compresses time by pooling similar
NDCs. Example: 5 NDCs with ~50 lookups/day each become ~250 lookups/day pooled,
making per-quantity analysis viable at shorter recency windows.

**Merge** per-quantity rows into the merged override file (handled automatically by `merge_overrides.py`). All qualifying quantities get per-quantity rows — no divergence gate needed. Replace the single aggregate row with:
- Per-quantity rows for each qualifying quantity (specific spread ranges)
- A fallback row (blank quantity) covering all OTHER quantities — this row uses the general/aggregate classification so unmatched quantities still get sensible spreads

---

## Phase 4: Review & Generate (main agent)

1. **Check existing overrides** for the chain
2. **Present classifier summary** to user (counts per status, eligible/skipped totals)
3. After user confirms: **copy the merged CSV** (`{PREFIX}_merged.csv`) to override path — this combines draft + per-quantity rows + preserved human/locked rows
4. **CSV filename rule**: first segment before `-` MUST equal `pricing_client_name` (enforced by `validate_filename()`)
   - Pattern: `{pricing_client}-{contract_name}-pos_{drug_type}_overrides.csv`
   - Look up `contract_name` from the mapping table above
5. **CSV formatting** is handled by `auction_classifier.py` — generates header blocks, section separators, date-prefixed comments
6. **Update config**: add `pos_manual_overrides_path` to `ingr_cost_rules` (just the filename, code builds full path)

---

## Phase 4b: PR Description — Before/After Table

The PR **Demo** section MUST include an enriched before → after table.
Use the table generated by `auction_classifier.py --pr-table-output /tmp/<chain>_pr_table.md`:

- Table format: `| Drug | Action | Old Range | New Range | Rationale | Score | Lookups | Bucket Data |`
- Rationale column comes BEFORE metadata (Score/Lookups/Conv%) so reviewers read the reasoning first
- Groups duplicate NDCs automatically (e.g., "Mounjaro (6 NDCs)")
- Uses `<details><summary>` for large tables (>20 rows)
- Includes all actions: EXTEND, NARROW DOWN, SCAN, GIVE UP
- Mark new NDCs (not in previous CSV) as "NEW (default X-Y%)"
- Copy the generated table directly into the PR Demo section

---

## Phase 4c: PR Self-Review (non-CSV changes only)

Check if the PR includes non-CSV changes:
```bash
git diff --name-only origin/main..HEAD | grep -v '\.csv$'
```

If ALL changes are CSV files → **skip this phase**.

If any non-CSV files changed → launch a **Sonnet subagent** (Task tool with `model="sonnet"`):

Prompt the subagent with:
- The PR number (from `gh pr create` output)
- Instruction to run `gh pr diff <number>` and review all non-CSV changes
- Review checklist:
  - **Code correctness**: logic bugs, off-by-one errors, edge cases
  - **Doc accuracy**: do doc changes match the code changes?
  - **Algorithm consistency**: do classifier/analyzer changes match documented domain rules?
  - **No regressions**: do changes break existing behavior for other chains/schedules?
  - **validate_csv.py**: if validation logic changed, are edge cases handled?
- Return a short review summary (findings or "LGTM")

The main agent reads the subagent's review output and:
- If issues found → fix them, push, update PR
- If LGTM → proceed to Phase 5

---

## Phase 5: Validation

**Create the PR FIRST, then run validation.** Dry run takes 2-5 minutes.

Launch a **Bash subagent** running:
```bash
source ~/repositories/etl-testenv/bin/activate && \
bash .claude/scripts/dry_run_validator.sh <partner> <csv_path>
```

The script runs parse check + dry run and prints a 5-line PASSED/FAILED summary.
Only the summary needs to be reviewed — full output is in `/tmp/dry_run_output_*.log`.

---

## Domain Rules (ALWAYS enforce)

1. **NDC format**: Always NDC11 (11 digits, zero-padded string)
2. **Drug names**: Look up via `medications_v1.drug_ndcs_with_rxnorm` — never hardcode
3. **Per-bucket trust**: NEVER classify based on buckets below the min L/B threshold. A bucket with 4 lookups is noise.
4. **Target conversion**: API Auction = 40-50% conv target. 100% conv = underpriced. DO NOT optimize for max conversion.
5. **Phase progression**: SCAN (~3 wide buckets) → NARROW DOWN (precision) → LOCK (never skip). **LOCK criteria**: tested spread range ≤5% AND ≥200 total lookups. At that point the classifier has enough data in a tight range — no further exploration needed.

    | Phase | Status | Step | Range | Purpose |
    |-------|--------|------|-------|---------|
    | 1a (Wide Scan) | SCAN | 2 (10% buckets) | Wide (0-100%) | Find approximate peak region |
    | 1b (Focused Scan) | SCAN | 1 (5% buckets) | Narrowed around 1a region | Confirm peak at 5% granularity |
    | 2 (Narrow Down) | NARROW DOWN | 0.4 (2% buckets) | ±5% around SCAN peak | Refine to 2% granularity |
    | 3 (Ultra-Precision) | NARROW DOWN | 0.2 (1% buckets) | ±2% around Phase 2 peak | Refine to 1% granularity (only if ≥50 L/B) |
    | Final | LOCK | — | Single value | Set fixed spread at confirmed optimal |
6. **Score tracking**: `score=X` = max bucket profit score. Formula: `(spread/100) × conv%`. NOT overall conversion.
7. **Git history**: Always check before proposing new ranges (handled in Phase 3 subagent)
8. **CSV wiring**: Always verify `pos_manual_overrides_path` in config after creating CSV
9. **Scope**: Only brand rebate-eligible drugs (in `brand_channel_ndc_config_v2`). POS spread CSV overrides do NOT affect high-volume generics (statins, PPIs, BP meds, etc.) — those go through the default POS formula. Before investing time in a chain review, verify the chain has meaningful brand rebate volume, not just high overall POS traffic driven by generics.
10. **Filename validation**: First segment before `-` must equal `pricing_client_name`
11. **RPU threshold**: ≥10% change = windowed analysis; <10% = all-time data (handled by data collector script)
11b. **Deployment date floor**: When override CSV exists, `analysis_start` is floored at the NDC's production deployment date (from `origin/release` git history). Pre-deployment data collected under the old formula is excluded. Priority: `max(rpu_start, deployment_date)`.
12. **History table sentinel**: `valid_to = '2300.01.01T00:00:00'` means currently valid (NOT NULL, uses dots not dashes)
13. **CSV comments**: formatted by `auction_classifier.py` — include analysis date, drug name, status, score, lookups, since date, per-bucket data, and auto-generated rationale (see rule #22)
14. **No CSV entry for insufficient data**: If a drug has <3 reliable buckets, skip it (stays on default, no CSV row)
15. **Spread ceiling**: Hard cap at **99%** (spread should never reach 100%). Above ~90% spread the drug is likely uncontested — high conversion (94-100%) is expected and acceptable at those spreads. The dynamic target formula naturally scales: at spread 95% the target is [90-99%] conv.
16. **GIVE UP requires full range**: If only tested 30%+, that's SCAN (scan lower), NOT GIVE UP. GIVE UP ranges start at 1%, never 0% (0% spread = giving away for free).
16b. **Weighted strategy (advanced, high-volume only)**: For EXTENDs where you want finer control, a weighted strategy can split traffic between known-good spreads and exploration. Only practical on high-volume drugs with **≥500 total lookups AND ≥50 lookups/day** — splitting traffic means each bucket accumulates lookups slower, so drugs below this threshold will take months to reach min L/B per bucket. For lower-volume drugs, use a plain uniform range instead (e.g., `spread_min=20, spread_max=40, step=1`) — similar exploratory effect but every lookup counts.
    - Add `strategy,probability_weights,probability_values` columns to CSV (Schedule A already has them; add to Schedule B header if needed)
    - Example: `strategy=weighted, probability_weights=0.5-0.1-0.1-0.1-0.1-0.1, probability_values=25-30-35-40-45-50` → 50% at safe 25%, 10% each exploring 30-50%
    - Dash-separated, weights must sum to 1.0
    - The `pos_formulas.py` code generates `{probability weights [...], values: [...]}` formula syntax
    - See Schedule A examples: Dexcom G6, Dulera, Repatha SureClick
    - **For lower volume**: use a plain uniform range instead (e.g., `spread_min=20, spread_max=40, step=1`) — similar exploratory effect but every lookup counts toward min L/B
17. **Minimum 3 reliable buckets to classify**: Never NARROW DOWN, EXTEND, or GIVE UP based on fewer than 3 statistically significant buckets. With 1-2 buckets you don't have enough data points to know the shape of the conversion curve — classify as SCAN instead.
18. **Dynamic target conversion by spread** (API Auction only):
    The classifier uses a **continuous formula** based on where the best profit_score bucket sits:
    `eff_target_low = max(20, best_spread − 5)`, `eff_target_high = eff_target_low + 10`
    This is a 10pp window. At best_spread=50 the target is [45,55]. At best_spread=20 the target is [20,30].
    - Best score at 25% spread → target [20–30%] conv
    - Best score at 50% spread → target [45–55%] conv (standard)
    - Best score at 70% spread → target [65–75%] conv
    - Best score at ≥90% spread → target [85–95%] conv (uncontested — high conversion expected and fine)
    Applied per-NDC. The floor of 20% prevents targeting unrealistically low conversions.
19. **Projection-based scan decision** — when all buckets are below target, decide
    scan-vs-narrow using projected scores at lower spreads:
    - At any spread X, the max plausible conv = `max(30, X − 5)` (from dynamic target formula:
      ~30% conv at ≤30% spread, ~45% at 50%, ~65% at 70%). Projected score = X × conv.
    - If best projected score at untested lower spreads > current actual best → **SCAN lower**
      (e.g., Zepbound: projected 50%×55%=2750 > actual 90%×3%=281 → scan)
    - If current best beats all projections → **NARROW DOWN** around it
      (e.g., FreeStyle Libre: actual 80%×36%=2868 > best proj 1700 → narrow)
    - This prevents wasting rounds scanning lower when the drug already performs
      optimally at high spreads with moderate conversion.
20. **SCAN uses ~3 buckets** — wide enough to see the curve shape, few enough for
    fast signal. Range is computed by `compute_scan_range(lowest)` — centered on
    the lowest tested bucket, one step below and one step above, aligned to the
    step grid. Step = round(range/10), min 1. This keeps buckets anchored to
    tested data and avoids scanning too-low spreads where projected conversion
    would far exceed target (wasted buckets). Progressive iteration: if SCAN
    finds 0% conv, next round centers on the new lowest (e.g., 30→15,30,45 →
    if 15% still 0, next: 1,21,2).
21. **NDC grouping** has two tiers, both opt-in via `--ndc-metadata`:
    - **Tier 1 — DDID grouping** (pre-classification, correctness): NDCs with the same DDID (Drug Descriptor ID, from RxNorm `drug_ndcs_with_rxnorm`) AND identical RPU AND identical cost are treated as one unit of analysis. Data is pooled first, classified once, and the same result applies to every NDC in the group. Always on by default when metadata is provided. Handles cases like Gemtesa 30-ct vs 90-ct — same clinical product, same economics. Conversion differences between same-DDID NDCs reflect PBM bidding behavior (some PBMs can't do random bidding but can win one NDC and lose another), not drug-level demand.
    - **Tier 2 — RPU/cost tolerance grouping** (post-classification, statistical power): pools lookups across NDCs sharing the same drug_name where RPU and cost per unit are within tolerance (default 5%). Used to improve recency windows for drugs like Zepbound/Wegovy/Mounjaro where different strengths individually lack data. Only applies to NDCs that don't meet the 3-bucket threshold individually and weren't already classified via DDID grouping. When `--daily-buckets` is provided, grouped NDCs use adaptive recency on the pooled daily data (same Fibonacci steps as individual recency).
    - The `ddid` field is the 6th pipe-delimited column in `ndc_metadata.txt`: `ndc|drug_name|rpu|cost|payout_type|ddid`
22. **Rationale is MANDATORY in CSV comments** — every override row ends with
    `| Rationale: <explanation>` auto-generated by the classifier from its internal
    `notes` field. The rationale explains WHY this classification was chosen in plain
    language (e.g., "projected 50%×55%=2750 > actual 90%×3%=281; scan 60-90%" or
    "all reliable >target conv; push from 80%; step=3→4b@50L/B"). This is generated
    automatically via `format_comment()` — no manual step required. The `+NL@outlier`
    prefix before `bucket_str` indicates N lookups in extreme spread buckets (>200%
    or <0%) that were excluded from classification.
23. **Straddle range bias**: When NARROW DOWN detects a straddle (adjacent buckets above and below target conv), the test range biases toward the **upper (unknown) end** — ~1/3 below straddle, ~2/3 above (-7/+13 from center). This avoids stacking too many buckets at "guaranteed win" spreads which inflates overall conversion and signals aggression to competitors. Worst-case overall conv ~60%, then recalibrate next round.
24. **SHIFT HIGHER / SHIFT LOWER** — actionable edge-detection statuses when all buckets
    are below target and the current best beats tier-model projections (Case D2c).
    The classifier slides the test window by one step in the shift direction:
    - **SHIFT HIGHER**: best profit score is at the highest tested bucket (high edge).
      Peak not found — higher spreads could yield better score at same flat conv.
      Slides window up: drops the lowest bucket, adds one step above the highest.
    - **SHIFT LOWER**: best profit score is at the lowest tested bucket (low edge), and
      linear regression on actual (spread, conv) data projects a capped score that beats
      current best at one step below. Projected conv is capped at the effective target
      for that spread — never projects aggressive pricing.
      Slides window down: drops the highest bucket, adds one step below the lowest.
    - Both generate CSV rows with proper `spread_min`/`spread_max`/`spread_step` for
      the shifted range, using `calc_step` to validate bucket count and lookups/bucket.
25. **Handling market changes** — when reviewing results and a previous peak no longer behaves as expected:

    **Conversion dropped** (competitors lowered prices):

    | Situation | Action |
    |-----------|--------|
    | NARROW DOWN peak now shows 0% conv | Demote to SCAN — scan lower spreads (old peak is too high) |
    | LOCK value now shows 0% conv | Demote to SCAN — scan below old lock value |
    | SCAN peak shifted to different region | Update range to follow the new peak |
    | All spreads show 0% conv | Demote to GIVE UP |

    Key principle: when a peak stops converting, competitors likely lowered their prices. The new optimal spread is almost certainly *lower* — scan backwards, not around the old region.

    **Conversion spiked** (competitors raised prices):

    | Pricing client type | Action |
    |---------------------|--------|
    | API Auction (`buzzrx_tpdt`, `meijer_gbm`) | Demote to SCAN — scan upward from current spread to find new higher optimum |
    | Integration Partner (`caprx`, `rxsense_ibo`) | Keep current spread locked — higher conversion is a good outcome. Re-evaluate next cycle. |

26. **EXTEND_LOW_CONV (dragging foot)**: When an NDC has been tested at low spreads (lowest bucket ≤10%) but shows 3-10% aggregated conversion across all reliable buckets with ≥2 reliable buckets in the 0-30% range, the data is statistically unreliable (at 20 L/B with 5% conv, each bucket has ~1 fill — coin-flip noise). Instead of classifying on noisy data, the classifier spreads wide (1-80%) and waits for early signals. Uses **raw fills** (before noise defense) for signal detection — noise defense can zero scattered fills that still represent a faint signal worth exploring. Lifecycle:
    - **<3% agg conv (floor)**: GIVE UP — too dead for wide exploration. Caught directly in pre-dispatch (never falls through to SHIFT HIGHER).
    - **3-10% agg conv**: EXTEND_LOW_CONV — spread wide and collect data.
    - **>10% agg conv**: exits to normal classification (NARROW DOWN, SCAN, etc.)
    - **Confirmed dead**: if all raw fills are zero → skipped entirely, GIVE UP handles it
    - **Not triggered**: if lowest tested spread is >10%, SCAN lower first — don't declare drag foot until the bottom has been tested
27. **Score-first routing**: The dynamic target (conv ≈ spread ± 5%) is a *ceiling*, not a floor. When all buckets have non-zero conversion, the best profit score is at the highest tested spread, there's no straddle, and the best bucket's conv ≥ 35%, the classifier bypasses target-based routing and optimizes directly for score via `_status_all_above` → EXTEND. This prevents high-spread drugs (e.g., Alvesco: 30%:100%cv, 60%:64%cv, 90%:71%cv) from being misrouted to gap-scanning because the dynamic target (85%) makes 71% look "below target."
28. **Inverted pattern REVIEW**: If ANY higher-spread bucket converts ≥ 2× the lowest-spread bucket's conv AND has ≥ 20% conv AND ≥ 5 fills, the pattern is economically suspicious — lower spread should convert better in a normal auction. Flagged as REVIEW for human investigation (quantity mix, data artifact, etc.). Catches right-wing inversions (25%,30%,80%), middle-peak patterns (15%,80%,70%), and middle-spike cliffs (15%,60%,20%). Also fires when the lowest bucket has 0% conv — any higher bucket with ≥20% conv and ≥5 fills is anomalous. Does NOT fire when no higher bucket reaches the 2× threshold (e.g., 60%,20%,60% — the dip pattern).
29. **Conversion consistency (noise defense)**: In auction pricing, lower spread = lower price = higher conversion. A bucket showing fills while all lower-spread buckets show zero is likely an isolated noise spike (e.g., 1 random fill at 40% with 0 fills at 10-35%). The classifier processes buckets low-to-high and zeroes out any bucket whose fills lack corroboration from at least one non-noise lower-spread bucket. Cascading prevents noise from corroborating other noise. Raw fill counts are preserved in `bucket_str` comments for transparency. This applies to both aggregate and per-quantity classification.
30. **Range alignment (status-agnostic)**: After every range computation, the range is aligned so (spread_max − spread_min) is evenly divisible by bucket_width (step × 5). Without this, the last bucket gets a truncated range and fewer lookups. **At ceiling (spread_max ≥ 95)**: nudge spread_min down. **At floor (spread_min ≤ 5)**: nudge spread_max up. **Middle**: nudge spread_min down (prefer wider range). Clamps to [1, 99]. Examples: `55,99,3` → `54,99,3` (ceiling); `1,80,3` → `1,91,3` (floor); `10,30,2` → already aligned. Applies to ALL statuses: EXTEND, NARROW DOWN, SCAN, SHIFT HIGHER/LOWER, REVIEW, EXTEND_LOW_CONV. GIVE UP is exempt (hardcoded 1-6 step 1, already aligned).
31. **Old range in merge comments**: `merge_overrides.py` appends `| old range: {min},{max},{step}` to comments when the NDC had a prior entry in the existing override CSV. Matches by `(ndc, quantity)` with fallback: per-qty rows show the old aggregate range if switching from aggregate→per-qty, and vice versa.

---

## Related References

- `docs/EXPLORATION_EXPLOITATION.md` — Theory doc: why we target ~50% conversion (cooperative game theory / race to the top), multi-armed bandit framing, dual-window UCB analogy, regime change detection rationale
- `docs/COST_CALCULATION_REFERENCE.md` — Cost components (AWP/WAC/NADAC), hippo cost formula, profit spread calculation
- `docs/POS_FORMULA_GENERATION_REFERENCE.md` — How pos_formulas.py generates Random/weighted formulas from spread configs
- `docs/CHAIN_OPTIMIZATION_WORTHINESS.md` — Chain tier list and volume thresholds (referenced in Phase 1)
