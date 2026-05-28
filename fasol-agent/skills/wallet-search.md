# `wallet_search` — discover wallets by profit / activity / behaviour

> **Sub-skill of [Fasol Agent](../SKILL.md).** Feeds into
> [tracked-wallets](tracked-wallets.md) → [tracked-wallet-trades](tracked-wallet-trades.md)
> → [swap](swap.md).

`POST /wallet_search` finds Solana wallets matching a profit / activity /
behaviour profile. Primary use case: feed for `tracked_wallets` —
*"find top profit-trader wallets active in the last hour and add them to my
tracking list, then mirror their buys."*

Pipeline:

```
wallet_search → tracked_wallets (POST) → tracked_wallet_trade_stream → swap or place_order
```

Requires `read_wallets` (default-on). Read-only. Tier: `medium`.

## Request

```bash
curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "filters": {
      "min_total_profit_usd": 5000,
      "min_win_rate": 0.55,
      "min_cnt_coins": 30,
      "max_snipe_p": 0.3,
      "last_active_within_sec": 7200,
      "min_trades_24h": 50
    },
    "sort": "total_profit_usd desc",
    "limit": 50
  }' \
  "$FASOL_API_BASE_URL/wallet_search"
```

| Body field | Type | Notes |
|---|---|---|
| `filters` | object | At least 1 whitelisted key. Required. |
| `sort` | string | `<col> [asc\|desc]`. Default `total_profit_usd desc`. |
| `limit` | number | 1..100. Default 50. |
| `pf_share_48h` | boolean | Opt-in. Adds `pf_share_48h` field per row (share of trades on pumpfun over 48h, 0..1). **Heavy CH JOIN** — use sparingly, cache. |

## Common patterns

| Goal | Snippet |
|---|---|
| Active in last hour | `"last_active_within_sec": 3600` |
| Active in last day | `"last_active_within_sec": 86400` |
| Top profit traders, NOT MEV/snipers | `"wallet_type": "profit_trader", "max_snipe_p": 0.2, "max_median_hold_sec": 300` |
| Hold-and-wait (slow) traders | `"min_median_hold_sec": 600, "min_cycle_count": 10` |
| Have ≥ 1 SOL on hand | `"min_balance_sol": 1` |
| Pumpfun-heavy traders | `"pf_share_48h": true` (top-level) + `"min_pf_share_48h": 0.7` (in filters) |
| ≥ 30 completed trade cycles in last 30d | `"min_cycle_count": 30` |

## Filter whitelist

Filters split into two stages by where they apply:

### A) Pre-filters — applied at SQL level on `db.wallet` (cheap, partition-pruned)

| Key | Source | Note |
|---|---|---|
| `min_total_profit_usd` / `max_total_profit_usd` | `db.wallet.total_profit_usd` | Lifetime realised PnL in USD. |
| `min_total_x` / `max_total_x` | `db.wallet.total_x` | Lifetime sold/invested ratio. |
| `min_win_rate` / `max_win_rate` | `db.wallet.win_rate` | Share of coins with positive PnL (0..1). |
| `min_cnt_coins` / `max_cnt_coins` | `db.wallet.cnt_coins` | Distinct coins traded. |
| `max_low_coin_p` | `db.wallet.low_coin_p` | Share of low-quality coins. Lower = cleaner. |
| `min_snipe_p` / `max_snipe_p` | `db.wallet.snipe_p` | Share of sniper-pattern entries. |
| `max_scum_deal` | `db.wallet.scum_deal` | Count of "scum deal" coins. |
| `min_gini` / `max_gini` | `db.wallet.gini` | Profit concentration; high = single-coin lottery. |
| `min_profit_80` | `db.wallet.profit_80` | Profit from top-80% of coins. |
| `last_active_within_sec` | `db.wallet.last_tx_at` | Active in last N seconds. |
| `wallet_type` | `db.wallet.type` | One of `fresh`, `sniper`, `profit_trader`, `scammer`. |
| `min_trades_24h` | live count from `db.swap_w` | Adds a JOIN — counts swaps in last 24h. |

### B) Post-filters — applied in memory after Stage 1 + behavior + balance enrichment

Drop rows where the field is null. When you use any post-filter the server
**over-fetches the candidate pool 3×** to still try to fill `limit`.

| Key | Field checked | Use for |
|---|---|---|
| `min_cycle_count` | `behavior.cycle_count` | Statistically-meaningful sample |
| `max_median_hold_sec` / `min_median_hold_sec` | `behavior.median_hold_sec` | Avoid MEV (sub-minute) or scalpers |
| `max_avg_hold_sec` | `behavior.avg_hold_sec` | Same axis, mean instead of median |
| `min_balance_sol` | `balance_sol` | Wallets that can actually trade size |
| `min_pf_share_48h` / `max_pf_share_48h` | `pf_share_48h` | Pumpfun vs. Raydium — requires `pf_share_48h: true` top-level flag |

Unknown keys are silently ignored — clients can't sneak through arbitrary SQL.

**Result-count contract:** when post-filters are present, `rows.length` may
be less than `limit` if too few candidates passed. Inspect `candidates_scanned`
in the response — if it's at the over-fetch ceiling (`limit × 3`) and you got
few rows back, your post-filters are tighter than the candidate pool. Loosen
criteria, or raise `limit`.

### Not supported

- **`balance_sol` as a pre-filter** — by design. Redis-only field, can't be
  SQL-filtered. Use the `min_balance_sol` post-filter.
- **Behavior pre-filters** — by design. Computed per-cycle from `db.trade`;
  filtering before computing them would scan the entire `db.trade`.

## Sort whitelist

`total_profit_usd`, `total_x`, `win_rate`, `cnt_coins`, `gini`, `profit_80`,
`last_tx_at` (each with `asc` / `desc`).

## Response shape

```jsonc
{
  "data": {
    "applied_filters": ["min_total_profit_usd", "min_win_rate", "min_trades_24h"],
    "sort": "w.total_profit_usd DESC",
    "limit": 50,
    "pf_share_48h_enabled": false,
    "candidates_scanned": 50,
    "cache": { "hit": false },
    "rows": [
      {
        "wallet": "4BdKaxN8...",
        "cnt_coins": 277,
        "low_traders_coins": 12, "snipe_coins": 8, "scum_deal": 0,
        "total_profit_usd": 65263.82, "total_x": 1.68, "win_rate": 0.73,
        "low_coin_p": 0.04, "snipe_p": 0.03,
        "gini": 0.42, "profit_80": 52211.05,
        "type": "profit_trader",
        "last_tx_at": "2026-04-30T12:34:56Z",
        "first_tx_at": "2025-08-01T03:11:00Z",
        "trades_24h": 552,
        "behavior": {
          "cycle_count": 68,
          "avg_hold_sec": 312, "median_hold_sec": 57, "sub_60s_cycles": 21,
          "avg_buys_per_cycle": 1.4, "avg_sells_per_cycle": 1.1,
          "winning_cycles": 49, "losing_cycles": 19,
          "avg_pnl_sol_per_cycle": 0.34, "best_cycle_pnl_sol": 12.5
        },
        "balance_sol": 2684.05
      }
    ]
  }
}
```

`behavior` is `null` for wallets with no completed cycles in the 30d window.
`balance_sol` is `null` for wallets inactive 7+ days (Redis cold).

## Worked example — auto-discover smart money + start tracking

```bash
# Step 1 — discover top profit traders, active recently, healthy hold pattern.
WALLETS=$(curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
  -d '{"filters":{"min_total_profit_usd":50000,"min_win_rate":0.6,"max_snipe_p":0.2,"last_active_within_sec":3600,"min_trades_24h":20},"limit":10}' \
  "$FASOL_API_BASE_URL/wallet_search" \
  | jq -r '.data.rows[].wallet')

# Step 2 — track them.
echo "$WALLETS" | jq -R -s 'split("\n") | map(select(length>0)) | {wallets: map({wallet: .})}' \
  | curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
      -d @- "$FASOL_API_BASE_URL/tracked_wallets"

# Step 3 — subscribe and react. See tracked-wallet-trades.md.
```

## Caching & freshness

- **Server-side cache:** 5 min TTL keyed on canonicalised request. Re-issue
  the same request within 5 min → `"cache":{"hit":true}` — free of charge
  against rate limit.
- **`db.wallet` rebuild cadence:** dbt runs hourly. Stats can be up to ~1h
  stale.
- **`behavior` window:** rolling last 30 days from `db.trade`, dbt-rebuilt
  hourly.
- **`balance_sol`:** per-block by `fasol_py_stat`, TTL 7 days. `null` only
  for wallets inactive 7+ days.

Combined: results are at most ~1 h 5 min stale relative to chain. For
real-time activity, subscribe to
[tracked-wallet-trades](tracked-wallet-trades.md) after adding them.

## Cost & rate limits

- Default request: ~150–600 ms (uncached).
- With `pf_share_48h: true`: 700–2500 ms — expensive, gate behind your own
  logic so the agent doesn't poll it.
- Cached: <5 ms.
