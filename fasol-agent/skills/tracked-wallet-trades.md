# `tracked_wallet_trade_stream` — copy-trade other wallets

> **Sub-skill of [Fasol Agent](../SKILL.md).** Read the parent first for auth
> (`fsl_live_…`), rate-limit tiers, and the wallet-binding model. This page
> only describes the streaming endpoint and the copy-trader pattern.

`GET /agent_stream/tracked_wallet_trades` — live Server-Sent-Events feed of
every swap from any wallet the user has added to their tracked list. The
server filters by the authenticated `user_id`, so you only ever see your own
list's activity. Requires the `manage_tracking` scope.

**Tier:** `sse` (concurrent-connection-capped, not rpm).

---

## Request

```bash
curl -N -H "Authorization: Bearer $FASOL_API_KEY" \
  "https://api.fasol.trade/agent_stream/tracked_wallet_trades"
```

> ⚠️ Agent stream endpoints live under `/agent_stream/...`, NOT under
> `/trading_bot/agent/...`. Same `$FASOL_API_KEY` bearer token, but a different
> URL tree from the regular HTTP routes.

## Wire format

```
event: ready
data: { "user_id": 50772161, "agent_id": 3, "server_time": "...", "note": "..." }

data: {
  "type": "tracked_trade",
  "user_id": 50772161,
  "trade": {
    "wallet": "Cs7c...",
    "coin_address": "...",
    "buy_sell": "buy",
    "amount_sol": 0.42,
    "in_sol": 0.5,
    "out_sol": 0.0,
    "coin_balance": 1234567,
    "buy_fees": 0.000054,
    "sell_fees": 0,
    "buy_bot_fees": 0,
    "sell_bot_fees": 0,
    "buy_count": 1,
    "sell_count": 0,
    "first_tx_at": 1745779200123,
    "last_tx_at":  1745779200123,
    "trade_type": "first_buy",
    "pnl_sol": 0,
    "pnl_percent": 0,
    "symbol": "...",
    "image": "...",
    "pair_version": "...",
    "coin_created_at": 1745778000000,
    "mc": "...",
    "wallet_label": "...",
    "wallet_emoji": "🦊",
    "group_id": null,
    "wallet_sol_balance": 12.34
  }
}

: heartbeat
```

Heartbeats arrive every 15 seconds. Use them to detect a dead TCP pipe — if
you go > 30 seconds without one, the connection is gone.

### Quirks of the trade payload

- **Timestamps:** the payload uses `first_tx_at` / `last_tx_at` (ms epoch).
  There is no plain `date` field. `first_tx_at` is the first swap in the current
  cycle (resets on `sell_all`); `last_tx_at` is the swap you just received.
- **No price:** the payload doesn't carry `amount_coin` or `price_usd`.
  To act on a wallet trade, either fire an instant `/swap` (the server uses
  live on-chain reserves, no price needed) or call `coin_stats` for the current
  price snapshot.

### `trade_type` values — copy-trading filter logic

Ignore at your peril.

| Value | Meaning | Copy-trading note |
|---|---|---|
| `first_buy` | Wallet had 0 of this coin before this swap | **Best entry signal** — mirror these aggressively. |
| `buy_more` | Wallet adding to existing position | Trader's position is already in motion. Their `pnl_percent` here is cumulative since `first_tx_at` — if it's already +30% the pump is mostly behind you. **Filter out unless `pnl_percent < 10`** (you'd be entering near their average). |
| `sell_part` | Selling some, still holds | **Exit signal** — the trader is locking profit. Don't wait for `sell_all`; many wallets ride a tail. |
| `sell_all` | `coin_balance` returns to 0 | Final exit. Mirror immediately. |
| `sell_air` | Edge case: sell with no buy in window | Rare; treat as a sell. |

---

## Connection lifecycle — read this once

The SSE has a stale-state class of bugs whose root cause has now been
identified and fixed on dev — but you should still understand the shape, because
the workaround needs to stay until the fix reaches prod.

### Historical symptom (still possible on `api.fasol.trade` until prod roll-out)

A connection alive ~10+ minutes silently stops delivering `tracked_trade`
events **while heartbeats keep arriving**. A fresh `curl` on the same key /
wallets gets events within seconds. The connection looks fine at the socket
layer but the server-side dispatch table no longer routes events to your stream.

### Root cause (now known)

REDIS_STAT's per-user `walletToUsers` cache evicts a user after 30 min without
any swap on their tracked wallets (see `CACHE_TTL_MS` in
`src/redisStat/wallet/liveTradesTrack.ts`). The eviction is a no-op for the
SSE connection — heartbeats are generated locally in the WS process, so
TCP/HTTP look healthy from both sides. But once evicted, `dispatchSwap` no
longer publishes `NEW_LIVE_TRADE` for this user_id, and the agent sees a
silent feed.

### Fix (dev rolled 2026-05-28, see [changelog](changelog.md))

The WS now re-publishes `JOINED_LIVE_TRADES` every ~60s from the heartbeat
timer. REDIS_STAT's `addUserToCache` is idempotent and refreshes `userLastSeen`,
so the eviction never fires while the SSE is alive.

### What the agent should do

- **Connecting to `api.dev-1.mymadrobot.com`:** the workaround can be removed.
  Just open the SSE and stay on it. If you go silent, it's a different bug —
  report it via the changelog.
- **Connecting to `api.fasol.trade` (prod):** keep the 4-minute `AbortController`
  force-reconnect (see [Runnable template](#runnable-template) below) until
  [changelog.md](changelog.md) records "✅ prod roll-out". Same hosts use the
  same `$FASOL_API_KEY`.

---

## Worked example — mirror a smart-money wallet

```js
import { subscribeTrackedWalletTradeStream } from "./lib/sse.mjs";

const SMART_WALLET = "Cs7c...";  // already added to tracked_wallets via REST
const COPY_RATIO = 0.1;           // copy 10% of their size

for await (const evt of subscribeTrackedWalletTradeStream()) {
  if (evt.event !== "tracked_trade") continue;
  const t = evt.data.trade;
  if (t.wallet !== SMART_WALLET) continue;       // multiple wallets in your list — narrow client-side
  if (t.buy_sell !== "buy") continue;            // only mirror their buys
  const amount_sol = (t.amount_sol * COPY_RATIO).toFixed(4);

  // Sanity check first — fetch coin_stats and decide whether to enter.
  // Then fire an instant market buy. /swap uses the current on-chain price,
  // so there's no stale-price risk like there would be with limit_buy.
  await api("POST", "/swap", {
    body: { direction: "buy", coin_address: t.coin_address, amount_sol },
  });
}
```

## When to use this stream vs. polling

| Pattern                                           | Use                                  |
|---------------------------------------------------|--------------------------------------|
| Watch one wallet for buy signals (copy-trade)     | `tracked_wallet_trade_stream`        |
| Detect dev sells on coins you hold                | `tracked_wallet_trade_stream` + filter to deployer wallets |
| Catch up on what tracked wallets did while offline | `GET /tracked_wallets/live_trades` (one-shot warm-up — see parent SKILL.md) |
| Manage the watch list itself                      | `wallet_groups` / `tracked_wallets` CRUD |

## Production copy-trader checklist

A toy `for await` loop will work for an hour and then quietly drift. A
production-grade copy-trader needs all of the following:

1. **Bootstrap warm-up.** Before opening the stream, call
   `GET /tracked_wallets/live_trades` once and pre-load your local "what state
   is each (wallet, coin) in" Map. The stream gives you _new_ swaps, not history.
2. **Dedup against the bootstrap.** The first few SSE events can replay the
   tail of the bootstrap batch. Skip any event whose `(wallet, coin_address, last_tx_at)`
   you already saw.
3. **Filter `trade_type` deliberately** — see the table above. The default of
   "react to every buy_sell" is wrong for almost every strategy.
4. **Rate-limit your own outbound `/swap` calls.** Concurrent buys from a
   burst of `first_buy` events can starve the `place_orders` rpm quota.
5. **Wallet-binding awareness.** Every swap fires on the agent's bound wallet
   (see parent SKILL.md). Confirm `GET /scope` returns a wallet you intended.
6. **Trailing TP/SL hygiene** — see the parent SKILL.md "TP/SL/trailing
   lifecycle" section. Orders persist and re-arm across cycles; explicitly
   `DELETE` them after each exit.
7. **(prod only, until fix rolls out)** Wrap the fetch in an `AbortController`
   with a 4-minute timeout. On abort, restart the loop with the bootstrap.
   See [Connection lifecycle](#connection-lifecycle--read-this-once) above.

A complete reference implementation lives at
[`scripts/copy-trader.mjs`](../scripts/copy-trader.mjs).
