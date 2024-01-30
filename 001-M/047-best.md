Wild Bronze Marmot

high

# New staking between reward epochs will dilute rewards for existing stakers. Anyone can then front-run `OperationalStaking.rewardValidators()` to steal rewards

## Summary

In the Covalent Network, validators perform work to earn rewards, which is distributed through `OperationalStaking`. The staking manager is expected to regularly invoke `rewardValidators()` to distribute staking rewards accordingly to the validator, and its delegators.

However, the function takes into account all existing stakes, including new ones. This makes newer stakes being counted equally to existing stakes, despite newer stakes haven't existed for a working epoch yet.

An attacker can also then front-run `rewardValidators()` to steal a share of the rewards. The attacker gains a share of the reward, despite not having fully staked for the corresponding epoch.

## Vulnerability Detail

The function `rewardValidators()` is callable only by the staking manager to distribute rewards to validators. The rewards is then immediately distributed to the validator, and all their delegators, proportional to the amount staked.

However, any new staking in-between reward epochs still counts as staking. They receive the full reward amount for the epoch, despite not having staked for the full epoch. 

An attacker can abuse this by front-run the staking manager's distribution with a stake transaction. The attacker stakes a certain amount right before the staking manager distributes rewards, then the attacker is already considered to have a share of the reward, despite not having staked during the epoch they were entitled to.

This also applies to re-stakings, i.e. unstaked tokens that are re-staked into the same validator: Any stake recovers made through `recoverUnstaking()` is considered a new stake. Therefore an attacker can use the same funds to repeatedly perform the attack.

### Proof of concept

0. Alice is a validator. She has two delegators:
    - Alicia delegating $5000$ CQT.
    - Alisson delegating $15000$ CQT.
    - Alice herself stakes $35000$ CQT, for a total of $50000$.
1. The staking manager distributes $1000$ CQT to Alice. This is then distributed proportionally across Alice and her delegators.
2. Bob notices the staking manager, and front-runs with a $50000$ CQT stake into Alice.
    - Alice now has a total stake of $100000$, with half of it belonging to Bob.
    - Bob's staking tx goes through *before* the staking manager's tx.
3. Bob now owns half of Alice's shares. He is distributed half of the rewards, despite having staked into Alice for only one second.
4. Bob can repeat this attack for as often as they wish to, by unstaking then restaking through `recoverUnstaking()`.

## Impact

Unfair distribution of rewards. New stakers get full rewards for epochs they didn't fully stake into.

## Code Snippet

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L262

## Tool used

Manual Review

## Recommendation

Use a checkpoint-based shares computation. An idea can be as follow:
- For new stakings, don't mint shares for them right away, but add them into a "pending stake" variable.
- For unstakings, do burn their shares right away, as with the normal setting.
- When reward is distributed, distribute to existing stakes first (i.e. increase the share price only for the existing shares), only then mint new shares for the pending stakes.

