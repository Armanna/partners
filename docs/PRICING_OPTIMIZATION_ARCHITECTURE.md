# Pricing Optimization — Strategic Architecture

## Current State

`/optimize-pricing` lives as a Claude Code skill (~6K lines in `.claude/scripts/`) inside etl-partners. It produces override CSVs committed to git via PRs, which the ETL pipeline reads to generate POS formulas. This works well for manual brand optimization at current scale (~1,500 lines across ~10 CSVs).

## Upcoming Transitions

| Transition | Timeline | Impact |
|-----------|----------|--------|
| Schedule A adoption | Now | Same architecture, extend skill to handle Schedule A |
| Brand automation | ~1 month | Schedule B polished first, then automated; Schedule A follows ~1 month later |
| Generics optimization | ~2-3 months | Same optimization framework, different lever (target price instead of profit spreads) |

The end state is NOT "classify and LOCK" — it's continuous adaptive optimization that responds to market changes ("learn to dance").

## Recommended Architecture: New Repo + etl-partners as Thin Consumer

### Why a New Repo

1. **etl-partners = config management**, not optimization engine. A 6K-line multi-armed bandit algorithm doesn't belong in a config pipeline.
2. **Automation needs its own DAG.** etl-partners runs 2x/day for formula generation. The optimization cycle (analyze conversion data → classify → adjust spreads) is a separate concern on its own schedule.
3. **Generics will scale beyond git.** Thousands of NDCs can't be reviewed as CSV diffs — need a data pipeline, not git artifacts.
4. **Follows established pattern.** UI auctions already split into `etl-auction-pricing` (compute) + `etl-auction-formulas` (export). POS should follow the same separation.
5. **"Learn to dance" requires state management.** Continuous re-evaluation, regime detection, and adaptive windows need proper state tables, not git blame archaeology.

### Unifying Insight: Both Brands and Generics Move Ingredient Cost

Both optimization types ultimately move the same thing: **ingredient cost in the formula DSL**. The difference is the internal lever:

- **Brands**: profit spread % → formula builder translates to ingredient cost (cost is stable, spread is the control variable)
- **Generics**: target price → translates to ingredient cost (costs fluctuate, so we target price directly)

The new repo's **output interface is unified**: per-NDC ingredient cost parameters that etl-partners consumes identically regardless of drug type.

### The Split

```
etl-pricing-optimization (NEW REPO — per-NDC optimization)
  ├── Reads: reconciliation conversion data, NDC configs, cost tables
  ├── Algorithm: classification engine (ported from .claude/scripts/)
  ├── Brands: lever = profit spread (spread_min/max/step per NDC)
  ├── Generics: lever = target price (per NDC, translates to ingr cost params)
  ├── Unified output: per-NDC ingredient cost parameters → S3
  ├── State: SCD2 tables for classification history, round tracking
  └── DAG: daily or more frequent, per chain/schedule

etl-partners (KEEPS — general rules + formula generation)
  ├── Baseline/default configs (e.g., default_spread_override: 30-90-6)
  ├── POSProcessor reads per-NDC rules from S3 (automated)
  ├── Manual CSV overrides still supported (take precedence)
  ├── Formula generation unchanged (etl-lib pos_formulas.py)
  └── Exports final formulas to Redshift as today

Claude Code skill (EVOLVES — R&D + monitoring)
  ├── Phase 1-2: Full optimization workflow (as today)
  ├── Phase 3+: R&D lab for testing new algorithm versions
  ├── Monitoring: query automated decisions, flag anomalies
  └── Override: write manual exceptions when automation is wrong
```

### State Management (Replacing Git History)

Currently git provides deployment dates (`git blame`), round counting (commit history), and regime detection (dual-window pre/post-change analysis). The new repo needs proper queryable state:

| Table | Purpose |
|-------|---------|
| `classification_state` | NDC, chain, schedule, current_status, spread_range, classified_at, deployed_at |
| `classification_history` (SCD2) | Full history with valid_from/valid_to |
| `optimization_rounds` | Round metadata, window used, bucket counts, scores |

## Phased Rollout

### Phase 1: Complete Schedule A (Now → Month 1)

No architecture changes. Keep skill, keep CSVs in git.

- Finish Schedule A adaptation in current `.claude/scripts/` framework.
- Algorithm still evolving — rapid iteration is key.

### Phase 2: Stabilize + Extract Algorithm (Month 1-2)

Schedule B is "polished," ready for automation.

- **Extract core algorithm** from `.claude/scripts/` into a proper Python package.
  - Option A: New package in etl-lib (co-located with pos_formulas.py)
  - Option B: Package inside the new repo (self-contained)
  - **Recommendation: Option B** — the optimization algorithm is specific to this pipeline, not a general ETL utility.
- **Design state tables** — schema for classification_state, classification_history, optimization_rounds.
- **Design the DAG** — schedule, chain/schedule parallelism, failure handling.

### Phase 3: Launch etl-pricing-optimization for Brands (Month 2-3)

New repo with:
- Ported classification engine as proper Python modules with tests
- Airflow DAG running daily (or per-chain schedule)
- S3 export of per-NDC spread rules
- Redshift state tables for classification tracking

Modify etl-partners POSProcessor:
- Add `automated_overrides_source` config option (S3 path)
- Load automated rules, merge with manual CSV overrides
- **Precedence: manual CSV > automated S3** (human intent always wins)

Claude Code skill shifts to monitoring: "show me what the automated system decided for CVS Schedule B this morning."

**"Learn to dance" logic**: the automated pipeline doesn't classify-and-stop. It continuously:
- Detects regime shifts (conversion drops after competitor changes)
- Widens exploration when conditions change
- Tightens when converging
- Never truly "LOCKs" — LOCK becomes "stable, check weekly not daily"

### Phase 4: Add Generics (Month 3-4)

Same repo, same optimization framework — classification engine, adaptive windowing, multi-armed bandit all transfer.

**Different lever: target price instead of profit spread.**

| Property | Brands | Generics |
|----------|--------|----------|
| Cost stability | NADAC/WAC relatively stable | Costs fluctuate frequently |
| Control variable | Profit spread % | Target price |
| Why | Cost stays put, you move the margin | Spread would silently drift as cost changes underneath |

Open question: could we express generics as spread around PBM standard profit (allowing <0% and >100%)? This would reuse more spread infrastructure and keep a unified output shape. Needs data analysis to determine if PBM standard profit is stable enough to anchor on.

**Unified output**: regardless of lever, the new repo produces per-NDC ingredient cost parameters that etl-partners consumes identically.

**Scale**: thousands of NDCs per chain — validates the S3/Redshift decision over git CSVs.

### Phase 5: Mature + Expand (Month 4+)

- Expand to all chains (follow `CHAIN_OPTIMIZATION_WORTHINESS.md` tiers).
- Add caprx generics once proven on buzzrx.
- Claude Code skill becomes primarily a diagnostic/override tool.
- Consider: should formula generation also move here? (Probably not — etl-lib's `pos_formulas.py` is well-suited as a shared library.)

## Key Design Decisions to Make Later

| Decision | Options | When |
|----------|---------|------|
| Generics lever | Target price directly vs spread-around-PBM-standard-profit | Phase 4 — needs data analysis on generic cost volatility |
| DAG frequency | Daily (natural for brands) vs different cadence for generics | Phase 3-4 |
| Auto-deploy vs approval gate | Direct update with monitoring vs Slack/UI approval step | Phase 3 |
| Algorithm package location | etl-lib (shared) vs new repo (self-contained) | Phase 2 |

## What Stays in etl-partners

- Pricing config structure (Python dicts with contract/schedule/program hierarchy)
- Formula generation (via etl-lib's `process_pos_formulas()`)
- Cash card and brand rebates processing (unrelated to optimization)
- Manual CSV overrides (escape hatch for human intent)
- The `pos_manual_overrides_path` pattern — augmented with `automated_overrides_source`

## What Moves to etl-pricing-optimization

- Classification algorithm (~6K lines from `.claude/scripts/`)
- Data collection (Redshift queries for conversion data)
- Per-quantity analysis
- NDC grouping logic
- History/regime detection
- Stale NDC cleanup
- State management (replaces git history)
