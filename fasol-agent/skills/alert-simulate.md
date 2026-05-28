# `alert_simulate` — multi-day alert backtest

> **Sub-skill of [Fasol Agent](../SKILL.md).** Auth, rate-limit tiers, and the
> `AlertConfig` shape used by the live engine are described in the parent.
> This page only covers the simulation endpoint.

`POST /alert/simulate` — replay an alert config against the last 1–5 days of
`db.coin_snapshot` to see which coins would have triggered it, and what each
one did afterward (ATH multiplier, 1.5×/2×/5×/10× hit rates).

It's literally the same backtest the UI's alert modal runs (both surfaces share
`simulateAlert(...)` in `fasol_services`), so results match across UI ↔ agent
to the row, for any given config.

Requires `read_alerts`. **Heavy tier** (5 rpm) — each call scans 5–15 GiB
from ClickHouse. Don't loop; it's a report, not a feed.

---

## When to use this — vs. `snapshot_scan`

| Question | Tool |
|---|---|
| Would this filter have caught any coin in the last N days, and how did those coins perform? | **`alert_simulate`** (this) |
| What was the state of one specific coin during the last hour? | `snapshot_history` |
| Which coins right now match a filter set? | `snapshot_scan` (point-in-time) |

`alert_simulate` is the only surface that gives you `reached_2x` /
`reached_5x` ATH rollup over multiple days. The snapshot tools are spot
queries.

## Request

```bash
curl -s -X POST \
  -H "Authorization: Bearer $FASOL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "launchpads": ["pf"],
    "booleanFilters": ["with_socials", "only_migrated"],
    "minMaxFilters": {
      "mc":              [50000, 500000],
      "liq":             [10000, null],
      "dev_hold_p":      [null, 5],
      "drop_from_ath_p": [null, 50]
    }
  }' \
  "$FASOL_API_BASE_URL/alert/simulate?days=3&include_statistics=true"
```

### Query params

| Param | Values | Default | Notes |
|---|---|---|---|
| `days` | `1` … `5` | `1` | Lookback window in days. Anything else → 400. Wider windows aren't supported — `db.coin_snapshot` is partitioned for short scans. |
| `include_statistics` | `true` / `false` | `true` | Pass `false` to skip the 1.5×/2×/5×/10× rollup; response is then just `{ data: { total_matching_coins } }`. |
| `alert_id` | number | — | If set, the response marks rows whose underlying coin was deployed _before_ the alert's last filter change with `excluded_by_config_cutoff: true` (those would not have fired live — the engine skips them). |

### Body shape

Same `AlertConfig` the UI's `/trading_bot/alert` upsert takes:

```jsonc
{
  // Launchpad list. "mayhem" is virtual — matches is_mayhem_mode=1 coins
  // regardless of their real launchpad. At least one entry required.
  "launchpads": ["pf", "rl", "letsbonk", "mayhem"],

  // Boolean filters. Each is a CH column = 1 condition.
  "booleanFilters": ["with_socials", "only_migrated", "dex_paid",
                     "is_cashback_coin", "dev_last_migrated"],

  // Min/max ranges. Same key set as snapshot_scan's min_/max_ filters
  // (both surfaces share a builder). Each value is [min, max] — pass null
  // to omit a side.
  "minMaxFilters": {
    "mc":                [50000, 500000],
    "liq":               [10000, null],
    "vol_5m":            [1000, null],
    "holders_count":     [100, null],
    "dev_hold_p":        [null, 5],
    "drop_from_ath_p":   [null, 50]
  }
}
```

## Response

```jsonc
{
  "data": {
    "coins": [
      {
        "coin_address": "<base58>",
        "symbol": "BONK",
        "supply": "100000000000",
        "image": "...",
        "alert_price_usd": 0.0000123,        // price at first trigger
        "ath_price_usd": 0.0000281,          // ATH AFTER first trigger
        "alerted_at": 1747567890000,         // ms epoch
        "ath_at": 1747571490000,
        "excluded_by_config_cutoff": false
      }
      // ... up to 300 rows
    ],
    "total_matching_coins": 142,
    "config_updated_at_ms": 1747000000000,   // null when no alert_id passed
    "statistics": {
      "reached_1_5x": 87,  "reached_1_5x_percent": 61.27,
      "reached_2x":   52,  "reached_2x_percent":   36.62,
      "reached_5x":   14,  "reached_5x_percent":    9.86,
      "reached_10x":   3,  "reached_10x_percent":   2.11
    }
  }
}
```

## Notes / gotchas

- **Same filter semantics as `snapshot_scan`.** `launchpads: ["mayhem"]` matches
  `is_mayhem_mode = 1`; real launchpads pair with `is_mayhem_mode = 0`. The
  phantom-gate (`tx_count > 1`) is applied. What you see here is exactly what
  the live alert engine would have evaluated.
- **Statistics describe ATH-after-trigger**, not arbitrary windows.
  `reached_2x = 52` means 52 coins that triggered the alert later hit ≥ 2 ×
  the trigger price during the same backtest window.
- **`excluded_by_config_cutoff` rows are informational.** They appear in
  `coins[]` and in `total_matching_coins`, but the live engine would not have
  fired them because the underlying coin was deployed before the alert's last
  filter change. Use it to gauge "what would change if I edit and re-enable
  now".
- On `TIMEOUT_EXCEEDED` / `MEMORY_LIMIT_EXCEEDED` the server returns **503**
  with `error_text: "Server is too loaded right now, please try again in a
  minute"`. That's the ClickHouse cluster, not your filters — retry with
  backoff.
- This endpoint **never mutates state**. It logs to `db.alert_simulation` for
  product analytics; no alerts are created, modified, or fired.

## UI ↔ Agent parity check (2026-05-28 verification)

Same filter via the UI's `POST /alert/simulate?sim_days=1` and via the agent
`POST /agent/alert/simulate?days=1` returned **byte-identical JSON** on dev:
identical `total_matching_coins`, identical `statistics`, identical `coins[]`
array order and content. The shared `coinSnapshotWhere.ts` builder
guarantees this — any drift between the two surfaces is a bug, not a feature.
