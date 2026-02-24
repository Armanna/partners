# Chain Optimization Worthiness Assessment

## Minimum Thresholds for POS Spread Optimization

Thresholds depend on the competitive environment (determined by `pricing_client_name`, not chain name).

### Integration Partner (`caprx`) — vs Insurance
| Criterion | Threshold | Why |
|-----------|-----------|-----|
| Total lookups (30d) | ≥10,000 | Need 100+ lookups/bucket; 10k supports ~11 buckets at 20% step |
| LOCK or EXTEND NDCs | ≥1 | Proves there are NDCs where we can improve margin |
| Estimated monthly impact | ≥$100/mo | Below this, engineering time exceeds ROI |
| Skip rate | <80% | High skip rates mean most NDCs are already optimal or unpriceable |

### API Auction (`buzzrx_tpdt`) — vs Other Discount Cards
| Criterion | Threshold | Why |
|-----------|-----------|-----|
| Total lookups (30d) | ≥5,000 | Only need 20+ lookups/bucket (5x lower threshold) |
| LOCK or EXTEND NDCs | ≥1 | Proves there are NDCs where we can improve margin |
| Estimated monthly impact | ≥$100/mo | Below this, engineering time exceeds ROI |
| Skip rate | <80% | High skip rates mean most NDCs are already optimal or unpriceable |

**Skip rate defined:** % of eligible NDCs where every spread bucket has fewer lookups than
the minimum threshold (20 L/B for API Auction, 100 L/B for Integration Partner), so no
bucket is reliable enough to classify. A high skip rate means there is insufficient
competitive data to make evidence-based spread decisions for most drugs.

## Chain Tier List (assessed 2026-02-09)

> **Important**: A single chain can appear in multiple pricing environments with very different worthiness.
> Always assess worthiness **per pricing_client_name**, not per chain name alone.

### High Priority — Active Optimization
| Chain | Pricing Client | Schedule | LOCKs | EXTENDs | Lookups (30d) | Status |
|-------|---------------|----------|-------|---------|---------------|--------|
| CVS | caprx | A | 5 | 2 | 86k | Override CSVs deployed |
| CVS | caprx | B | 7 | 2 | 408k | Override CSVs deployed |
| Walgreens | caprx | A | 0 | 1 | 69k | Override CSVs deployed |
| Walgreens | caprx | B | 1 | 0 | 323k | Override CSVs deployed |

### In Progress — Per-NDC Overrides Deployed
| Chain | Pricing Client | Schedule | NDCs w/ overrides | Lookups (42d) | Status |
|-------|---------------|----------|-------------------|---------------|--------|
| Meijer | meijer_gbm | B | 59 | 8.5k | Round 2 deployed: 5 NARROW DOWN, 6 EXTEND, 7 SCAN, 41 on default. 1,288 eligible, 429 with data. |

### Not Ready — Insufficient Volume or Signal
| Chain | Pricing Client | Schedule | Brand Rebate Lookups (30d) | Fills | NDCs | Total Lookups (all drugs) | Blocker |
|-------|---------------|----------|---------------------------|-------|------|--------------------------|---------|
| Kroger | caprx | A | 1.7k | 1 | 69 | 851k | Passes 10k threshold but 0% conv — no NDCs to optimize |
| Kroger | caprx | B | 10.9k | 3 | 378 | (incl above) | Passes 10k threshold but 0% conv across all top NDCs |
| Health Mart Atlas | caprx | A | 336 | 30 | 28 | 40k | Very low brand rebate volume |
| Health Mart Atlas | caprx | B | 994 | 32 | 153 | (incl above) | Very low brand rebate volume |
| Harris Teeter | caprx | A | 2.4k | 0 | — | ~6k | No fills, very low volume |
| Harris Teeter | caprx | B | 3.7k | 0 | — | (incl above) | No fills, very low volume |
| Albertsons | caprx | A/B | ~3k | 0 | — | ~3k | All high-volume NDCs are generics (not rebate-eligible) |
| Publix | caprx | A/B | — | — | — | 79 | Essentially zero traffic |
| Meijer | caprx | A/B | — | — | — | 28 | Essentially zero traffic (integration — different from meijer_gbm) |

## When to Revisit Low-Volume Chains

- **Frequency**: Every ~3 months
- **Trigger**: When brand rebate lookups cross 10k/month (integration) or 5k/month (API auction)
- **Quick check query**: Run the profit score analysis from `docs/PRICING_OPTIMIZATION_SKILL.md` (Step 5) and look for any LOCK/EXTEND NDCs
- **Key insight**: Volume alone is insufficient — the competitive environment matters. A chain with 100k lookups but 0 LOCKs/EXTENDs is not worth optimizing until the competitive landscape changes.

## Why Some High-Volume Chains Have No Optimization Potential

1. **All integration partner chains** (pricing_client_name = `caprx`) compete against insurance, not other discount cards
2. This means thresholds are high: 100+ lookups/bucket, 0.5% conversion rate
3. At these thresholds, most NDCs fall into SCAN (keep current spread) or GIVE UP (not enough signal)
4. Only NDCs with genuinely mispriced spreads will show LOCK/EXTEND — and many chains simply don't have enough of these to matter

**Example: Kroger** — 851k total lookups (30d) but only 12.5k for brand rebate-eligible drugs. The overwhelming majority of Kroger traffic is generics that go through the default POS formula and are unaffected by spread override CSVs. Total volume is misleading — always check brand rebate-eligible volume specifically.
