# `snapshot_history` — historical trajectory of one coin

> **Sub-skill of [Fasol Agent](../SKILL.md).** Same data source
> (`db.coin_snapshot`) as [snapshot-agg](snapshot-agg.md),
> [snapshot-first-match](snapshot-first-match.md), and
> [snapshot-scan](snapshot-scan.md). For multi-day backtests with stats use
> [alert-simulate](alert-simulate.md).

`GET /snapshot/coin/{coin_address}/history` — time-series of all the
`coin_snapshot` rows for one coin in a window. Server auto-buckets so you
always get ≤ 1000 rows; the bucket size widens with the window.

Requires `read_coins`. Tier: `medium`.

## Request

```bash
curl -s -G \
  -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "from=2026-04-27T08:00:00Z" \
  --data-urlencode "to=2026-04-27T09:00:00Z" \
  "$FASOL_API_BASE_URL/snapshot/coin/<COIN>/history"
```

Defaults: `to = now`, `from = now − 1h`. Window is capped at **24 hours** —
wider requests return `400 window_exceeds_24h_cap`.

## Response

```json
{
  "coin_address": "...",
  "from": "2026-04-27T08:00:00Z",
  "to": "2026-04-27T09:00:00Z",
  "bucket_seconds": 60,
  "rows": [
    {
      "ts": "2026-04-27T08:00:00Z",
      "price_usd": "0.0000123",
      "mc": "12345.67",
      "liq": "8000.00",
      "holders": 234,
      "vol_5m": "1234.56",
      "top_10_p": "12.3",
      "drop_from_ath_p": "8.4"
    }
    // ... up to ~1000 rows
  ]
}
```

## Notes

- **Phantom-gate** (`tx_count > 1`) is applied — deploy-second BC-seed
  snapshots with stale prices are filtered out. What you see here matches
  what the alerts pipeline would have evaluated.
- For min/max / extreme questions over a window, use the cheaper
  [snapshot-agg](snapshot-agg.md) instead of post-processing this output.
- For "when did condition X first hold" use
  [snapshot-first-match](snapshot-first-match.md).
