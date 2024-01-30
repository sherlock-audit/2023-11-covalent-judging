Faint Licorice Barracuda

medium

# OperationalStaking::_unstake Delegators can bypass 28 days unstaking cooldown when enough rewards have accumulated

## Summary
When rewards are distributed, they are distributed evenly accross all of the shares held for a validator. However participants collecting rewards burn shares, and thus receive less rewards in subsequent rounds, compared to participants which have left rewards in the contract. This means that delegators can unstake instead of redeeming rewards, which will progressively replace the initial staked amount with claimable rewards.

## Vulnerability Detail
We can see in the function `redeemRewards` that the equivalent amount of shares is burned to redeem:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L616-L631

This means that by redeeming rewards a validator will receive less of the future rewards than the stake he invested.

Conversely, this means that delegators can progressively derisk their staking position by burning small quantity of shares from their stake (by calling `unstake`) instead of claiming rewards. When enough rewards accumulated, they have the equivalent of their initial amount in the contract, but it is less risky, since as rewards, they are not subject to the unstaking cooldown of 28 days and can be transferred out immediately.

### Scenario

Alice is a validator staking 35 000 CQT.
Bob delegates 35 000 CQT to Alice.

Shares distribution:
- Alice owns 35 000e18 shares
- Bob owns 35 000e18 shares

Owner distributes rewards to the tune of 70000 CQT (an exaggerated amount to make the point here)

Alice withdraws all her rewards, and burning half of her shares
Bob unstakes all of his position, but does not redeem his rewards

Shares distribution:
- Alice owns 17 500e18 shares
- Bob owns 17 500e18 shares

Alice and Bob still have the same number of shares, and have the same claim to future rewards.
However Bob's position has less risk compared to a normal delegator, since he can get all of his funds out anytime by calling `redeemRewards`, and does not have to wait a cooldown period.

## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Use a `claimedRewards` mapping to track already claimed rewards instead of burning shares when redeeming rewards