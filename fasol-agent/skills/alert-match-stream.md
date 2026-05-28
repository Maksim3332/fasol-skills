# `alert_match_stream` — live SSE of alert matches + milestones

> **Sub-skill of [Fasol Agent](../SKILL.md).** For the alert config itself
> see [alerts-read](alerts-read.md) and [alerts-write](alerts-write.md).

`GET /agent_stream/alert_matches[?alert_id=<id>]` — live Server-Sent-Events
feed of two event types from the user's alert pipeline:

- `event: alert_match` — a coin matched an alert (same event Telegram receives).
- `event: alert_milestone` — a previously-matched coin hit a multiplier
  target (1.5× / 2× / 5× / 10× by default, configurable per alert).

Optional `?alert_id=` narrows the stream to one alert. Requires `read_alerts`.
Tier: `sse`.

## Request

```bash
curl -N -H "Authorization: Bearer $FASOL_API_KEY" \
  "https://api.fasol.trade/agent_stream/alert_matches"

# Narrow to one alert
curl -N -H "Authorization: Bearer $FASOL_API_KEY" \
  "https://api.fasol.trade/agent_stream/alert_matches?alert_id=123"
```

> ⚠️ Agent stream endpoints live under `/agent_stream/...`, not under
> `/trading_bot/agent/...`.

## Wire format

```
event: ready
data: { "user_id": 50772161, "agent_id": 3, "alert_filter": null, "server_time": "..." }

event: alert_match
data: {
  "alert_id": 123,
  "alert_name": "Migrated + dev sold",
  "coin": { "coin_address": "...", "symbol": "...", "price_usd": "...", /* full CoinStat */ },
  "trigger_price": 0.0000123,
  "timestamp": 1745779200123
}

event: alert_milestone
data: {
  "alert_id": 123,
  "coin": { /* CoinStat */ },
  "multiplier": 2,
  "baseline_price": 0.0000123,
  "current_price": 0.0000247,
  "timestamp": 1745779260000,
  "alert_timestamp": 1745779200123
}

: heartbeat
```

The stream is filtered server-side by your owner's `user_id` — you only see
their alerts. Telegram-delivery fields (`chat_id`, `should_send_tg`) are
stripped — agent doesn't need them.

## Behaviour notes

- **Match dedup is in-memory in REDIS_STAT.** A coin matches at most once per
  match-window per `(alert, coin)` pair. You won't see duplicate `alert_match`
  events for the same hot pumpfun coin.
- **Milestones are post-match.** No match → no milestones. After a match,
  milestones fire on each multiplier crossing using the alert's `milestones`
  config (default `[1.5, 2, 5, 10]`).
- **Heartbeats every 15s** — same liveness contract as other SSEs.

## Worked example — react to matches without platform autobuy

```js
import { subscribeAlertMatchStream } from "./lib/sse.mjs";

for await (const evt of subscribeAlertMatchStream()) {
  if (evt.event === "alert_match") {
    const { alert_id, alert_name, coin, trigger_price } = evt.data;
    // Pull a fresh stats snapshot, decide whether to enter:
    const stats = await api("GET", `/coin/${coin.coin_address}/stats`);
    if (looksGood(stats)) {
      await api("POST", "/swap", { body: {
        direction: "buy", coin_address: coin.coin_address, amount_sol: 0.05,
      }});
      // Then arm TP/SL via /orders — see place-order.md
    }
  }
  if (evt.event === "alert_milestone") {
    // Coin moved Nx since match — maybe trim the position.
  }
}
```

## When to use what

| Pattern | Use |
|---|---|
| Mirror the TG-bot match feed in your strategy | `alert_match_stream` (this) |
| Trade the alert's autobuy yourself instead of platform | `alert_match_stream` + [`swap`](swap.md) + [`place_order`](place-order.md) |
| Let platform autobuy fire; you only manage exits | Set `autobuy_amount` via [alerts-write](alerts-write.md) |
| Pause a noisy alert mid-run | `POST /alert/:id/pause` (see [alerts-write](alerts-write.md)) |
