# `snapshot_scan` — cross-coin discovery at a moment in time

> **Sub-skill of [Fasol Agent](../SKILL.md).** Same filter language as
> [alert-simulate](alert-simulate.md) (shared CH builder). For per-coin
> historical state see [snapshot-history](snapshot-history.md).

`POST /snapshot/scan` — "find the coins in state X right now (or at moment T)".
For each coin we take its freshest snapshot within a 5-minute lookback and
apply your filters.

Requires `read_coins`. **Heavy tier** (5 rpm) — each call scans `db.coin_snapshot`
across hundreds of thousands of coins. Don't loop.

## Request

```bash
curl -s -X POST \
  -H "Authorization: Bearer $FASOL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "at": "now",
    "filters": {
      "min_liq": 50000,
      "max_mc": 500000,
      "max_dev_hold_p": 5,
      "is_migrated": true,
      "launchpad": "pf"
    },
    "sort":  "liq desc",
    "limit": 50
  }' \
  "$FASOL_API_BASE_URL/snapshot/scan"
```

| Param | Values | Default |
|---|---|---|
| `at` | `"now"` or ISO timestamp within last 24h | `"now"` |
| `filters` | object — required (≥ 1 filter), see whitelist below | required |
| `sort` | one of `mc`, `liq`, `vol_5m`, `holders`, `drop_from_ath_p`, `coin_age_sec`, `snapshot_date` — each with optional `asc` / `desc` (default `desc`) | `mc desc` |
| `limit` | ≤ 100 | 50 |

**`400 filter_required`** if `filters` is empty — server refuses to scan all
coins.

## Filter whitelist

The numeric filter set is **auto-derived** from the same field map the alert
backtest uses, so what you can filter on here is identical to what you can
filter on in `/alert/simulate`. Every CH column on `db.coin_snapshot` gets a
`min_<col>` and `max_<col>` filter; pass only the side you want.

**Numeric columns** (each available as `min_<col>` and `max_<col>`):

- Market: `mc`, `ath_mc`, `liq`, `holders`, `coin_age_sec`, `drop_from_ath_p`, `migration_p`, `dex_tax_p`, `dex_boost_amount`
- Volume: `vol_usd`, `vol_1m`, `vol_3m`, `vol_5m`, `buy_vol_1m`, `buy_vol_3m`, `buy_vol_5m`, `sell_vol_1m`, `sell_vol_3m`, `sell_vol_5m`
- Tx counts: `tx_count`, `buy_tx`, `sell_tx`, `buy_tx_count_1m`/`3m`/`5m`, `sell_tx_count_1m`/`3m`/`5m`, `makers_1m`/`3m`/`5m`
- Holders: `top_10_p`, `dev_hold_p`, `snipers_hold_p`, `bundlers_hold_p`, `fresh_hold_p`, `fresh_count`, `bot_traders_count`, `bot_traders_hold_p`, `profit_trader_count`
- Dev history: `dev_launched_count`, `dev_migrated_count`, `dev_migrated_p`, `dev_pf_launched_count`, `dev_pf_migrated_count`, `dev_pf_migrated_p`, `dev_last3_avg_ath`, `dev_last3_avg_bot_fee`, `dev_last5_avg_ath`, `dev_last5_avg_bot_fee`
- Fees: `bot_fee`, `buy_bot_fee_1m`/`3m`/`5m`, `sell_bot_fee_1m`/`3m`/`5m`, `global_fees`

**Boolean:** `is_migrated`, `with_socials`, `dex_paid`, `is_mayhem_mode`, `is_cashback_coin`, `dev_last_migrated`.

**String:** `launchpad` — one of `pf`, `rl`, `letsbonk`, `believe`, `bags`,
`moonshot`, `jupstudio`, `dbc`, `mayhem`, `heaven`. Passing `"mayhem"` matches
coins with `is_mayhem_mode = 1` (the virtual launchpad) regardless of their
real launchpad; passing any other value matches coins on that launchpad with
`is_mayhem_mode = 0`. Same semantics as the alert backtest — you can't get a
snapshot row here that the alert engine would have rejected.

## Response

```json
{
  "data": {
    "at": "2026-05-28T19:00:00Z",
    "rows": [
      {
        "coin_address": "...",
        "symbol": "BONK",
        "snapshot_date": "2026-05-28T18:59:42Z",
        "price_usd": "0.0000123",
        "mc": "234000",
        "liq": "67000",
        "holders": 456,
        "dev_hold_p": "3.2",
        "is_migrated": true,
        "launchpad": "pf"
      }
      // ... up to `limit` rows
    ]
  }
}
```

## Notes

- **Phantom-gate** (`tx_count > 1`) is applied — deploy-second BC-seed
  snapshots are dropped. Same as the alert engine.
- **Don't reconstruct this with `coin_stats`.** `coin_stats` is one coin at a
  time; `snapshot_scan` does the same job in one bounded query.
- For multi-day backtests with ATH-multiplier statistics use
  [alert-simulate](alert-simulate.md) instead.
