# Setting fees and rewarding market makers

## Summary

The aim of this specification is to set out how fees on Vega are set based on committed market making stake and prevailing open interest on the market. Let us recall that market makers can commit and withdraw stake by submitting / amending a special market making pegged order type [market maker order spec](????.md). 

## Definitions / Glossary of terms used
- **Open interest**: the volume of all open positions in a given market (ie order book)
- **Liquidity**: measured as per [liquidity measurement spec](0034-prob-weighted-liquidity-measure.ipynb) (but it's basically volume on the book weighted by the probability of trading)
- **Supplied liquidity**: this counts only the liquidity provided through the special market making order that market makers have committed to as per [market maker order spec](????.md) 
- **Liquidity demand window length `t_liquidity_window`**: sets the length of the window over which we estimate liquidity demand for fee setting purposes. This is a network parameter.  
- **Liquidity demand estimate**: as defined in [liquidity demand estimate spes](????-liquidity-demand-estimate.md) using `t_fee_liquidity_window` which is a network paratemer. 
- **Sufficient liquidity trigger `c_2`**: a network parameter `c_2` defined in [liquidity monitoring](????-liquidity-monitoring.md) spec. 
- **Market value estimate** is calculated to be the estimated fee income for the entire future existence of the market using recent fee income. See further in this spec for details.


## Calculating market fees

As part of the [commit liquidity network transaction ](????-mm-mechanics.md), the market maker submits their desired fee level for the market. Here we describe how the market fee is set from the values submitted by all market makers for a given market. 
First, we produce a list of pairs which capture committed liquidity of each mm together with their desired fee and arrange this list in an increasing order by fee amount. Thus we have 
```
[MM 1 liquidity, MM 1 fee]
[MM 2 liquidity, MM 2 fee]
...
[MM N liquidity, MM N fee]
```
where `N` is the number of market makers who have committed to supply liquidity to this market. Note that `MM 1 fee <= MM 2 fee <= ... <= MM N fee` because we demand this list of pairs to be sorted. 

We now find smallest integer `k` such that `c_2 x [liquidity demand estimate] < sum from i=1 to k of [MM liquidity i]`. In other words we want in this ordered list to find the market makers that supply the liquidity that's required. If no such `k` exists we set `k=N`.

Finally, we set the fee for this market to be the fee `MM k fee`. 

### Example for fee setting mechanism
Let us say that `c_2 = 10`. 
``` 
[MM 1 liquidity = 120, MM 1 fee = 0.5%]
[MM 2 liquidity = 20, MM 2 fee = 0.75%]
[MM 3 liquidity = 60, MM 3 fee = 3.75%]
```
1. If the `liquidity demand estimate = 10` then `c_2 x [liquidity demand estimate] = 100` which means that the needed liquidity is given by MM 1, thus `k=1` and so the market fee is  `MM 1 fee = 0.5%`. 
1. If the `liquidity demand estimate = 12.5` then `c_2 x [liquidity demand estimate] = 125` which means that the needed liquidity is given by MM 1 and MM 2, thus `k=2` and so the market fee is  `MM 2 fee = 0.75%`. 
1. If the `liquidity demand estimate = 123` then `c_2 x [liquidity demand estimate] = 1230` which means that even putting all the liquidity supplied above does not meet the estimated market liquidity demand and thus we set `k=N` and so the market fee is `MM N fee = MM 3 fee = 3.75%`. 
1. Initially (before market opened) the `[liquidity demand estimate]` is by definition zero (it's not possible to have a position on a market that's not opened yet). Hence by default the initial fee is the lowest fee.

## Timing market fee changes

Once the market opens (opening auction starts) a clock starts ticking. We calculate the `[liquidity demand estimate]` using [liquidity demand estimate spes](????-liquidity-demand-estimate.md) with the network parameter `t_fee_liquidity_window`. The fee is continuosly re-evalueated using the mechanism above. 

## Calculating market value proxy

This will be used for determining what "equity like share" does commiting market making stage at a given time lead to. 
It's calculated, with `t` denoting time now, as follows:
```
total_stake = sum of all mm stakes
active_time_window = [max(t-t_fee_liquidity_window,0), t]

traded_value_over_window = total trade value for fee purposes of all trades executed on a given market the active_time_window

market_value_proxy = max(total_stake, traded_value_over_window)
```

Note that trade value for fee purposes is provided by each instrument, see [fees][0024-fees.md]. For futures it's just the notional and in the examples below we will only think of futures. 


### Example
1. The market was just proposed and one MM commited stake. No trading happened so the `market_value_proxy` is the stake of the committed MM. 
1. A MM has committed stake of `10000 ETH`. The traded notional over `active_time_window` is `9000 ETH`. So the `market_value_proxy` is `10000 ETH`.
1. A MM has committed stake of `10000 ETH`. The traded notional over `active_time_window` is `250 000 ETH`. Thus the `market_value_proxy` is `250 000 ETH`.

## Calculating market maker equity-like share

The guiding principle of this section is that by commiting stake a market maker buys a portion of the `market_value_proxy` of the market. 

At any time let's say we have `market_value_proxy` calculated above and existing market makers with equity-like share as below
```
[MM 1 eq share, MM 1 ownership amt]
[MM 2 eq share, MM 2 ownership amt]
...
[MM N eq share, MM 3 ownership amt]
```
Here `MM i ownership amt = [MM i equity share] x market_value_proxy`. 

If a new market maker `MM N+1` who commits stake (by sucessfully submitting a MM order) equal to `S` then purchases a `MM N+1 eq share = S / (market_value_proxy + S)`. 
This "dilutes" the equity-like share of existing MMs as follows: 
```
New MM i eq share = (1 - [MM N+1 eq share]) x [MM i eq share].
```

**Check** the sum from over `i` from `1` to `N` of `New MM i` is equal to `1`.

If an existing market maker, say, without loss of generality, MM 1 as in the above list there is no order, wishes to reduce their MM stake `S` to a lower amount `0 <= New S < S` then we adjust every MM's `eq share` as follows. 
1. Calculate the MM 1 reduction factor `f=[New S] / S`. 
1. Calculate the rescaling factor `r = 1/(1-[MM 1 eq share] x (1-f))`
1. For MM 1 let `New MM 1 eq share = r x f x [MM 1 eq share]`.
1. For `i` in `2` to `N` update `New MM i eq share = r x [MM i eq share]`. 

**Check** the sum from over `i` from `1` to `N` of `New MM i` is equal to `1`.


## Distributing fees
The fees are collected into a per-market "bucket" belonging to market makers for that market. We will create a new network parameter (which can be 0 in which case fees are transferred immediately) called `market_maker_fee_distribition_time_step` which will define how frequently fees are distributed from the per-market "bucket" belonging to market makers to their margin accounts for the market. 

The fees are distributed pro-rata depending on the `MM i eq share` at a given time. 

### Example
The fee bucket contains `103.5 ETH`. We have `3` MMs with equity shares:
share as below
```
MM 1 eq share = 0.65
MM 2 eq share = 0.25
MM 3 eq share = 0.1
```
When the time defined by ``market_maker_fee_distribition_time_step` elapses we do transfers:
```
0.65 x 103.5 = 67.275 ETH to MM 1's margin account
0.25 x 103.5 = 25.875 ETH to MM 2's margin account
0.10 x 103.5 = 10.350 ETH to MM 3's margin account
```