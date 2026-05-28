# `candles` — OHLC historical + near-real-time

> **Sub-skill of [Fasol Agent](../SKILL.md).** For sub-second tick reaction
> use [coin-price-stream](coin-price-stream.md) instead. For one-shot current
> price use [coin-stats](coin-stats.md) `.price_usd`.

Two candle endpoints, both `read_coins`. Use for chart-style data —
backtesting an entry, plotting a recent move, computing simple TA (EMA, ATR)
on the fly.

Both return the same shape:

```json
{
  "coin_address": "...",
  "interval": 5,
  "candles": [
    { "ts": 1745779200, "open": "0.0000123", "high": "...", "low": "...", "close": "..." }
  ]
}
```

`ts` is unix seconds. Each candle's `open` equals the previous candle's
`close` — server-side post-processing for visual continuity.

## `get_candles` — historical from ClickHouse

`GET /coin/{coin_address}/candles?interval=5&before=<ts>&after=<ts>`

Tier: **heavy** (`db.price` scan; rate-limited as such — don't sweep).
Cursor-paginated: pass `before=<unix_sec>` to walk back, `after=<unix_sec>`
to walk forward. Up to **1000 candles per call**, `interval` 1–3600 seconds.

```bash
# Last 1000 5-second candles, walking back from now
curl -s -G -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "interval=5" \
  "$FASOL_API_BASE_URL/coin/<COIN>/candles"

# Walk further back: pass the oldest ts from the previous call as `before`
curl -s -G -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "interval=5" \
  --data-urlencode "before=1745776000" \
  "$FASOL_API_BASE_URL/coin/<COIN>/candles"
```

Tips:
- For "since the coin migrated" — call [coin-stats](coin-stats.md) first,
  read `pair_created_seconds_ago`, compute the unix timestamp, pass as `after`.
- 1-min candles = `interval=60`; 1-hour = `interval=3600` (cap).
- Coverage is bounded by `db.price` retention (rolling weeks).

## `get_candles_fast` — last ~5 minutes from Redis

`GET /coin/{coin_address}/candles_fast?interval=5`

Tier: **standard**. Sub-second freshness, no cursor — just the latest window.
`interval` 1–300 seconds.

```bash
curl -s -G -H "Authorization: Bearer $FASOL_API_KEY" \
  --data-urlencode "interval=5" \
  "$FASOL_API_BASE_URL/coin/<COIN>/candles_fast"
```

## When to use which

| You want… | Use |
|---|---|
| Last 5–60 min of 5s candles to plot or scan | `get_candles_fast` |
| Anything older than ~5 min, or a long window | `get_candles` (with `before`) |
| Tick-by-tick reaction inside a flip / scalp | [coin-price-stream](coin-price-stream.md) (SSE) |
| One-shot "price right now" | [coin-stats](coin-stats.md) `.price_usd` |
| Full coin state at a moment | [snapshot-history](snapshot-history.md) |

Don't poll `get_candles_fast` faster than every 5 seconds — for sub-second
loops use the SSE stream and aggregate ticks yourself.
