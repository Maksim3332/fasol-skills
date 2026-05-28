# `alerts_write` — create / update / pause / autobuy

> **Sub-skill of [Fasol Agent](../SKILL.md).** For reading alerts see
> [alerts-read](alerts-read.md). For multi-day backtests of an
> AlertConfig see [alert-simulate](alert-simulate.md).

All write endpoints require `manage_alerts` — a scope that **isn't granted by
default**; your owner must explicitly hand it over. Tier: `medium`.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/alerts` | Create alert. Body = full `AlertUpsertData` |
| `PUT` | `/alert/{alert_id}` | Update alert. Same body. **Filter change clears match history.** |
| `DELETE` | `/alert/{alert_id}` | Delete alert and its match history |
| `POST` | `/alert/{alert_id}/pause` | `is_paused = true`. Empty body. Idempotent |
| `POST` | `/alert/{alert_id}/unpause` | `is_paused = false`. Empty body. Idempotent |
| `POST` | `/alert/{alert_id}/toggle-telegram` | Flip `should_send_tg`. Empty body |
| `POST` | `/alert/{alert_id}/autobuy` | Set autobuy config (see below). Pass `null` / `0` to disable |

## `AlertUpsertData` body

```jsonc
{
  "name": "Migrated + dev sold",
  "launchpads": ["pumpfun", "raydium"],         // ≥1 required
  "booleanFilters": ["only_migrated", "with_socials", "dex_paid"],
  "minMaxFilters": {                            // any subset; nulls allowed
    "min_mc_usd": 50000, "max_mc_usd": 1000000,
    "min_vol_5m_usd": 10000,
    "min_holders": 200,
    "max_dev_hold_p": 5
    // Full key list = same min_/max_ set as snapshot_scan / alert_simulate
  },
  "milestones": [1.5, 2, 5, 10],                // multipliers tracked after match
  "is_paused": false,
  "chat_id": null,                              // null = DM the bot owner

  // Autobuy (optional — fire-and-forget buys when the alert matches)
  "autobuy_amount": 0.05,                       // SOL per match; null/0 disables
  "autobuy_orders": [                           // TP / SL / trailing armed after the buy
    { "type": "take_profit", "trigger_p": 50,  "sell_p": 100 },
    { "type": "stop_loss",   "trigger_p": -25, "sell_p": 100 }
  ],
  "ab_fee": 0.001,
  "ab_slip": 0.5,
  "ab_jito_on": false
}
```

Server enforces: `name` non-empty, ≥1 launchpad, valid `booleanFilters`
strings, sufficient SOL balance when `autobuy_amount > 0`. Returns the saved
row (`{ data: alert }`).

## Pause / autobuy shims — no round-trip on full config

```bash
# Pause
curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" \
  "$FASOL_API_BASE_URL/alert/123/pause"

# Set autobuy size + TP/SL
curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
  -d '{"autobuy_amount":0.05,"autobuy_orders":[{"type":"take_profit","trigger_p":50,"sell_p":100}]}' \
  "$FASOL_API_BASE_URL/alert/123/autobuy"

# Disable autobuy (preserves the rest of the alert)
curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
  -d '{"autobuy_amount":null,"autobuy_orders":null}' \
  "$FASOL_API_BASE_URL/alert/123/autobuy"
```

## Lifecycle gotchas

- **Filter change clears match history.** `PUT /alert/:id` with a different
  `booleanFilters` / `launchpads` / `minMaxFilters` triggers
  `clearAlertHistory(alert_id)` server-side. `triggered_count` resets and
  `stat_*` go null until new matches accumulate.
- **Autobuy positions tag `source_kind: "alert"` + `source_id: <alert_id>`.**
  With `manage_alerts` scope you may treat them as yours (cancel TP/SL, exit
  early, etc.) — without it, don't touch.
- **Tune filters with the backtest first.** Use [alert-simulate](alert-simulate.md)
  to validate a filter set against last 1-5 days BEFORE pushing it live with
  `PUT /alert/:id`. Avoids burning days of `triggered_count` history on a
  filter that turns out to over-match.
