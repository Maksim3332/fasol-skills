# `wallet_groups` — folders for tracked wallets

> **Sub-skill of [Fasol Agent](../SKILL.md).** For the wallets themselves and
> the live SSE feed, see [tracked-wallets](tracked-wallets.md) and
> [tracked-wallet-trades](tracked-wallet-trades.md).

Wallet groups are user-defined folders that organise the tracked-wallets
list — "smart money", "snipers", "personal", etc. CRUD only; the engine
doesn't behave differently based on group membership. `manage_tracking` scope.

## Endpoints

| Method + Path | Purpose |
|---|---|
| `GET /wallet_groups` | List the user's wallet groups |
| `POST /wallet_groups` | Create a group: `{ name, color? }` |
| `PUT /wallet_groups/:id` | Rename / re-colour a group |
| `DELETE /wallet_groups/:id` | Delete a group (its wallets become ungrouped, not deleted) |

## Examples

```bash
# Create a group
curl -s -X POST -H "Authorization: Bearer $FASOL_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"snipers","color":"#FF6633"}' \
  "$FASOL_API_BASE_URL/wallet_groups"

# List
curl -s -H "Authorization: Bearer $FASOL_API_KEY" "$FASOL_API_BASE_URL/wallet_groups"
```

## Use

- Pre-organise wallets before bulk-adding via `POST /tracked_wallets` (which
  takes an optional `group_id` per wallet).
- Surface group names in copy-trader UIs / notifications so the user can
  recognise which cohort fired a signal.
- Filter the live stream client-side by `trade.group_id`.

Delete is non-destructive to the wallets — they survive ungrouped.
