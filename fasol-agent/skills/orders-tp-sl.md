# Take-profit / stop-loss / trailing orders — lifecycle that re-arms

> **Sub-skill of [Fasol Agent](../SKILL.md).** Auth (`fsl_live_…`),
> rate-limit tiers, and the wallet-binding model are described in the parent.
> This page covers the persistence-and-re-arm behaviour that every strategy
> author has to internalise to avoid silent self-sabotage.

## TL;DR

When a `take_profit`, `stop_loss`, or `trailing` order **fires** (sell
executes), the order entity is **not deleted** — it just becomes
deactivated/sleeping. **The next time a position opens on the same coin,
the order re-arms** with a fresh `trigger_price` computed against the new
entry.

This is by design (so you can hit the same exit ladder repeatedly on a coin
you scalp), but it's a sharp edge for any agent that loops.

## What goes wrong if you don't manage the lifecycle

1. **Stale SL fires instantly on a new buy.** If you placed `stop_loss -7%`
   for cycle N, then opened a fresh buy in cycle N+1 at a different entry
   price, the **old SL re-arms with a new trigger relative to the new entry**.
   If that recomputed trigger has already been breached at the moment of the
   new buy, the position closes the instant it opens. From the agent's
   perspective: buy succeeds, then ~1 second later the position vanishes
   for no apparent reason. That's the deactivated SL waking up.

2. **Multiple stale orders stack.** A queue like `[TP+25%, TP+10%, SL-7%, SL-7%]`
   accumulated over 4 cycles will _all_ activate on the next buy. The first
   one to fire wins; the others sleep again. Sometimes that's fine, sometimes
   you wanted the TP+25 and got TP+10.

3. **Cross-wallet activation.** Orders are per-wallet on the engine side
   (the engine filters `o.wallet === tx.wallet`). Since your agent is bound
   to **one wallet** (see parent SKILL.md "Wallet binding"), this is invisible
   to you in practice — every order you place and every swap you fire are on
   the same wallet, so activation always works. Don't try to manage orders
   on the user's other wallets — the server rejects, and even if it didn't
   the cross-wallet activation would never trigger.

## The pattern — track every order ID, cancel on cycle end

```js
// Cycle pattern — track every relative-order id you create, cancel them all
// when the cycle ends (whether the cycle ended via TP, SL, manual exit, or
// time-out). This guarantees a clean slate for the next entry.
const placedOrderIds = [];

async function openCycle(coinAddress, entryTriggerPrice, amountSol) {
  const buy = await api("POST", "/orders", {
    body: { type: "limit_buy", coin_address: coinAddress,
            trigger_price: entryTriggerPrice, amount_sol: amountSol },
  });
  // limit_buy is one-shot, no need to track for cleanup

  const tp = await api("POST", "/orders", {
    body: { type: "take_profit", coin_address: coinAddress,
            trigger_p: "30", sell_p: "100" },
  });
  placedOrderIds.push(tp.data.id);

  const sl = await api("POST", "/orders", {
    body: { type: "stop_loss", coin_address: coinAddress,
            trigger_p: "-15", sell_p: "100" },
  });
  placedOrderIds.push(sl.data.id);
}

async function closeCycle(coinAddress) {
  // Try to cancel every relative order we created. Safe even if some already
  // fired — DELETE on a deactivated order is a no-op.
  await Promise.all(placedOrderIds.map((id) =>
    api("DELETE", `/orders/${id}`, { body: { coin_address: coinAddress } })
      .catch((err) => console.warn(`[cleanup] cancel ${id} failed: ${err.message}`)),
  ));
  placedOrderIds.length = 0;
}
```

## Surviving a restart

If the strategy is killed mid-cycle (Ctrl-C, process restart), the next run
won't know which IDs were yours. Two ways to handle this:

1. **Persist the IDs.** Write them to a file / Redis as you place them;
   reload on startup and cancel before the first buy of the new run.
2. **Best-effort sweep on startup.** Call `GET /orders` (or, more efficient,
   `GET /coin/:ca/orders` once you know which coin you're about to trade) and
   cancel anything tagged `take_profit` / `stop_loss` / `trailing` for that
   coin.

The strategy template in [`scripts/strategy-template.mjs`](../scripts/strategy-template.mjs)
does (1) automatically — it tracks the IDs it places and cancels them on
`stop` / SIGINT.

## Trailing orders — same rule, sharper consequence

Trailing TP and trailing SL re-arm on the next buy too, but with their
trailing pivot computed from the new entry. The result is more surprising
than a fixed-percent SL because the trail can chase a price that has
already moved, then snap the position closed seconds after you open it.

If your strategy uses trailing, the cleanup discipline above is **mandatory**
— don't even start without it.

## Reference

| Operation | Endpoint | Scope |
|---|---|---|
| Create TP/SL/trailing | `POST /orders` | `place_orders` |
| List your open orders | `GET /orders` | `read_positions` |
| List by coin | `GET /coin/:ca/orders` | `read_positions` |
| Cancel one | `DELETE /orders/:id` | `cancel_orders` |

Body schemas for `POST /orders` are in the parent SKILL.md "place_order"
section.
