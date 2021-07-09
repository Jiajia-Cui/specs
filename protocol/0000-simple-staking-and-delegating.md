# Simple Staking and Delegation
Vega runs on a delegated proof of stake (DPOS) blockchain. Participants who hold a balance of the governance asset can stake these on the network by delegating their tokens to one or more validators that they trust. This helps to secure the network. 

Validators and delegators receive incentives from the network, depending on various factors, including how much stake is delegated and how honest they are.

## Note on terminology

Staking requires the combined action of:
- Locking tokens on the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md); and 
- Delegating these tokens to a one or more validators

Delegation and staking are terms that may be used interchangably, since delegation is the act of staking VEGA tokens on a validator.

A delegator can lock a token in the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md), which is then available for
staking. To this end, an Vega token
(or a fraction thereof) can be
- Unlocked: The tokenholder is free to do with the token as they
	want, but cannot delegate it
- Locked: The token is locked in the smart contract, and cen be used
	inside the delegation system
- Delegated: The (locked) token is delegated to a validator
- Undelegated: The token is not delegated to a validator, and can be either
	delegated or unlocked.

## Smart Contract / Staking Bridge Interaction

It is important that no action triggered on Vega needs to directly invoke the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md) through the validators; thus, all actions regarding locking 
and unlocking of stake are initiated by the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md), not by Vega.

In order to delegate, users require tokens that will be locked in a smart contract (see [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md)). Vega will be made aware of how many tokens a given party has locked through bridge events. When the same tokens are unlocked, a corresponding event will be emitted:

```
  event Stake_Deposited(address indexed user, uint256 amount, bytes32 vega_public_key);
  event Stake_Removed(address indexed user, uint256 amount);
```

This provides the information the core needs to keep track of:

	* Total Delegatable Stake
	* Undelegated Stake
	* [n] Stake delegated per validator
	* [n] Stake marked for delegation per validator in the next epoch
	* Total stake (should be the summ of all the others)

There is no interaction with the smart contract that is initiated by Vega.

The validators watch the smart contract, and observe the following actions:

- A token gets locked: This token is now available for delegation
- A token gets unlocked: 
If the token holder has sufficient undelegated tokens, these are used to cover this request (i.e., the available amount of delegatable tokens is reduced to match the (un)locking status). 

This could mean that the token-holder has a delegation-command scheduled that is no longer executable; this command will then be ignored at the start of the next epoch. 

If the token holder does not have sufficient undelegated stake, at first the validators verify if tokens are in the process of being delegated(i.e., the delegation command has been issued, but not yet executed), and uses those tokens to cover the unlocking. If this is insufficient, the `undelegate-now` command is automatically triggered, undelegating evenly from all validators to cover the needs. 

## Delegation Transaction

Any locked and undelegated stake can be delegated at any time by putting a 
delegation-message on the chain. However, the delegation only becomes valid 
towards the next epoch, though it can be undone through undelegate.

Once Vega is aware of locked tokens, the users will have an account with the balance reflecting how many tokens were locked. At this point, the user can submit a transaction to stake (delegate) their tokens. The amount they stake must be `<= balance`, naturally. 

```proto
message Delegate {
	uint256   Amount = 1;
	Validator Val = 2;
}

message Undelegate {
	uint256   Amount = 1;
	Validator Val = 2;
}
```

Where `Delegate` adds the `Amount` to the delegation of validator `Val` at the biginning of the next epoch (if still available to them), and `Undelegate` subtracts the amount from the delegation of `Val` by the next epoch if available.

To avoid fragmentation or spam, there is a system parameter `minimum delegateable stake` that defines the smallest unit of (fractions of) tokens that can be used for delegation.

To delegate stake, a delegator simply puts a command "delegate x stake to y" on
the chain. It is verified at the beginning (when the command is issued and before
it is put on the chain) that the delegator has sufficient unlocked stake, as 
well as in the beginning of the next epoch just before the command takes effect.
The amount of delegatable stake is reduced right away once the command is put into 
a block.

Each validator will have a maximum amount of stake that they can accept as delegation (initially this will be the same for all validators, governed by a network parameter `max stake per validator`). If a participant is delegating such that the size of their stake would cause this amount to be exceeded, then they are only staked up to this maximum amount. The remaining of their stake is therefore eligible to stake to another validator.

### Undelegating
Users can remove stake by submitting an `Undelegate` transaction. The tokens will then be restored back to their token balance.

At the top level, `Stake_Deposited` simply adds `amount` of tokens to the account of the user associated with the `user`. Likewise, the `Stake_Removed` event subtracts the `amount` of tokens from their account.

- If the `Stake_Removed` amount of tokens is higher than the balance of said user, something went seriously wrong somewhere. This is a good time to panic.
- If the amount is higher than the amound of undelegated stake, the missing amount must be freed using the undelegate function (see section above about bridge contract interaction). There is currently no rule how to choose this; 

*Option-1*
A first heuristic would be to take from the highest delegation first and then go down, e.g.
	* If the delegation is 100, 90, 80, 70, and we need to free 30 stake, we first take from the richest ones until they are no longer the richest:
	* Free 10, delegation is 90, 90, 80, 70
	* Free 30, delegation is 80, 80, 80, 70
This has the benefit of lowering the probability that a single withdrawal will leave any one validator with zero delegated stake.

*Option-2*
Another option would be to withdraw stake proportionally from the validators.
	* If the delegation is 100, 90, 80, 70, and we need to free 30 stake, we split the withdrawal across all validators proportionately:
	* Free from delegator-1 (to whom the participant has delegated 100) an amount equal to 30 * (100/(100+90+80+70)) etc. Not sure how to deal with rounding.


#### Types of undelegations

_Undelegate towards the end of the epoch_
- The action is announced in the next available block, but the delegator keeps the delegation alive till the last block of the epoch. The delegator can then re-delegate the stake, which then be valid once the next epoch starts. The delegator cannot move the tokens before the epoch ends, they remain locked.


_Undelegate Now_
`UndelegateNow`:
The action can be announced at any time and is executed immediately following the block it is announced in.
The user is marked to not receive any reward from the validator in that epoch. The reward should instead go into the [on-chain treasury account for that asset](). The stake is marked as free for the delegator, but is not yet removed from the validator stake (this happens at the end of the epoch).

Rationale: This allows a delegator to sell their tokens in a rush, without requiring any interaction between the smart contract and the details of the delegation system. This also allows the delegator to change their mind about a delegation before it is activated.


_optional (not needed now, but later)_
`UndelegateInAnger`:
To unlock any stake fast, this has the same effect as `UndelegateNow`, but the stake is removed from the validator right away (at least as far as voting rights are concerned). The delegator loses the delegated stake and the income with it, as well as their voting weight.
As this is not required for first mainnet, and involves more subtleties (weights need to be recalculated on the fly, there may be a mixture of normal undelegated and undelegate in anger, ...), this feature does not need to be implemented right away for Mainnet alpha.

## Fringe Cases:
A delegator can delegate some stake, and immediatelly undelegate it before the next
epoch starts. This is fine with us.

If the value of `minimum_delegateable_stake` changes in a bad way, stakers might be stuck with
some fraction they can't modify anymore. To this end, the undelegate commands also should
support a parameter "all".

With this setup, a delegator can use a constant delegation/undelegate-now to spam the network.	

If several delegators change the delegation within the same block, some of them may not be allowed to 
execute (as this would exceed the maximum stake the validator wants). To save resources, the
block creator has the responsibility to filter out these transactions.


## Network Parameters
`minimum delegateable stake` - the smallest unit of (fractions of) tokens that can be used for delegation
`max stake per validator` - maximum amount of stake that a validator can accept


## Acceptance Criteria

### Staking for the first time
- To lock tokens, a participant must:
  - Have some balance of vested or unvested governance asset in an Ethereum wallet. These assets must not be locked to another smart contract (including the [Vega collateral bridge]()).
  - Have a Vega wallet
  - Lock the tokens on the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md)
- To delegate the locked tokens, a participant must:
  - Have enough tokens to satisfy the network parameter: "Minimum delegateable stake" 
  - Delegate the locked tokens to one of the eligible validators (fixed set for Alpha mainnet).
- These accounts will be created:
  - A [staking account](./0013-accounts.md#party-staking-accounts) denominated in the governance token asset is created
  - When first fees are received as a staking reward, a general account for each settlement currency (so they can receive infrastructure fee rewards)
  - It is possible that a [separate reward function](./0000-reward-functions.md) will cause an account to be created for the user as a result of rewards.
- Timings
  - Any locked (but undelegated) tokens can be delegated at any time. 
  - The delegation only becomes valid at the next [episode](./0050-epochs.md), though it can be undone through undelegate.
  - The balance of "delegateable stake" is reduced immediately (prior to it coming into effect in the next epoch) 

### Adding more stake
- More tokens may be locked at any time on the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md)
- More stake may be delegated at any time (see [function: Stake](../non-protocol-specs/0004-staking-bridge.md) - amount refers to size by which to increment existing staked amount)
- Same timings apply as per staking for the first time

### Removing stake
- Any stake may be withdrawn from the [Vega staking bridge contract](../non-protocol-specs/0004-staking-bridge.md) at any time
 - Unlocking your tokens in the bridge contract will effectively "remove" them from any delegation they're doing (unless you have remaining undelegated tokens that could fulfil your delegation)
- Delegation may be fully or partially removed. The amount specified in the [function: Remove](../non-protocol-specs/0004-staking-bridge.md) - is the size by which the existing staked amount will be decremented
- Removal of delegation may happen in the following 2 ways:
  - Announcing removal, but maintaining stake until last block of the current epoch. This "announced stake" may be then (re)delegated (e.g. to a different validator).
  - Announcing removal and withdrawing stake immediately. Rewards are still collected for this stake until the end of the epoch, but they are sent to the onchain treasury account for that asset.

### Changing delegation
- Changing the validator to whom a participant wants to validate to involves:
  - Announcing removal of stake for current validator
  - Staking on the new validator, as per normal [function: Stake](../non-protocol-specs/0004-staking-bridge.md)
  - These can happen concurrently, so that at the next epoch, the stake is removed from the current validator and staked on the new validator 