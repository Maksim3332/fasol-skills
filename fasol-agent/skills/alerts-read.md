# `alerts_read` — list and inspect the user's alerts

> **Sub-skill of [Fasol Agent](../SKILL.md).** For writing alerts see
> [alerts-write](alerts-write.md). For multi-day backtests see
> [alert-simulate](alert-simulate.md). For the live SSE feed see
> [alert-match-stream](alert-match-stream.md).

Three read endpoints, all gated by `read_alerts`. Tier: `standard` for the
list, `heavy` for per-alert stats (5 rpm — don't poll).

| Endpoint | Tier | Purpose |
|---|---|---|
| `GET /alerts` | standard | List the user's alerts with `triggered_count` + hit-rate stats |
| `GET /alert/{alert_id}/stats` | heavy | Drill-down: every coin that matched this alert + multipliers reached |
| `GET /alerts/triggered/{coin_address}` | medium | Which of the user's alerts matched this coin (chart-marker data) |

## `GET /alerts` — overview

```bash
curl -s -H "Authorization: Bearer $FASOL_API_KEY" "$FASOL_API_BASE_URL/alerts"
```

Response items include:

- `id`, `name`, `is_paused`, `should_send_tg`, `autobuy_amount`
- The full filter config (`launchpads`, `booleanFilters`, `minMaxFilters`)
- `triggered_count` — distinct coins that have matched **since `config_updated_at`** (filter changes reset history)
- `stat_*` fields — `hit_1_5x_pct`, `hit_2x_pct`, `hit_5x_pct`, `hit_10x_pct`. Aggregate post-match price-multiplier hit rates. **Null** when there are zero matches in window.

## `GET /alert/{id}/stats` — per-alert hit-rate drill-down

For one alert: the list of matched coins, their alert price, ATH-after-match,
and which milestones each one hit. Same shape as the `coins[]` array in
[alert-simulate](alert-simulate.md) — this is for the **live** alert, not a
hypothetical config.

```bash
curl -s -H "Authorization: Bearer $FASOL_API_KEY" "$FASOL_API_BASE_URL/alert/123/stats"
```

Heavy tier — cache the result client-side for at least a minute.

## `GET /alerts/triggered/{coin_address}` — chart-marker data

Which of the user's alerts matched **this specific coin** (and when). The UI
uses it to render markers on the coin chart; an agent can use it to decide
whether the user has already had an alert opinion on a coin before reacting
to a fresh signal.

> ⚠️ **The path parameter is a coin mint, not an alert_id.** Pass a base58
> Solana mint (32 bytes / 32–44 chars). Production telemetry shows agents
> often mis-interpret this endpoint and pass an alert_id like `14867` or
> the string `all` — the server now returns **400 Invalid coin_address**
> in that case, but the underlying request is wrong: you want
> [`/alert/:id/stats`](alerts-read.md) for per-alert detail. This endpoint
> tells you "which of MY alerts matched THIS coin", not "what matched this
> alert".

```bash
curl -s -H "Authorization: Bearer $FASOL_API_KEY" \
  "$FASOL_API_BASE_URL/alerts/triggered/<COIN_MINT>"
```

On bad input the server replies:

```jsonc
{
  "error_text": "Invalid coin_address: expected a base58 Solana mint (32 bytes / 32–44 chars). The path param is a coin mint, not an alert_id.",
  "got": "<whatever you passed>"
}
```
