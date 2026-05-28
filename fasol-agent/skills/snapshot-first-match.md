# `snapshot_first_match` — when a condition first / last held

> **Sub-skill of [Fasol Agent](../SKILL.md).** For full per-snapshot detail
> use [snapshot-history](snapshot-history.md). For window extremes use
> [snapshot-agg](snapshot-agg.md). For coin-wide discovery use
> [snapshot-scan](snapshot-scan.md).

`POST /snapshot/coin/{coin_address}/first_match` — finds the **first** (or
**last**) snapshot in the window where ALL given filters hold.

Requires `read_coins`. Tier: `medium`. **At least one filter required.**

## Request

```bash
curl -s -X POST \
  -H "Authorization: Bearer $FASOL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "from": "2026-04-27T00:00:00Z",
    "to":   "2026-04-27T12:00:00Z",
    "direction": "first",
    "filters": { "min_mc": 1000000, "max_drop_from_ath_p": 50 }
  }' \
  "$FASOL_API_BASE_URL/snapshot/coin/<COIN>/first_match"
```

| Param | Values | Default |
|---|---|---|
| `from` / `to` | ISO 8601 | `from = now − 1h`, `to = now` |
| `direction` | `"first"` / `"last"` | `"first"` |
| `filters` | object — same key set as [snapshot-scan](snapshot-scan.md) | required |

Window cap: **24 hours**.

## Response

```json
{
  "match": {
    "ts": "2026-04-27T03:42:00Z",
    "price_usd": "0.0000234",
    "mc": "1023000",
    "liq": "12300",
    "holders": 567,
    "drop_from_ath_p": "12.4"
  }
}
```

`match: null` when nothing qualifies in the window.

## Use cases

- "When did BONK first cross $1M market cap?" → `direction: "first"`,
  `filters: { min_mc: 1000000 }`
- "Last time liq was above $50k" → `direction: "last"`,
  `filters: { min_liq: 50000 }`
- Position-management hooks: "first moment dev_hold_p dropped below 5% AND
  liq > 10k" → `filters: { max_dev_hold_p: 5, min_liq: 10000 }`
