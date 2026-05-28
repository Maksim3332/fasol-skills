# `coin_price_stream` — live SSE price ticks for one coin

> **Sub-skill of [Fasol Agent](../SKILL.md).** For every swap (not just
> price) on a coin see [coin-trade-stream](coin-trade-stream.md). For your
> own wallet's tx confirmations see [tx-stream](tx-stream.md).

`GET /agent_stream/coin/{coin_address}` — long-lived HTTP SSE. The server
forwards every price tick from the on-chain pipeline (~1 batch per Solana
block, ~400 ms) for the requested coin.

Requires `read_coins`. Tier: `sse` (concurrent-connection-capped, not rpm).

## Request

```bash
# Strip /trading_bot/agent from the API base; append /agent_stream/coin/<COIN>.
STREAM_BASE="${FASOL_API_BASE_URL%/trading_bot/agent}/agent_stream"
curl -N -H "Authorization: Bearer $FASOL_API_KEY" \
  "$STREAM_BASE/coin/<COIN>"
```

`-N` disables curl buffering so ticks appear as they arrive.

> Agent stream endpoints live under `/agent_stream/...`, not under
> `/trading_bot/agent/...`. Same Bearer token, different URL tree.

## Wire format

```
event: ready
data: { "coin_address": "...", "agent_id": 3, "server_time": "2026-04-27T..." }

data: { "type": "price", "coin_address": "...", "pair_address": "...",
        "version": "pam", "price_usd": "0.00001234", "price_sol": "0.0000000123",
        "sol_reserve_d": "...", "coin_reserve_d": "...", "slot": 372881234,
        "ts": 1745779200123 }

: heartbeat
```

- `event: ready` — sent once on connect with stream metadata
- `data: { type: "price", ... }` — sent on every tick
- `: heartbeat` — comment line every 15 s. Ignore.

If the coin migrates while you're connected, the stream **does not break**
— the server filters by `coin_address`, and the new pair's prices flow
through the same connection. `pair_address` and `version` tell you about the
migration.

SSE is one-way (server → you). To act on a tick you still call `place_order`
/ `cancel_order` over normal HTTP.

## Node consumer

```js
import { subscribeCoinPriceStream } from "./lib/sse.mjs";

const COIN = "DezX...263";
for await (const evt of subscribeCoinPriceStream(COIN)) {
  if (evt.event !== "price" && evt.event !== "ready") continue;
  if (evt.event === "ready") {
    console.log("[sse] connected", evt.data);
    continue;
  }
  const tick = evt.data;
  decideAndAct(tick);
}
```

## When to stream vs. poll

| Pattern | Use |
|---|---|
| Watch-and-buy on a price condition that moves fast (flip) | **Stream** |
| Trail / scalp / ladder exits inside a pump | **Stream** |
| Background monitor of an open position checking PnL slowly | Poll [coin-stats](coin-stats.md) every 30s |
| Heartbeat checks "is dev still holding" | Poll every 5–10 min |
| One-shot lookups for confirmation | [coin-stats](coin-stats.md) |

HTTP poll has a 60 rpm rate limit. Stream is **not** rate-limited per tick
(only the initial connect counts). One stream per coin per agent.

## Failure modes

- **Server restart** → connection drops, helper reconnects with backoff (1 s → 30 s capped).
- **Auth revoked / scope changed** → 401 / 403 — helper throws; strategy must stop.
- **Coin not active for a long time** → no ticks (normal, not a bug). Use a
  deadline timer in the strategy if "no price for N minutes" is itself a
  signal.
- **Pair migration** → no break; expect a `version` change inside the same
  stream.
