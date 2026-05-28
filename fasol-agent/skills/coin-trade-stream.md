# `coin_trade_stream` — every swap on one coin (volume / VWAP / order-flow)

> **Sub-skill of [Fasol Agent](../SKILL.md).** For OHLC-style price-only
> ticks see [coin-price-stream](coin-price-stream.md). For your wallet's tx
> confirmations see [tx-stream](tx-stream.md).

`GET /agent_stream/coin/{coin_address}/trades` — SSE: one event per
on-chain swap on the coin (buy or sell, any wallet), with size, price, and
per-wallet aggregated context.

This is what you subscribe to when you need to compute rolling indicators
yourself: 1m volume, buy/sell ratio, VWAP, order-flow imbalance, maker count,
anything tick-based.

Requires `read_coins`. Tier: `sse`.

## Request

```bash
STREAM_BASE="${FASOL_API_BASE_URL%/trading_bot/agent}/agent_stream"
curl -N -H "Authorization: Bearer $FASOL_API_KEY" \
  "$STREAM_BASE/coin/<COIN>/trades"
```

## Wire format

```
event: ready
data: { "coin_address": "...", "agent_id": 3, "server_time": "..." }

data: {
  "type": "trade",
  "coin_address": "...",
  "trade": {
    "wallet": "Cs7c...",
    "buy_sell": "buy" | "sell",
    "amount_sol": 0.42,
    "amount_coin": 1234567,
    "hash": "5Qw...",
    "date": 1745779200123,
    "price_usd": "0.0000123",
    "in_sol": 0.5, "out_sol": 0.0,
    "coin_balance": 1234567,
    "buy_count": 1, "sell_count": 0,
    "first_tx_at": 1745779200123, "last_tx_at": 1745779200123,
    "trade_type": "...",
    "pnl_sol": 0, "pnl_percent": 0
  }
}

: heartbeat
```

The `trade` object is the same `LiveTrade` shape the coin terminal renders.
Most useful for indicators:

- `buy_sell`, `amount_sol`, `amount_coin`, `price_usd`, `date` — the swap itself
- `wallet` — unique-makers / sniper detection / dev-sell guards
- `trade_type` — server-side classification (sniper / fresh / bot_trader / scammer / profit_trader / …)

## Worked example — rolling 1m volume + buy/sell ratio

```js
import { subscribeCoinTradeStream } from "./lib/sse.mjs";

const window_ms = 60_000;
const buf = []; // { ts, side, sol }

for await (const evt of subscribeCoinTradeStream(COIN)) {
  if (evt.event !== "trade") continue;
  const t = evt.data.trade;
  const ts = t.date ?? Date.now();
  buf.push({ ts, side: t.buy_sell, sol: t.amount_sol });

  // Drop stale entries
  const cutoff = Date.now() - window_ms;
  while (buf.length && buf[0].ts < cutoff) buf.shift();

  const buys  = buf.filter(x => x.side === "buy");
  const sells = buf.filter(x => x.side === "sell");
  const buyVol  = buys.reduce((a, b) => a + b.sol, 0);
  const sellVol = sells.reduce((a, b) => a + b.sol, 0);
  const ratio = sellVol === 0 ? Infinity : buyVol / sellVol;

  log("indicators_1m", {
    swaps: buf.length,
    buy_vol_sol: buyVol.toFixed(4),
    sell_vol_sol: sellVol.toFixed(4),
    buy_sell_ratio: ratio.toFixed(2),
  });
}
```

## Difference from `coin_price_stream`

| Stream | Granularity | Volume? | Per-wallet? | When to use |
|---|---|---|---|---|
| [coin-price-stream](coin-price-stream.md) | one event per parsed block (price-only) | ❌ | ❌ | Reactive entry/exit triggers, chart-style |
| coin-trade-stream | one event per swap | ✅ | ✅ | Volume / VWAP / order-flow / maker analysis |

Use both side-by-side if your strategy needs both — they share zero load on
the server.

## Failure modes

Same as price stream: reconnect with backoff, terminal on 401/403/404. Pair
migrations don't break the stream.
