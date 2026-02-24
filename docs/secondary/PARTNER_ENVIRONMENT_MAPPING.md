# Partner Environment Mapping

## How to Determine Environment Type

Check `src/partners/sources/clients.csv` → `network_reimbursement_id` column:

- **NRID starts with "INT"** → Integration Partner (vs Insurance)
- **Otherwise** → API Auction (vs Other PBMs)

---

## Integration Partner (vs Insurance)

**Characteristics:**
- Competing against insurance copays
- Low conversion (0.5-5%)
- Statistical threshold: 100+ lookups/bucket
- Strategy: Maximize profit_score

---

## API Auction (vs Other PBMs)

**Characteristics:**
- Competing against other PBM cards
- Higher conversion (5-30%)
- Statistical threshold: 20+ lookups/bucket
- Strategy: Collaborative (reduction function by spread)
