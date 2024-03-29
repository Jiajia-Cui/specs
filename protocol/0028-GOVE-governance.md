# Governance

Governance allows the vega network to arrive at on-chain decisions. Implementing this specification will provide the ability for users to create proposals involving assets, markets, network parameters and free form text.

This is achieved by creating a simple protocol framework for the creation, approval/rejection, and enactment (where appropriate) of governance proposals.

To implement this framework, two new transactions must be supported by the Vega core:
 - Submit Proposal: deploy a new (valid) proposal to the network
 - Vote: record a vote for or against a live proposal

In this document, a "user" refers to a "party" (private key holder) on a Vega network.


# Guide-level explanation

Governance actions can be the end result of a passed proposal. The allowable types of change to be proposed are known as "governance actions". In the future, enactment of governance actions may also be possible by other means (for example, automatically by the protocol in response to certain conditions), which should be kept in mind during implementation.

The types of governance action are:

1. Create a new market
2. Change an existing market's parameters
3. Change network parameters
4. Add an external asset to Vega (covered in a [separate spec - see 0027](./0027-ASSP-asset_proposal.md))
5. Authorise a transfer to or from the [Network Treasury](./0055-TREA-on_chain_treasury.md)
6. Freeform proposals

## Lifecycle of a proposal

Note: there are some differences/additional points for market creation proposals, see the section on market creation below.

1. Governance proposal is accepted by the network as a transaction.
1. The nodes validate the proposal. Note: this is where the network parameters that validate the minimum duration, minimum time to enactment (where appropriate), minimum participation rate, and required majority are evaluated. The proposal is not revalidated. This is also where, if not specified on the proposal, the required participation rate and majority for success are defined and copied to the proposal. The proposal is immutable once entered and future parameter changes don't impact it (this is to prevent surprising behaviour where other proposals with as yet unknown outcomes can impact the success of a proposal).
1. If valid, the proposal is considered "active" for a proposal period. This period is defined on the proposal and must be at least as long as the minimum duration for the proposal type/subtype (specified by a network parameter)
1. During the proposal period, network participants who are eligible to vote on the proposal may submit votes for or against the proposal.
1. When the proposal period closes, the network calculates the outcome by:
    - comparing the total number of votes cast as a percentage of the number eligible to be cast to the minimum participation requirement (if the minimum is not reached, the proposal is rejected)
		- comparing the number of positive votes as a percentage of all votes cast (maximum one vote counted per party) to the required majority. 
1. If the required majority of "for" votes was met and the proposal has a governance action defined with it, the action described in the proposal will be taken (proposal is enacted) on the enactment date, which is defined by the proposal and must be at least the minimum enactment period for the proposal type/subtype (which is specified by a network parameter) _after_ voting on the proposal closes.

Any actions that result from the outcome of the vote are covered in other spec files.

## Governance Asset
The Governance Asset is the on-chain [asset](./0040-ASSF-asset_framework.md) representing the [token configured in the staking bridge](../non-protocol-specs/0006-erc20-governance-token-staking.md). Users with a staking account balance in the governance asset can:

- [Create proposals](#restriction-on-who-can-create-a-proposal)
- [Vote on proposals](#voting-for-a-proposal)
- [Delegate to validators](./0059-STKG-simple_staking_and_delegating.md)

## Governance weighting
A party on the Vega network will have a weighting for each type of proposal that determines how strongly their vote counts towards the final result. 

To submit a proposal the party has to have more (strictly greater) than a minimum set by a network parameter `governance.proposal.market.minProposerBalance` deposited on the Vega network (the network parameter sets the number of tokens). The minimum valid value for this parameter is `0`. 

Weighting will initially be determined by the sum of the locked and staked token balances on the [staking bridge](../non-protocol-specs/0004-staking-bridge.md).

In future, governance weighting for some proposal types will be based on alternative measures, such as:

1. The amount of market making bond that a participant has placed with the network for a specific market, or in total.
1. The value of some other internally calculated number specific to a participant (e.g. the size of their open positions on a particular market). See note below.

The governance system must be generic in term of weighting of the vote for a given proposal. As noted above, the first implementation will start with _the amount of a particular token that a participant holds_ but this will be extended in the near future, as additional protocol features and governance actions are added.

Initially the weighting will be based on the amount of the configured governance asset that the user has on the network as determined *only* by their staking account balance of this asset. 1 token represents 1 vote (0.0001 tokens represents 0.0001 votes, etc.). A user with a balance of 0 cannot vote or submit a proposal of that type, and ideally this would be enforced in a check _before_ scheduling the voting transaction in a block.

The governance token used for calculating voting weight must be an asset that is configured within the asset framework in Vega (this could be a "Vega native" asset on some networks or an asset deposited via a bridge, i.e. an ERC20 on Ethereum). Note: this means that the asset framework will _always_ need to be able to support pre-configured assets (the configuration of which must be verifiably the same on every node) in order to bootstrap the governance system. The governance asset configuration will be different on different Vega networks, so this cannot be hard coded.

Note: in the future, some or all proposals for changes to a market will be weighted by a measure of participation in that market. The most likely way this would be calculated would be by the size of the voter's market making commitment or vs. the total committed in the market (and participation ratios would be calculated from the same), although we may also consider metrics like the voter's share of traded volume over, say, the voting period or some other algorithm. _Importantly this means a voter's weighting will vary between markets for these types of proposal._


## Voting for a proposal

Users of the vega platform will be able to vote for or against a proposal, if they have an eligible (non-zero) voting weight. A user may choose whether or not to vote. If a user votes, the action is binary: they may either vote **for the proposal** or **against the proposal**, and this will apply to their full weighting.

A user can vote as many times as needed, only the last vote will be accounted for in the final decision for the proposal. We do not consider prevention of spam/DOS attacks by multiple voting in this spec, though they will need to be covered (potentially by a fee and/or proof of work cost).

The amount of voting weight that a user is considered to be voting with is the full amount they hold, as measured by the network, **at the conclusion of the proposal period** - as part of calculating the vote outcome. For example, if a user votes "yes" for a proposal and then adds to or withdraws from (including via movements to and from margin accounts for trading the asset) their governance token balance after submitting their vote and prior to the end of the proposal period, their new balance of voting asset is the one used. (Note: this may change in future, if it is deemed to allow misleading or exploitative voting behaviour. Particularly, we may lock the balance from being withdrawn or used for trading for the duration of the vote, once a participant has voted.)


## Restriction on who can create a proposal

Anyone can create a proposal if the weighting of their vote on the proposal would be >0 (e.g. if they have more than 0 of the relevant governance token).

In a future iteration of the governance system we may restrict proposal submission by type of proposal based on a minimum weighting. e.g: only user with a certain number or percentage of the governance asset are allowed to open a "network parameter change" proposal.

Market change proposals additionally require certain minimum [equity like share](0042-LIQF-setting_fees_and_rewarding_lps.md) set by `governance.proposal.market.minEquityLikeShare`.


## Configuration of a proposal

When a proposal is created, it can be configured in multiple ways. 


### Duration of the proposal

A new proposal will have a close date specified as a timestamp. After the proposal is created in the system and before the close date, the proposal is open for votes. e.g: A proposal is created and people have 3 weeks from the day it is sent to the network in order to submit votes for it.

The proposal's close date may optionally be set by the proposer and must be greater than or equal to a minimum duration time that is set by the network. Minimum duration times will be specified as network parameters depending on the type of proposal. 

The network's _minimum proposal duration_ - as specified by a network parameter specific to each proposal type - is used as the default when the new proposal does not include a proposal duration. If a proposal is submitted with a close date would fail to meet the network's minimum proposal duration time constraint, the proposal must be rejected.


### When a proposal is enacted

Note: market creation proposals are handled slightly differently, see below. Freeform proposals are never enacted.

A new proposal that contains a governance action can specify when any changes resulting from a successful vote would start to be applied. e.g: A new proposal is created in order to create a new market with an enactment date 1 week after vote closing. After 3 weeks the proposal is closed (the duration of the proposal), and if there are enough votes to accept the new proposal, then the changes will be applied in the network 1 week later.

This allows time for users to be ready for changes that may effect them financially, e.g a change that might increase capital requirements for positions significantly and thus could trigger close-outs. It also allows markets to be pre-approved early and launched at a chosen time in the future.

Proposals are enacted by timestamp, earliest first, as soon as the enactment time is reached by the network (i.e. "Vega time"). Proposals sharing the same exact enactment time are enacted in the order they were created. This means that in the case that two proposals change the same parameter with the same timestamp, the oldest proposal will be applied first and the newest will be applied last, overwriting the change made by the older proposal. There is no attempt to resolve differences between the two.

The network's `governance.proposal.*.minEnact` network parameter specific to each proposal type is used to validate whether the enactment date is acceptable. 
Here `*` stands for any of `asset, market, updateMarket, updateNetParam`. 
Note that this is validation is in units of time from current time i.e. if the proposal is received 
at e.g. `09:00:00 on 1st Jan 2021` and `governance.proposal.asset.minEnact` is `72h` then the proposal must contain enactment date/time that after `09:00:00 on 4th Jan 2021`. 
If there is `governance.proposal.asset.maxEnact` of e.g. `360h` then the proposed enactment date / time must be before `09:00:00 on 16th Jan 2021`.


## Editing and/or cancelling a proposal is not possible

A proposal cannot be edited, once created. The only possible action is to vote for or against a proposal, or submit a new proposal. 

If a proposal is created and later a different outcome is preferred by network participants, two courses of action are possible:

1. Vote against the proposal and create a new proposal with the correct change
1. Vote for or against the proposal and create a new proposal for the additional change

Which of these makes most sense will depend on the type of change, the timing of the events, and how the rest of the community votes for the initial proposal.


## Outcome

At the conclusion of the voting period the network will calculate two values:

1. The participation rate: `participation_rate = SUM ( weightings of ALL valid votes cast ) / max total weighting possible` (e.g. sum of token balances of all votes cast / total supply of governance asset, this implies that for this version it is only possible to use an asset with **fixed supply** as the governance asset)
1. The "for" rate: `for_rate = SUM ( weightings of votes cast for ) / SUM ( weightings of all votes cast )`

Any proposal that is not market parameter change proposal is considered successful and will be enacted (where necessary) if:

- The `participation_rate` is greater than or equal to the minimum participation rate for the proposal
- The `for_rate` is greater than or equal to the minimum required majority for the proposal
- The `participation rate` is calculated against the *total supply of the governance asset*.

Note: see below for details on minimum participation rate and minimum required majority, which are defined by type of governance action, and in some cases a category or sub-type.

Not in scope: minimum participation of active users, i.e. 90% of the _active_ users of the vega network have to take part in the vote. Minimum participation is currently always measured against the total possible participation.

For market change proposals the network will additionally calculate 
1. `LP participation rate = SUM (equity like share of all LPs who cast a vote)` (no need to divide by anything as equity like share sums up to `1`).
1. `LP for rate = SUM (all who voted for) / LP participation rate`. 

A market parameter change is passed only when:
- either the governance token holder vote is successful i.e. `participation_rate >= governance.proposal.updateMarketParam.requiredParticipation` AND `for_rate > governance.proposal.updateMarketParam.requiredMajority` (in this case the LPs were overridden by governance token holders)
- or the governance token holder vote `participation_rate < governance.proposal.updateMarketParam.requiredParticipation` AND `LP participation rate >= governance.proposal.updateMarketParam.requiredParticipationLP` AND `LP for rate >= governance.proposal.updateMarketParam.requiredMajorityLP`.

In all other cases the proposal is rejected.

In other words: LPs vote with their equity like share and can make changes to a market without requiring a governance token holder vote. However a governance token vote is running in parallel and if participation and majority rules for this vote are met then the governance token vote can overrule the LPs vote.  


# Reference-level explanation

We introduce 2 new commands which require consensus (needs to go through the chain)

- submit a proposal.
- vote for a given proposal.


## Types of proposals

## 1. Create market

This action differs from from other governance actions in that the market is created and some transactions (namely around liquidity provision) may be accepted for the market before the proposal has successfully passed. The lifecycle of a market and its triggers are covered in the [market lifecycle](./0043-MKTL-market_lifecycle.md) spec.

Note the following key points from the market lifecycle spec:
* A market is created in Proposed status as soon as the proposal is accepted
* A market enters a Pending status as soon as the proposal is Successful (before enactment)
* A market usually enters Active status at the proposal's enactment date/time, but some conditions may delay this or cause the market to be Cancelled instead

A proposal to create a market contains 
1. a complete market specification as per the Market Framework (see spec) that describes the market to be created. 
1. a liquidity provision commitment via LP commitment data structure, specifying stake amount, fee bid, plus buy and sell shapes [see lp-mechanics](./0044-LIQM-lp_mechanics.md). The proposal must be rejected if the liquidity provision commitment is invalid or the proposer does not have the required collateral for the stake.
The stake commitment must exceed the `minimum_proposal_stake_amount` which is a per-asset parameter.
1. an enactment time that is at least the *minimum auction duration* after the vote closing time (see [auction spec](./0026-AUCT-auctions.md))

All **new market proposals** initially have their validation configured by the network parameters `Governance.CreateMarket.All.*`. These may be split from `All` to subtypes in future, for instance when other market types like RFQ are created.


## 2. Change market parameters

[Market parameters](./0001-MKTF-market_framework.md#market) that may be changed are described in the spec for the Market Framework, and additionally the specs for the Risk Model and Product being used by the market. 
See the [Market Framework spec](./0001-MKTF-market_framework.md#market) for details on these parameters, including those that cannot be changed and the category of the parameters.

To change any market parameter the proposer submits the same data as to create a market with the desired updates to the fields / structures that should change. 
Ideally, it should be possible to not repeat things that are not changing or are immutable but we leave this to implementation detail.

The following are immutable and cannot be changed:
- marketID
- tickSize (will be removed I believe so it's not relevant)
- decimalPlaces
- settlementAsset
- name


## 3. Change network parameters

[Network parameters](./0054-NETP-network_parameters.md) that may be changed are described in the *Network Parameters* spec, this document for details on these parameters, including the category of the parameters. New network parameters require a code change, so there is no support for adding new network parameters.

All **change network parameter proposals** have their validation configured by the network parameters `Governance.UpdateNetwork.<CATEGORY>.*`, where `<CATEGORY>` is the category assigned to the parameter in the Network Parameter spec.

## 4. Add a new asset

New [assets](./0040-ASSF-asset_framework.md) can be proposed through the governance system. The procedure is covered in detail in the [asset proposal spec](./0027-ASSP-asset_proposal.md)). Unlike markets, assets cannot be updated after they have been added.

## 5. Transfers initiated by Governance (post Oregon trail)
### Permitted source and destination account types

The below table shows the allowable combinations of source and destination account types for a transfer that's initiated by a governance proposal. 

| Source type | Destination type | Governance transfer permitted |
| --- | --- | --- |
| Party account (any type) | Any | No |
| Network treasury | Reward pool account | Yes  |
| Network treasury | Party general account(s) | Yes |
| Network treasury | Party other account types | No |
| Network treasury | Network insurance pool account | Yes |
| Network treasury | Market insurance pool account | Yes |
| Network treasury | Any other account | No |
| Network insurance pool account | Network treasury | Yes |
| Network insurance pool account | Market insurance pool account | Yes |
| Network insurance pool account | Any other account | No |
| Market insurance pool account | Party account(s) | Yes  |
| Market insurance pool account | Network treasury | Yes  |
| Market insurance pool account | Network insurance pool account | Yes |
| Market insurance pool account | Any other account | No |
| Any other account | Any | No | 


### Transfer proposal details

The proposal specifies:

- `source_type`: the source account type (i.e. network treasury, network insurance pool, market insurance pool)
- `source` specifies the account to transfer from, depending on the account type:
  - network treasury: leave blank (only one per asset)
  - network insurance pool: leave blank (only one per asset)
  - market insurance pool: market ID
- `type`, which can be either "all or nothing" or "best effort":
	- all or nothing: either transfers the specified amount or does not transfer anything
  - best effort: transfers the specified amount or the max allowable amount if this is less than the specified amount
- `amount`: the maximum amount to transfer
- `asset`: the asset to transfer
- `fraction_of_balance`: the maximum fraction of the source account's balance to transfer as a decimal (i.e. 0.1 = 10% of the balance)
- `destination_type` specifies the account type to transfer to (reward pool, party, network insurance pool, market insurance pool)
- `destination` specifies the account to transfer to, depending on the account type:
  - reward pool: the reward scheme ID
  - party: the party's public key
  - network insurance pool: leave blank (there's only one per asset)
  - market insurance pool: market ID
- Plus the standard proposal fields (i.e. voting and enactment dates, etc.)


### Transfer proposal enactment

If the proposal is successful and enacted, the amount will be transferred from the source account to the destination account on the enactment date.

The amount is calculated by
```
  transfer_amount = min( 
    proposal.fraction_of_balance * source.balance, 
    proposal.amount, 
    NETWORK_MAX_AMOUNT,
    NETWORK_MAX_FRACTION * source.balance )
```

Where:
-  NETWORK_MAX_AMOUNT is a network parameter specifying the maximum absolute amount that can be transferred by governance for the source account type
-  NETWORK_MAX_FRACTION is a network parameter specifying the maximum fraction of the balance that can be transferred by governance for the source account type (must be <= 1)

If `type` is "all or nothing" then the transfer will only proceed if:

```
transfer_amount == min( 
    proposal.fraction_of_balance * source.balance, 
    proposal.amount )
```

## 6. Freeform governance proposal

The aim of this is to allow community to provide votes on proposals which don't change any of the behaviour of the currently running Vega blockchain. That is to say, at enactment time, no changes are effected on the system, but the record of how token holders voted will be stored on chain. Freeform proposals contain a URL to text describing the proposal in full. The proposal will contain:
- a link to a text file in markdown format and 
- a cryptographically secure hash of the text so that viewers can check that the text hasn't been changed since the proposal was submitted and
- a description field to show a short title / something in case the link goes offline. This is to be between `0` and `255` unicode characters.

The protocol (Vega core) is not expected to verify that the hash corresponds to the contents of the linked file. It is expected that any client tool that allows voting will do this at client level. 

The following network parameters will decide how these proposals are treated: 
`governance.proposal.freeform.maxClose` e.g. `720h`,
`governance.proposal.freeform.minClose` e,g. `72h`,
`governance.proposal.freeform.minProposerBalance` e.g. `1000000000000000000` i.e. 1 VEGA,
`governance.proposal.freeform.minVoterBalance`   e.g. `1000000000000000000` i.e. 1 VEGA,
`governance.proposal.freeform.requiredMajority`  e.g. `0.66`,
`governance.proposal.freeform.requiredParticipation` e.g. `0.20`.
      
There is no `minEnact` and `maxEnact` because there is no on-chain enactment (no governance action).

## Proposal validation parameters

As described throughout this specification, there are several sets of network parameters that control the minimum durations of the voting and pre-enactment periods, as well as the minimum participation rate and required majority for a proposal.

These sets of parameters are named in the form `Governance.<ActionType>.<Category>.*`, i.e.

* `Governance.<ActionType>.<Category>.MinimumProposalPeriod`
* `Governance.<ActionType>.<Category>.MinimumPreEnactmentPeriod`
* `Governance.<ActionType>.<Category>.MinimumRequiredParticipation` 
* `Governance.<ActionType>.<Category>.MinimumRequiredMajority`


See the details in 1-3 above for the action type and category (or references to where to find them). For example, for market creation the parameters are as below (and for updating market and network parameters, there are multiple sets of these by category):

* `Governance.CreateMarket.All.MinimumProposalPeriod`
* `Governance.CreateMarket.All.MinimumPreEnactmentPeriod`
* `Governance.CreateMarket.All.MinimumRequiredParticipation` 
* `Governance.CreateMarket.All.MinimumRequiredMajority`


Notes:

* The categorisation of parameters is liable to change and be added to as the protocol evolves.
* As these are themselves network parameters, a set of parameters will control these parameters for the actions that update these parameters (including being self-referential), i.e. the parameter `Governance.UpdateNetwork.GovernanceProposalValidation.MinimumRequiredParticipation` would control the amount of voting participation needed to change these parameters. See the Network Parameters spec.


## APIs

The core should expose via core APIs:
 - all the active proposals on the network
 - the current results for an active proposal or a proposal awaiting enactment

APIs should also exist for clients to:
 - list all proposals including historic ones, filter by status/type, sort either way by submission date, vote closing date, or enactment date
 - retrieve the summary results and status for any proposal
 - retrieve the party IDs (pub keys) of all votes counting for (i.e. only one latest vote per party) and against a proposal
 - retrieve the full voting history for a proposal including where a party voted multiple times
 - get a list of all proposals a party voted on

# Acceptance Criteria

- [x] As a user, I can create a new proposal, assuming my staking balance matches or exceeds `minProposerBalance` network param for my proposal type (<a name="0028-GOVE-001" href="#0028-GOVE-001">0028-GOVE-001</a>)
- [x] As a user, I can list the open proposals on the network (<a name="0028-GOVE-002" href="#0028-GOVE-002">0028-GOVE-002</a>)
- [ ] As a user, I can get a list of all proposals I voted for (<a name="0028-GOVE-003" href="#0028-GOVE-003">0028-GOVE-003</a>)
- [x] As a user, I can receive notification when a new proposal is created and may require attention. (<a name="0028-GOVE-004" href="#0028-GOVE-004">0028-GOVE-004</a>)
- [x] As the vega network, all votes from eligible users for an existing proposal are accepted when the proposal is still open (<a name="0028-GOVE-005" href="#0028-GOVE-005">0028-GOVE-005</a>)
- [x] As the vega network, all votes received before the proposal is [active](#lifecycle-of-a-proposal), or once the proposal voting period is finished, are *rejected* (<a name="0028-GOVE-006" href="#0028-GOVE-006">0028-GOVE-006</a>)
- [x] As the vega network, once the voting period is finished, I validate the result based on the parameters of the proposal used to decide the outcome of it. (<a name="0028-GOVE-007" href="#0028-GOVE-007">0028-GOVE-007</a>)
- [ ] As the vega network, proposals that close less than 2 days from enactment are rejected as invalid (<a name="0028-GOVE-009" href="#0028-GOVE-009">0028-GOVE-009</a>)
- [ ] As the vega network, proposals that close more/less than 1 year from enactment are rejected as invalid (<a name="0028-GOVE-010" href="#0028-GOVE-010">0028-GOVE-010</a>)
- [ ] As a user, I can vote for an existing proposal if I have more than 0 governance tokens in my staking account, assuming it matches or exceeds the `minVoterBalance` network parameter for the proposal type (<a name="0028-GOVE-014" href="#0028-GOVE-014">0028-GOVE-014</a>)
- [ ] As a user, my vote for an existing proposal is rejected if I have 0 governance tokens in my staking account (<a name="0028-GOVE-015" href="#0028-GOVE-015">0028-GOVE-015</a>)
- [ ] As a user, my vote for an existing proposal is rejected if I have 0 governance tokens in my staking account even if I have more than 0 governance tokens in my general or margin accounts (<a name="0028-GOVE-016" href="#0028-GOVE-016">0028-GOVE-016</a>)
- [ ] As a user, I can vote multiple times for the same proposal if I have more than 0 governance tokens in my staking account
  - [x] Only my most recent vote is counted (<a name="0028-GOVE-017" href="#0028-GOVE-017">0028-GOVE-017</a>)
- [ ] When calculating the participation rate of an auction, the participation rate of the votes takes in to account the total supply of the governance asset. (<a name="0028-GOVE-018" href="#0028-GOVE-018">0028-GOVE-018</a>)


## Governance proposal types
### New Market proposals
- [x] As the vega network, if a proposal is accepted and the duration required before change takes effect is reached, the changes are applied (<a name="0028-GOVE-008" href="#0028-GOVE-008">0028-GOVE-008</a>)
- [x] New market proposals must contain a Liquidity Commitment (<a name="0028-GOVE-011" href="#0028-GOVE-011">0028-GOVE-011</a>)

### Market change proposals
- [ ] As the vega network, if a proposal is accepted and the duration required before change takes effect is reached, the changes are applied (<a name="0028-GOVE-008" href="#0028-GOVE-008">0028-GOVE-008</a>)
- [ ] Verify that a market change proposal gets enacted if enough LPs participate and vote for. (<a name="0028-GOVE-009" href="#0028-GOVE-009">0028-GOVE-009</a>)
- [ ] Verify that a market change proposal does *not* get enacted if enough LPs participate and vote for *BUT* governance tokens holders participate beyond threshold and vote against (majority not reached). (<a name="0028-GOVE-010" href="#0028-GOVE-010">0028-GOVE-010</a>)
- [ ] Verify that an enacted market change proposal that doubles the risk model volatility sigma leads to increased margin requirement for all parties. (<a name="0028-GOVE-011" href="#0028-GOVE-011">0028-GOVE-011</a>)
- [ ] Verify that an enacted market change proposal that changes trading terminated oracle and price settlement oracle can be settled using transactions from the new oracle keys. (<a name="0028-GOVE-012" href="#0028-GOVE-012">0028-GOVE-012</a>)
- [ ] Verify that an enacted market change proposal that changes price monitoring bounds enters a price monitoring auction upon the *new* bound being breached (<a name="0028-GOVE-013" href="#0028-GOVE-013">0028-GOVE-013</a>)
- [ ] Verify that an enacted market change proposal that reduces `targetStakeParameters.timeWindow` leads to a reduction in target stake if recent open interest is less than historical open interest (<a name="0028-GOVE-014" href="#0028-GOVE-014">0028-GOVE-014</a>)




### Network parameter change proposals
- [x] As the vega network, if a proposal is accepted and the duration required before change takes effect is reached, the changes are applied (<a name="0028-GOVE-008" href="#0028-GOVE-008">0028-GOVE-008</a>)
- [x] Network parameter change proposals can only propose a change to a single parameter (<a name="0028-GOVE-013" href="#0028-GOVE-013">0028-GOVE-013</a>)

### Freeform governance proposals
- [ ] A freeform governance proposal with a description field that is empty, or not between 0 and 255 characters, will be rejected (<a name="0028-GOVE-019" href="#0028-GOVE-019">0028-GOVE-019</a>)
- [ ] A freeform governance must contain a hash field and it must not be null, but no other check is done to verify it (<a name="0028-GOVE-020" href="#0028-GOVE-020">0028-GOVE-020</a>)
- [ ] A freeform governance must contain a link field and it must not be null, but no other check is done to verify it (<a name="0028-GOVE-021" href="#0028-GOVE-021">0028-GOVE-021</a>)
- [ ] A freeform governance proposal does not have an enactment period set, and after it closes no action is taken on the system (<a name="0028-GOVE-022" href="#0028-GOVE-022">0028-GOVE-022</a>)
- [ ] Closed freeform governance proposals can be retrieved from the API along with details of how tokenholders voted. (<a name="0028-GOVE-023" href="#0028-GOVE-023">0028-GOVE-023</a>)

