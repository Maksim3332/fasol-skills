# `snapshot_agg` — min/max/extremes for one coin over a window

> **Sub-skill of [Fasol Agent](../SKILL.md).** For the full time-series see
> [snapshot-history](snapshot-history.md). For "when did X first happen" see
> [snapshot-first-match](snapshot-first-match.md).

`GET /snapshot/coin/{coin_address}/agg` — single-row aggregate (min/max/count)
of `db.coin_snapshot` for one coin over a time window. Very cheap; reach for
this instead of `snapshot_history` when you only need extremes.

Requires `read_coins`. Tier: `medium`.

## Request

```bash
curl -s -G \
  -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "from=2026-04-27T00:00:00Z" \
  --data-urlencode "to=2026-04-27T12:00:00Z" \
  "$FASOL_API_BASE_URL/snapshot/coin/<COIN>/agg"
```

Defaults: `to = now`, `from = now − 1h`. Window cap: **24 hours**.

## Response

```json
{
  "snapshot_count": 432,
  "min_price": "0.0000111",
  "max_price": "0.0000234",
  "min_mc":    "11100",
  "max_mc":    "23400",
  "min_liq":   "5500",
  "max_liq":   "12300",
  "max_holders": 567,
  "max_vol_5m":  "12345.67",
  "min_drop_from_ath_p": "0",
  "max_drop_from_ath_p": "67.8"
}
```

## When to use

- "What was BONK's ATH market cap in the last 6 hours?" → `max_mc`
- "Did liquidity ever drop below $5k?" → check `min_liq`
- "How many snapshots do we have for this coin in window?" → `snapshot_count`
  (zero = coin wasn't trading enough to clear the phantom-gate)

For full per-snapshot detail use [snapshot-history](snapshot-history.md). For
conditional first/last matching see
[snapshot-first-match](snapshot-first-match.md).
