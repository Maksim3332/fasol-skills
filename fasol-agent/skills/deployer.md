# `dev_history` — deployer's launch history

> **Sub-skill of [Fasol Agent](../SKILL.md).**

`GET /dev/{deployer_address}` — the deployer's last 50 tokens with summary
stats. Use to assess whether a coin's deployer has a history of getting
launches to migration vs. serial-rugging.

Requires `read_dev_history`. Tier: `medium`.

## Request

```bash
curl -s -H "Authorization: Bearer $FASOL_API_KEY" \
  "$FASOL_API_BASE_URL/dev/<DEPLOYER_ADDRESS>"
```

You can get the deployer address from [`coin_stats`](coin-stats.md)
(`deployer` field).

## Response

```json
{
  "data": {
    "deployer": "...",
    "launched_count": 87,
    "migrated_count": 12,
    "migrated_p": 13.8,
    "last3_avg_ath": 42000,
    "last5_avg_ath": 38500,
    "tokens": [
      {
        "coin_address": "...",
        "symbol": "...",
        "launchpad": "pf",
        "ath_mc_usd": 87000,
        "current_mc_usd": 12000,
        "is_migrated": true,
        "created_at": 1745779200000
      }
      // ...
    ]
  }
}
```

## When to use

- New-coin discovery: filter out deployers with `migrated_p < 5` and
  `launched_count > 20` — serial rug pattern.
- Confirm a "good deployer" hunch when `coin_stats.dev_pf_migrated_p` looks
  promising — the full token list shows ATH distributions, not just the rate.
- Cross-reference with [`wallet_search`](wallet-search.md) when looking for
  reliable deployer cohorts.

> Most `coin_stats` already includes the headline deployer stats
> (`dev_pf_launched_count`, `dev_pf_migrated_count`, `dev_pf_migrated_p`,
> `dev_last3_avg_ath`, `dev_last_migrated`). Reach for `dev_history` when you
> need the per-token list, not just aggregates.
