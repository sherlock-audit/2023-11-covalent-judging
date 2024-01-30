Fast Purple Rabbit

medium

# OperationalStaking may not possess enough CQT for the last withdrawal

## Summary
Both `_sharesToTokens` and `_tokensToShares` round down instead of rounding off against the user. This can result in users withdrawing few weis more than they should, which in turn would make the last CQT transfer from the contract revert due to insufficient balance.

## Vulnerability Detail
1. When users `stake`, the shares they will receive is calculated via `_tokensToShares`:
```solidity
    function _tokensToShares(
        uint128 amount,
        uint128 rate
    ) internal view returns (uint128) {
        return uint128((uint256(amount) * DIVIDER) / uint256(rate));
    }
```
So the rounding will be against the user, or zero if the user provided the right amount of CQT.

2. When users unstake, their shares are decreased by 

```solidity
    function _sharesToTokens(
        uint128 sharesN,
        uint128 rate
    ) internal view returns (uint128) {
        return uint128((uint256(sharesN) * uint256(rate)) / DIVIDER);
    }
```
So it is possible to `stake` and `unstake` such amounts, that would leave dust amount of shares on user's balance after their full withdrawal. However, dust amounts can not be withdrawn due to the check in _redeemRewards:
```solidity
        require(
            effectiveAmount >= REWARD_REDEEM_THRESHOLD,
            "Requested amount must be higher than redeem threshold"
        );
```
But, if the user does not withdraw immediately, but instead does it after the multiplier is increased, the dust he received from rounding error becomes withdrawable, because his `totalUnlockedValue` becomes greater than `REWARD_REDEEM_THRESHOLD`. 

So the user will end up withdrawing more than their `initialStake + shareOfRewards`, which means, if the rounding after all other operations stays net-zero for the protocol, there won't be enough CQT for the last CQT withdrawal (be it `transferUnstakedOut`, `redeemRewards`, or `redeemCommission`).

[Foundry PoC](https://gist.github.com/aslanbekaibimov/e0962c60213ac460c8ea1c3b013e5537)

## Impact

Victim's transactions will keep reverting unless they figure out that they need to decrease their withdrawal amount.

## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L386-L388

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L393-L395
## Tool used

Manual Review

## Recommendation
`_sharesToTokens` and `_tokensToShares`, instead of rounding down, should always round off against the user.