# Party 

A party is any entity participating on Vega blockchain with one-to-one mapping between a Vega wallet public key and “party”. Also known as user / trader / participant / voter.


# Summary

A standard party may have:
- 0 or more [accounts](./0013-accounts.md). These accounts may contain a balance of a single [asset](./0040-asset-framework). This is known as [collateral](./0005-collateral.md)
- positions on 0 or more [markets](./0001-market-framework.md)
- orders on 0 or more [markets](./0001-market-framework.md)
- 0 or more governance proposals submitted,see [governance](./0028-governance.md)

A standard party can:
- [Submit an order](./0025-order-submission.md)
- [Amend an order](./0026-amends.md)
- [Cancel an order](./0033-cancel-orders.md)
- [Submit a governance proposal](./0028-governance.md)
- [Vote on a governance proposal](./0028-governance.md)
- [Commit to provide liquidity on a market](./0038-liquidity-provision-order-type)


# Guide-level explanation
## Standard party
A standard party is any that is controlled via transactions signed by an [ed25519 keypair](./0027-auth.md). Throughout the specifications, if the term 'party' is used without qualification, it probably means a standard party.

## Special parties
There is currently one 'special' party in Vega. They are controlled by the Vega core, and no entity should be able to perform an action as a special party other than the core. Transactions cannot be submitted via the blockchain as any special party, and there is no keypair specifically associated with a special party.

### 'network' parties
The `network` party is a pseudoparty. It is used in [position resolution](./0012/position-resolution.md) to close distressed [positions](./0006-positions-core.md) or [orders](./0024-order-status.md).

# Acceptance Criteria


## Criteria
1. When a [standard party](#standard-party) must be uniquely identified, it must be identified by the public key of an ed25519 keypair
    1. The exception to this rule is [special parties](#special-party), which are unique strings
        1. A special party must not have a public/private keypair
1. A transaction recieved through the blockchain as any [special party](#special-parties) must be rejected/
1. A [party](#standard-party) with no balance in any account must have their transactions rejected.
1. The special network party...
1. It's not a party until the police shows up.