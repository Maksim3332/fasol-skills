# `tx_stream` — live confirmation events for the bound wallet's swaps

> **Sub-skill of [Fasol Agent](../SKILL.md).** For market data streams see
> [coin-price-stream](coin-price-stream.md) /
> [coin-trade-stream](coin-trade-stream.md). For tracked-wallet swaps (other
> wallets) see [tracked-wallet-trades](tracked-wallet-trades.md).

`GET /agent_stream/tx[?coin_address=<addr>]` — SSE of every swap the platform
processes for the bound wallet (your wallet), pushed as soon as the chain
confirms. This is what closes the loop on [`swap`](swap.md) /
[`place_order`](place-order.md): wait for the `tx` event instead of polling
[`list_trades`](list-trades.md).

Requires `read_positions`. Tier: `sse`.

## Request

```bash
STREAM_BASE="${FASOL_API_BASE_URL%/trading_bot/agent}/agent_stream"

# All wallet activity
curl -N -H "Authorization: Bearer $FASOL_API_KEY" "$STREAM_BASE/tx"

# Narrow to one coin
curl -N -G -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "coin_address=<COIN>" \
  "$STREAM_BASE/tx"
```

## Wire format

```
event: ready
data: { "user_id": 50772161, "agent_id": 3, "coin_filter": null, "server_time": "..." }

data: {
  "type": "tx",
  "hash": "5Qw...",
  "status": "success" | "failed" | "pending" | "rejected" | "processed",
  "commitment": "processed" | "confirmed",
  "user_id": 50772161,
  "wallet": "...",
  "coin_address": "...",
  "buy_sell": "buy" | "sell",
  "type": "limit_buy" | "take_profit" | "stop_loss" | "trailing" | "limit_sell" | "qb" | "ml_buy" | "ml_sell" | "agent_swap",
  "amount_sol": "0.10000000",
  "amount_coin": "8123456",
  "amount_usd": "12.34",
  "price_usd": "0.00000152",
  "wallet_coin_balance_d":     "8123456",
  "post_wallet_sol_balance_d": "1234567890",
  "fees": "0.000005",
  "fasol_fee": "0.000061",
  "error_text": null
}

: heartbeat
```

You'll typically see **TWO** events per swap:
1. `commitment: "processed"` (~400 ms after submit)
2. `commitment: "confirmed"` (~3-7 s later)

Treat `confirmed` as the authoritative fill — the chain has voted on it. If
you act on `processed` you're trading speed for a small risk of reorg
invalidating the trade.

## Node consumer

```js
import { subscribeTxStream } from "./lib/sse.mjs";

for await (const evt of subscribeTxStream({ coin_address: COIN })) {
  if (evt.event !== "tx") continue;
  const tx = evt.data;
  if (tx.commitment !== "confirmed") continue;     // wait for finality
  if (tx.error_text) {
    console.error(`Trade failed: ${tx.error_text}`);
    continue;
  }
  if (tx.buy_sell === "buy" && tx.type === "limit_buy") {
    console.log(`✅ Buy filled @ $${tx.price_usd}, balance now ${tx.wallet_coin_balance_d}`);
  }
  if (tx.buy_sell === "sell" && (tx.type === "take_profit" || tx.type === "stop_loss")) {
    console.log(`✅ Exit ${tx.type} @ $${tx.price_usd}, received ${tx.amount_sol} SOL`);
  }
}
```

## `type` values you'll see

| `type` | Meaning |
|---|---|
| `limit_buy` / `limit_sell` | Orders-engine fire |
| `take_profit` / `stop_loss` / `trailing` | Relative-order fires (all sells) |
| `ml_buy` / `ml_sell` | ml_order strategy fire |
| `agent_swap` | Your instant `/swap` |
| `qb` | User manual quick-buy (UI / Telegram) |

## When to use this vs. `?wait=true` on `/swap`

| You need… | Use |
|---|---|
| One fire-and-confirm cycle | `POST /swap?wait=true` |
| Many concurrent swaps to track | `tx_stream` (one connection, fan-in by `hash`) |
| Confirmation of orders-engine fires (TP / SL / trailing) | `tx_stream` — `/swap?wait=true` only covers `/swap` calls, not orders |

## Failure modes

Same as other streams: reconnect with backoff. The fix to the SSE
silent-stop bug ([changelog](changelog.md)) applies to the tracked-wallet
stream specifically; `tx_stream` uses a different REDIS_STAT path (per-wallet
NOTIFY_FASOL_TX) and was not affected.
