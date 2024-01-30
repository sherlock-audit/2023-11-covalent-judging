Wild Bronze Marmot

high

# No cooldown in `recoverUnstaking()`, opens up several possible attacks by abusing this functionality.

## Summary

The function `recoverUnstaking()` allows a user to recover any of their unstaking to the same validator. 

However, the function has no cooldown, i.e. a user can unstake and restake to the same validator at any time they prefer. 

We describe several attack vectors that may arise from this issue.

## Vulnerability Detail

When a user unstakes, they must wait for a cooldown of 28 days (180 days if validator) before being able to withdraw. The user also has the choice to revert the action, and re-stake said amount back to the same validator by calling `recoverUnstaking()`.

However, the function `recoverUnstaking()` has no cooldowns. In other words, a user can re-stake as many times as they like, at any time they prefer, without being subject to the unstaking cooldown. 

This opens up several possible attack vectors by abusing the lack of cooldown of this functionality.

Next section we describe some of the possible attack vectors.

### Denying delegators for any validators

An attacker with a large enough capital can do the following to deny a target validator from getting delegated stakings:
- Max stake to the target validator, up to `staked * maxCapMultiplier`, but immediately unstake.
- Whenever someone stakes, front-run them with a `recoverUnstaking()`, re-staking their max stake.
- The user's stake tx fails, as the attacker has taken up the maximum possible stake. 
- The attacker then unstakes, able to repeat the attack when necessary.

The impact is that a validator can be denied from getting delegators.

### Stealing of rewards

**This attack vector builds upon a separate issue we have submitted, about unfair distribution of rewards for mid-epoch stakers.**

An attacker can perform the following, while greatly de-risking themselves from staking.
- Stake an amount they want, but unstake.
- Listen to the staking manager's `rewardValidators()` with the intent of front-running.
- If the attacker choose to, front-run `rewardValidators()` calls, unfairly taking a share of the rewards.
- Unstake.

The attacker benefits from the following:
- While unstaking, their 28-day cooldown still counts. They greatly de-risk themselves by being able to choose whether or not to steal rewards (and reset countdown), or to take profits.
- Unstakings from frozen validators can still be withdrawn once the cooldown has passed. By keeping themselves in an unstaking state, they bypass the freezing mechanism.
- They eliminate the risk from other standard staking mechanism, should they be implemented in the future (e.g. slashing).

## Impact

`recoverUnstaking()` can be abused, opens up several possible attack vectors, with varying impacts (shown above).

## Code Snippet

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L542

## Tool used

Manual Review

## Recommendation

Implement a cooldown for `recoverUnstaking()`. A short cooldown (e.g. a few days) is sufficient.