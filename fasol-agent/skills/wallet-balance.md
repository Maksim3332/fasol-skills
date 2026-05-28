# `wallet_balance` — bound-wallet SOL balance

> **Sub-skill of [Fasol Agent](../SKILL.md).**

`GET /wallet_balance` — live SOL balance of the agent's bound wallet, with
the current USD value pre-computed using the SOL price.

Requires `read_positions`. Tier: `medium`.

## Request

```bash
curl -s -H "Authorization: Bearer $FASOL_API_KEY" "$FASOL_API_BASE_URL/wallet_balance"
```

## Response

```json
{
  "data": {
    "wallet": "...",
    "sol_balance_lamports": "1234567890",
    "sol_balance":          "1.234567890",
    "sol_balance_usd":      "234.57",
    "sol_price_usd":        "190.12"
  }
}
```

## Use

- Sanity check at the start/end of a strategy run.
- Sizing decisions: "can I afford this 0.1 SOL buy plus 0.0006 SOL of fees?".
- Detect funding events (balance jumped between checks).

`wallet_balance` is a coarse cross-check; for per-cycle accounting use
[`list_trades`](list-trades.md), which is precise and traces every fee.
