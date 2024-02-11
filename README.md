# Issue M-1: Validator cannot set new address if more than 300 unstakes in it's array 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/25 

## Found by 
PUSH0, SadBase, alexbabits, bitsurfer, nobody2018, thank\_you, yujin718
## Summary
If a validator has more than 300 accumulated unstakes associated with it, then it cannot set a new address for itself. The only way to decrease the length of the `Unstaking` array is through the `setValidatorAddress()` function, but will revert if it's array is longer than 300 entries. A malicious delegator could stake 1,000 tokens, and then unstake 301 times with small amounts to fill up the unstaking array, and there is no way to remove those entries from the array. Every time `_unstake()` is called, it pushes another entry to it's array for that validator.

A malicious validator could also set it's new address to another validator, forcing a merge of information to the victim validator. The malicious validator can do this even when it's disabled with 0 tokens. So it could get to 300 length, and then send all those unstakes into another victim validators unstaking array. This is the same kind of attack vector, but should be noted that validators can assign their address to other validators, effectively creating a forceful merging of them.

The README.md suggests there is a mechanism to counteract this: "In case if there are more than 300 unstakings, there is an option to transfer the address without unstakings." But there appears to be no function in scope that can transfer the address of the validator without unstakings, or any other function that can reduce the unstakings array at all.

## Vulnerability Detail
See Summary

## Impact
Validator can be permanently stuck with same address if there are too many entries in it's `Unstaking` array.

## Code Snippet
Validator & Unstaking Structs: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L35-#L51
`setValidatorAddress()` length check: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L702
deletion of array inside `setValidatorAddress()`: https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L707

## Tool used
Manual Review

## Recommendation
Consider having a way to set a new address without unstakings, or allow for smaller batches to be transferred during an address change if there are too many.
Consider disallowing validators from changing their address to other validators.




## Discussion

**sudeepdino008**

This is interesting. I do have few questions though

> 
A malicious validator could also set it's new address to another validator, forcing a merge of information to the victim validator. The malicious validator can do this even when it's disabled with 0 tokens. So it could get to 300 length, and then send all those unstakes into another victim validators unstaking array. This is the same kind of attack vector, but should be noted that validators can assign their address to other validators, effectively creating a forceful merging of them.


How is setValidatorAddress operates within a `Validator` storage struct, which is retrieved from a query `_validators[validatorId]`. So each validatorId maintains its own unstakings etc. Even if a malicious validator sets its validator address to another validator, the new unstakings are maintained within the current validatorId's Validator instance (which is different from the victim validatorId and therefore victim's Validator instance and its unstakings).

Furthermore setting the newAddress to an existing validator address would typically mean that the attacker doesn't own private key of the victim validator. This would cause loss of funds for the attacker, as the validator needs the private key corresponding to its address to retrieve the funds via redeeum or transferUnstakedOut etc.

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid:  there is no function for that unstaking if its more than 300 in the array; medium(1)



**noslav**

partially fixed by 
[ignore 0 amount unstakings and require unstaking length < 300 - sa46,](https://github.com/covalenthq/cqt-staking/pull/125/commits/63dae4a54c022b8d16e1df6467d98f2b998a1365)

**nevillehuang**

@sudeepdino008 @noslav 

- What is the purpose of setValidatorAddress()? Is it simply to a feature to allow current validator to operate with another address (given they would have to call themselves, so I believe this does not cater to a private key loss scenario)
- Can the validator still perform regular functionalities such as redeeming rewards, commissions submit proofs with no issue? 

If above scenarios are true, I believe this issue is low severity

**noslav**

> @sudeepdino008 @noslav
> 
> * What is the purpose of setValidatorAddress()? Is it simply to a feature to allow current validator to operate with another address (given they would have to call themselves, so I believe this does not cater to a private key loss scenario)
> * Can the validator still perform regular functionalities such as redeeming rewards, commissions submit proofs with no issue?
> 
> If above scenarios are true, I believe this issue is low severity

@nevillehuang This is indeed true, the function does not cater to the private key loss scenario but rather a private key leak scenario where an unknown entity has taken hold of the private key, the action required is for the original owner entity to quickly switch the validator staking address to another address they control to thereby save the stake also from being stolen. This function now has the check that it's not another existing address in the system that belongs to a delegator or a validator. 

In the case of a complete loss of private key, the validator is responsible for any such full loss and has to deal with the "not your keys not your crypto" essentially having no recourse to `_unstake` `_stake` or `_redeemRewards` with the same address

# Issue M-2: `setValidatorAddress` function doesn't take the `delegated` value into account when setting a new validator address as an delegator address 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/38 

## Found by 
SadBase, petro1912, zach223
## Summary
`setValidatorAddress` function checks if `newAddress` is current `validator`'s address or zero.
Therefore `newAddress` could be address of an `delegator` which `validator` can control or has been newly created.

## Vulnerability Detail
When a validator calls `setValidatorAddress` with the delegator address as `newAddress`, `shares` and `stake` are incremented by the value of the delegator.

```solidity
696:     v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
697:     v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
```

At this time, since the ‘delegator’ becomes a ‘validator’, the amount already delegated by the ‘delegator’ must be removed from the ‘delegated’.
However, this was not taken into account in the implementation.

## Impact
Since `delegated` is not changed, the amount that can be delegated later is not correct.

## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L689-L711

## Tool used

Manual Review

## Recommendation

In the `setValidatorAddress` function, if `stacked` value of `newAddress` is not zero, then reduce `delegated` value by its value.
```solidity
function setValidatorAddress(uint128 validatorId, address newAddress) external whenNotPaused {
        Validator storage v = _validators[validatorId];
        require(msg.sender == v._address, "Sender is not the validator");
        require(v._address != newAddress, "The new address cannot be equal to the current validator address");
        require(newAddress != address(0), "Invalid validator address");
        require(!v.frozen, "Validator is frozen");

+       uint128 delegatedBefore = v.stakings[newAddress].staked;
        v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
        v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
        delete v.stakings[msg.sender];

        Unstaking[] storage oldUnstakings = v.unstakings[msg.sender];
        uint256 length = oldUnstakings.length;
        require(length <= 300, "Cannot transfer more than 300 unstakings");
        Unstaking[] storage newUnstakings = v.unstakings[newAddress];
        for (uint128 i = 0; i < length; ++i) {
            newUnstakings.push(oldUnstakings[i]);
        }
        delete v.unstakings[msg.sender];
      
+      if (delegatedBefore != 0)
+         delegated -= delegatedBefore;

        v._address = newAddress;
        emit ValidatorAddressChanged(validatorId, newAddress);
    }
   
      
``` 



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: according to watson's recommendation the tokens will be completely lost as they weren't take account of later.



**rogarcia**

> 1 comment(s) were left on this issue during the judging contest.
> 
> **takarez** commented:
> 
> > invalid: according to watson's recommendation the tokens will be completely lost as they weren't take account of later.

@sherlock-admin2 
can you elaborate on the invalid comment and why this was set as a valid issue while the duplicates weren't? 

**noslav**

the issue is with checking that a validator does not set a delegator address as the new validator address. There's no check in the code for that!


**nevillehuang**

@rogarcia I duplicated #82 and #89 with this issue, so all should be valid. I believe this issue essentially causes issues when setting new validator address as existing delegator. 

However, since this was not supported by the protocol, could this be user input error? Is there a reason why a delegator cannot be a validator at the same time?

**sudeepdino008**

https://github.com/covalenthq/cqt-staking/pull/125/commits/d89d3e2108179658ae0d96f1b2af1ba364f7a772

this resolves the issue

**noslav**

> @rogarcia I duplicated #82 and #89 with this issue, so all should be valid. I believe this issue essentially causes issues when setting new validator address as existing delegator.
> 
> However, since this was not supported by the protocol, could this be user input error? Is there a reason why a delegator cannot be a validator at the same time?
@nevillehuang 

The issue with the delegator being the the same address as the validator is the access to extra delegations within the max multiplier cap on minimum validator stake and explicit avoidance of the incentive to stake more as a delegate to your own validator stake. @sudeepdino008 's commit resolves this now if the validator tries to switch both addresses to be the same using this function. 

# Issue M-3: OperationalStaking may not possess enough CQT for the last withdrawal 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/39 

## Found by 
aslanbek, bitsurfer, cheatcode, dany.armstrong90
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: watson explained how rounding error would prevent the the last to withdraw the chance to unless there are some changes in place; medium(5)



**noslav**

The issue lies in this check 
```solidity
        require(
            effectiveAmount >= REWARD_REDEEM_THRESHOLD,
            "Requested amount must be higher than redeem threshold"
        );
```

where the value by default for REWARD_REDEEM_THRESHOLD is 10*8 and hence redemption below that value is not possible leading to the build up of dust as the issue describes until that threshold is crossed

**noslav**

fixed by [round up sharesToBurn and sharesToRemove due to uint258 to uint128 co](https://github.com/covalenthq/cqt-staking/pull/125/commits/1f957c05aacfb765d751a5ec3cbfd1798e1fae15)

# Issue M-4: New staking between reward epochs will dilute rewards for existing stakers. Anyone can then front-run `OperationalStaking.rewardValidators()` to steal rewards 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/47 

## Found by 
Atharv, Bauer, PUSH0, alexbabits, aslanbek, cawfree, cergyk, hunter\_w3b, ljj, petro1912, thank\_you
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




## Discussion

**sudeepdino008**

- there is a bit of unpredictability with which rewardValidators is called. It might not be possible to ascertain when it will be called.
- What value does Bob get by getting in just before the reward distribution, and then getting out afterwards? Why would he not simply remain staked?
- Note that for each quorum, the stake states (needed for reward calculation) is fetched for the block height (in which that particular quorum was emitted). Even though rewardValidators is called every 12 hours, the offchain system is doing a lot more frequent stake state checks. 

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: users that stakes immediately will be accounted for during the reward distribution; medium as there is no loss of funds; medium(4)



**nevillehuang**

1. This can be done by observing the mempool for calls to `rewardValidators()` given staking contract is deployed on mainnet
2. I believe the PoC in #34 can help you understand a possible impact:

> In the simple demonstration below, we show that an attacker _BOB may steal ~66% of [Validator](https://github.com/sherlock-audit/2023-11-covalent/blob/218d4a583286fa0d3d4263151a927f6cc9465b62/cqt-staking/contracts/OperationalStaking.sol#L40) _ALICE's rewards by frontrunning an incoming call to [rewardValidators(uint128,uint128[],uint128[])](https://github.com/sherlock-audit/2023-11-covalent/blob/218d4a583286fa0d3d4263151a927f6cc9465b62/cqt-staking/contracts/OperationalStaking.sol#L262C14-L262C101):

3. How does this prevent rewards siphoning stated in this issue?

**sudeepdino008**

1. you can observe the mempool for rewardValidators, but by that point, the offchain system has already calculated the validator rewards by taking the stake states into account i.e. when the call is made for rewardValidators, the rewards for each validator is already decided. Front running this call wouldn't change anything.
2. a cooldown for recoverUnstaking is implemented which prevents delegators from entering and exiting staked positions with the same validator.
3. BOB really doesn't "steal" the rewards; He entered the staking position at a particular time and was part of the staking pool. Surely we can make it difficult for them to re-enter after exiting, which is what 2. achieves.


https://github.com/covalenthq/cqt-staking/pull/125/commits/a609cca0426cb22cbf5064212341c14c288efeda for 2.

# Issue M-5: No cooldown in `recoverUnstaking()`, opens up several possible attacks by abusing this functionality. 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/52 

## Found by 
PUSH0
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: the coolDown is in place for withdrawing tokens not for reStaking 'em



**noslav**

fixed by [implement recoverUnstakingCoolDown period for recoverUnstaking levera…](https://github.com/covalenthq/cqt-staking/pull/125/commits/e7ab9ab7eb89f47669dfc0c4ef175f6ca074328b)

**noslav**

supporting fixes by https://github.com/covalenthq/cqt-staking/commit/a609cca0426cb22cbf5064212341c14c288efeda 

# Issue M-6: BlockSpecimenProofChain::finalizeSpecimenSession strict inequality yields wrong quorum ratio 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/55 

## Found by 
cergyk, whoismxuse
## Summary
A quorum factor is used to determine the valid entry for a block specimen, however in some cases a valid quorum may be rejected because of a strict inequality condition upon check in `finalizeSpecimenSession`

## Vulnerability Detail
The following check in `BlockSpecimenProofChain::finalizeSpecimenSession` checks that participants is strictly greater than the applied quorum ratio:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L431

However we can see that with the default parameters this check is incorrect:

- default quorum threshold = 50%
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L134

- minimum submissions required = 2
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L137

This means that in the case 2 of 4 validators agree on a block specimen, the quorum is rejected, because:
`(2*_DIVIDER/4) > _DIVIDER/2` is not verified

This means that legitimate block specimen end up audited

## Impact
Some quorums are marked as invalid during finalization step, and end up DoSing the process of producing block specimens

## Code Snippet

## Tool used

Manual Review

## Recommendation
Please consider replacing with a gte check:

```diff
-if (_minSubmissionsRequired <= max && (max * _DIVIDER) / contributorsN > _blockSpecimenQuorum) {
+if (_minSubmissionsRequired <= max && (max * _DIVIDER) / contributorsN >= _blockSpecimenQuorum) {
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: there is inconsistency in the calculations of the quorum; medium(7)



**noslav**

fixed by [fix quorum calc in case of 4 auditors and default values - sa55/sa115](https://github.com/covalenthq/cqt-staking/pull/125/commits/7405f5a456096337f69f7f9af89c7dc945b9fd7e)

# Issue M-7: `validatorMaxStake` can be bypassed by using `setValidatorAddress()` 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/66 

## Found by 
Al-Qa-qa, PUSH0, aslanbek, cergyk, qmdddd
## Summary

`setValidatorAddress()` allows a validator to migrate to a new address of their choice. However, the current logic only stacks up the old address' stake to the new one, never checking `validatorMaxStake`.

## Vulnerability Detail

The current logic for `setValidatorAddress()` is as follow:

```solidity
function setValidatorAddress(uint128 validatorId, address newAddress) external whenNotPaused {
    // ...
    v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
    v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
    delete v.stakings[msg.sender];
    // ...
}
```

The old address' stake is simply stacked on top of the new address' stake. There are no other checks for this amount, even though the new address may already have contained a stake.

Then the combined total of the two stakings may exceed `validatorMaxStake`. This accordingly allows the new (validator) staker's amount to bypass said threshold, breaking an important invariant of the protocol.

### Proof of concept

0. Bob the validator has a self-stake equal to `validatorMaxStake`.
1. Bob has another address, B2, with some stake delegated to Bob's validator.
2. Bob migrates to B2.
3. Bob's stake is stacked on top of B2. B2 becomes the new validator address, but their stake has exceeded `validatorMaxStake`.
4. B2 can then repeated this procedure to addresses B3, B4, ..., despite B2 already holding more than the max allowed amount.

Bob now holds more stake than he should be able to, allowing him to earn an unfair amount of rewards compared to other validators.

We also note that, even if the admin tries to freeze Bob, he can front-run the freeze with an unstake, since unstakes are not blocked from withdrawing (after cooldown ends).

## Impact

- Breaking an important invariant of the protocol.
- Allowing any validator to bypass the max stake amount. In turn allows them to earn an unfair amount of validator rewards in the process.
- Allows a validator to unfairly increase their max delegator amount, as an effect of increasing `(validator stake) * maxCapMultiplier`.

## Code Snippet

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L689-L711

## Tool used

Manual Review

## Recommendation

Check that the new address's total stake does not exceed `validatorMaxStake` before proceeding with the migration.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: user can exceed max  deposit and potentially increase his max delegator amoun; high(1)



**noslav**

fixed by [prevent new validator address stake from exceeding max stake - sa66/s…](https://github.com/covalenthq/cqt-staking/pull/125/commits/0eba6b318b9400e4a5d6511ba4c96922b83b9abd)

# Issue M-8: OperationalStaking::_unstake Delegators can bypass 28 days unstaking cooldown when enough rewards have accumulated 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/78 

## Found by 
cergyk, irresponsible, qmdddd
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid: should provide a POC



**noslav**

fixed partially by
[prevent bypassing cooldown with redelegateUnstaked - sa78/sa68/sa76](https://github.com/covalenthq/cqt-staking/pull/125/commits/5bf8940c8d5642652b1987cc74cb2f6780b06b08)

**rogarcia**

PR: https://github.com/covalenthq/cqt-staking/pull/125
commit: https://github.com/covalenthq/cqt-staking/pull/125/commits/5bf8940c8d5642652b1987cc74cb2f6780b06b08



# Issue M-9: BlockSpecimenProofChain::submitBlockSpecimenProof Block specimen producer can greatly reduce session duration by submitting fake block specimen in the future 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/79 

## Found by 
cergyk
## Summary
Block specimen producers submit specimens for a given block number during a limited time called a session. Any block producer can start a session by calling `submitBlockSpecimenProof` for a given block number. This means that a block specimen producer can start a session for a block which does not exist yet, and severily reduce the actual session time, since honest block specimen producer can only participate between the time the block has been created and the end of session.

## Vulnerability Detail
We can see that a session is started when the first specimen for the block height is submitted:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L346-L348

This means that if a malicious block specimen producer has sent some invalid data for a block height which is in the future, the session is still started for that block. The following check ensures that a producer can not call submit for a block too far in the future:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L344

But since the default value for `cd.allowedThreshold` would be 100 blocks, and a session duration would be approximately 240 blocks, we can see that a malicious block producer can reduce the actual session duration for honest producers by half.

## Impact
The block specimen production session can be greatly reduced by a malicious producer (up to a half with current deploy parameters).

## Code Snippet

## Tool used

Manual Review

## Recommendation
Please consider starting the session at the estimated timestamp of the considered `blockHeight`:
```diff
- session.sessionDeadline = uint64(block.number + _blockSpecimenSessionDuration);
+ uint64 timestampOnDestChain = (blockHeight-cd.blockOnTargetChain)*cd.secondsPerBlock-cd.blockOnCurrentChain*_secondsPerBlock; 
+ session.sessionDeadline = uint64(timestampOnDestChain + _blockSpecimenSessionDuration);
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: user can submit fake here also; medium(3)



**noslav**

we can mitigate this a bit although not completely as we’re currently tied to this way of doing things. We use the block numbers on the current chain (aka current block number on moonbeam + number of blocks to wait for session duration) to determine the deadline for any input block specimen number.. there’s currently no way to know the head of the source chain (ethereum) except through the estimated calculation being done in the contract where block time * time diff from last chain sync tx and we can use to create a shorted upper bound so blocks too far in the future cannot be submitted!

**noslav**

partially fixed by [impl proof submission upper bounds to mitigate future block deadlines](https://github.com/covalenthq/cqt-staking/pull/125/commits/481fcd4ea97e7f6e998dd30ef15122a8e256e5dc)

# Issue M-10: OperationalStaking::setValidatorAddress unstaked validator can grief delegator by setting his address as new validator 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/88 

## Found by 
cergyk
## Summary
Validators and delegators both stake CQT to earn rewards associated with a validator Id, however they have vastly different withdrawal times, respectively `6 months` and `28 days`. This means that a malicious validator can grief a delegator by setting his address as new validator and imposing on him to keep the funds in the contract for 6 months instead of 28 days.

## Vulnerability Detail
We can see that a delegator can set any address except current as new validator address:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L689-L711 

Also since this function does not check that a validator is disabled, the validator can have unstaked most of his funds already.

### Scenario

Setup:
Alice has a validator of id 1, Bob delegated to Alice's validator.

1/ Alice disables her validator by unstaking most of her funds:
> Alice can unstake the amount such as `maxDelegationCap` ratio is respected. Since the value currently used in production for this ratio is x20, we can estimate that Alice has to leave only 5% of the value of funds delegated to her in the contract.

2/ Alice waits the cooldown period and transfers all of her funds out.

3/ Alice sets Bob as the new validator, which would lock Bob's stake for 6 months instead of 28 days, and Alice loses 5% of Bob funds.

## Impact
Alice as a validator can cause griefing to Bob which has delegated, by sacrificing 5% of the value of the delegation, to make Bob lock his funds for 6 months instead of 28 days.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Please consider ensuring that validator does not set the new validator address to a current delegator, or enable the delegator to opt-out of being set as the new validator.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  invalid



**noslav**

This is a bigger ask since first we have to extract all addresses involved in the stakings map in the validator struct
```solidity
    struct Validator {
        uint128 commissionAvailableToRedeem;
        uint128 exchangeRate; // validator exchange rate
        address _address; // wallet address of the operator which is mapped to the validator instance
        uint128 delegated; // track amount of tokens delegated
        uint128 totalShares; // total number of validator shares
        uint128 commissionRate;
        uint256 disabledAtBlock;
        mapping(address => Staking) stakings;
        mapping(address => Unstaking[]) unstakings;
        bool frozen;
    }
    
    function getTotalValidators() public view returns (uint256) {
        return validatorsN;
    }

    function getAllDelegators() public view returns (bytes32[] memory delegatorKeys) {
        for (uint128 i = 0; i < validatorsN; i++) {
            Validator storage v = _validators[i];
            for (uint128 j = 0; j < v.stakings.length; j++) {
                address delegator = v.stakings[msg.sender][j].address; // Extract the delegate's address from the staking mapping
                bytes32 delegatorKey = keccak256(abi.encodePacked(delegator)); // Generate a unique identifier for the delegate's key
                if (!delegatorKey.equals(bytes32[](new bytes32[]))) {
                    delegatorKeys.push(delegatorKey); // Add the delegate's key to the array of delegator keys
                }
            }
        }

        return delegatorKeys;
    }
```

**noslav**

partially fixed by [implement unique delegator extraction for setValidatoAddress - sa88](https://github.com/covalenthq/cqt-staking/pull/125/commits/9f2efadaaf894a5593aef2131a2f4726ccd64506)

**sudeepdino008**

the implication of an malicious validator doing this is that he loses control of his funds (since delegator address is not owned by the validator). So sure malicious validator can do this, but at the cost of losing all his funds. So this attack vector doesn't make sense.

**noslav**

@sudeepdino008 good point however since the max cap allow a higher delegation amount than validator min stake, the validator can still unstake till they are at the min level and yet grieve a delegator with a higher delegated amount by setting them as a validator address (say for example: a delegator pulls out from a good validator and restakes to another validator with lower commissions etc), plus the delegator will also now be responsible for other delegators for that particular validator losing funds/rewards since infrastructure has to be set up to meet the needs. This can be attack vector we should prevent. This has parallels to the nothing at stake problem (except here its a "comparably low at stake") 


**rogarcia**

commit with fix https://github.com/covalenthq/cqt-staking/pull/125/commits/d89d3e2108179658ae0d96f1b2af1ba364f7a772

# Issue M-11: Validators can get prevented from unstaking all their tokens 

Source: https://github.com/sherlock-audit/2023-11-covalent-judging/issues/98 

## Found by 
Al-Qa-qa
## Summary
Validators will not be able to unstake their tokens (in case of leaving the staking) if there are a lot of delegates.

## Vulnerability Detail
The validator will get enabled when they stake > `validatorEnableMinStake`, and accepts other users to delegate their tokens to him, and if the validator unstaked his tokens with amount < `validatorEnableMinStake` it will be un-enable.

The problem arises when the validator staked tokens are > `validatorEnableMinStake` by a certain amount, and the number of delegates is relatively big, we will illustrate how, and why this occurs.

In `OS::_unstake()`, The validator maxCap check is done before `validatorEnableMinStake` check. So if the validator wanted to unstake all his tokens (has no plans to complete investing in the protocol for example), his function will get reverted if the `validatorMaxCap` decreases significantly. 

[OperationalStaking.sol#L466-L537](https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L466-L537)
```solidity
function _unstake(uint128 validatorId, uint128 amount) internal {
    ...

    bool isValidator = msg.sender == v._address;
    // checking for `newValidatorMaxCap`, and reverts if it does not met 
    if (isValidator && v.disabledAtBlock == 0) {
        // validators will have to disable themselves if they want to unstake tokens below delegation max cap
        uint128 newValidatorMaxCap = newStaked * maxCapMultiplier;
      
        require(v.delegated <= newValidatorMaxCap, "Cannot decrease delegation max-cap below current delegation while validator is enabled");
    }

    ...

    // disable validator if they unstaked to below their required self-stake
    if (isValidator && validatorEnableMinStake > 0 && v.disabledAtBlock == 0 && s.staked < validatorEnableMinStake) {
        uint256 disabledAtBlock = block.number;
        v.disabledAtBlock = disabledAtBlock;
        emit ValidatorDisabled(validatorId, disabledAtBlock);
    }

    ...
}
```

_NOTE: the validator can not be able to withdraw tokens which makes `validatorMaxCap` smaller than `validator.delegates`, and this is the system design. we are pointing out that the validator will not be able to unstake all his tokens or below `validatorEnableMinStake` if he wants to un-enable himself. So if the validator wanted to unstake all his tokens and simply leaves the protocol he would not be able to do this._

Let's illustrate an example when `validatorEnableMinStake = 35_000e18`, and `multiplier = 10`.

|validator.stakes|validator unstake tokens|delegated tokens|newValidatorMaxCap <=> delegated|reverts|
|----------------|------------------------|----------------|------------------|-------|
|35_000e18|35_000e18|0e18|0e18 = 0e18|no|
|35_000e18|35_000e18|10_000e18|0e18 < 10_000e18|yes|
|100_000e18|80_000e18|150_000e18|20_000e18 * 10 > 150_000e18|no|
|100_000e18|80_000e18|500_000e18|20_000e18 * 10 < 500_000e18|yes|
|350_000e18|320_000e18|200_000e18|30_000e18 * 10 > 200_000e18|no|
|350_000e18|320_000e18|2_000_000e18|30_000e18 * 10 < 2_000_000e18|yes|

So from the table of results, we can conclude that:

- The more delegates nears the `validatorMaxCap`, the more probability the validator will not be able to unstake all there tokens.
- When `validator.stakes` are large compared to `validator.delegates`, things work properly
- When `validator.stakes` are small compared to `validator.delegates`, validators will not be able to unstake all there tokens.

## Impact
Validators will not be able to unstake all their tokens if they want to leave the staking.

## Code Snippet
- [OperationalStaking.sol#L495-L502](https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L495-L502)
- [OperationalStaking.sol#L523-L527](https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L523-L527)
## Tool used
Manual Review + Foundry

## Recommendation

`validatorEnableMinStake` should be done before the `validatorMaxCap` check. So that the validator can unstake all his tokens, without getting his transaction reverted.

> `OS._unstake()`
```diff
function _unstake(uint128 validatorId, uint128 amount) internal {
    ...

    bool isValidator = msg.sender == v._address;

+    if (isValidator && validatorEnableMinStake > 0 && v.disabledAtBlock == 0 && newStaked < validatorEnableMinStake) {
+        uint256 disabledAtBlock = block.number;
+        v.disabledAtBlock = disabledAtBlock;
+        emit ValidatorDisabled(validatorId, disabledAtBlock);
+    }

    // checking for `newValidatorMaxCap`, and reverts if it does not met 
    if (isValidator && v.disabledAtBlock == 0) {
        // validators will have to disable themselves if they want to unstake tokens below delegation max cap
        uint128 newValidatorMaxCap = newStaked * maxCapMultiplier;
      
        require(v.delegated <= newValidatorMaxCap, "Cannot decrease delegation max-cap below current delegation while validator is enabled");
    }

    ...

    // disable validator if they unstaked to below their required self-stake
-    if (isValidator && validatorEnableMinStake > 0 && v.disabledAtBlock == 0 && s.staked < validatorEnableMinStake) {
-        uint256 disabledAtBlock = block.number;
-        v.disabledAtBlock = disabledAtBlock;
-        emit ValidatorDisabled(validatorId, disabledAtBlock);
-    }

    ...
}
```



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**takarez** commented:
>  valid: medium(9)



**noslav**

This issue begs the question: Do we want to allow validators to exit the system whenever possible even if they have a lot of delegations and are themselves staked at a lower (in comparison to total delegation amount or other validators)? 

This could be a feature than a bug aka when the validator has a lot of delegations in proportion to their own stake the `_unstake` tx reverts, not allowing them to proceed. This means they have convince the delegators to first unstake their stakes and only then be able to execute their own unstake. By these means the protocol applies a pressure to keep validators with high delegations in the system (and not be able to rug pull delegators at a whim). 

Covering cases
* When validator.stakes are small compared to validator.delegates, validators will not be able to unstake all there tokens.
* The more delegates nears the validatorMaxCap, the more probability the validator will not be able to unstake all there tokens.

**noslav**

fixed by [move validatorEnableMinStake check before validatorMaxCap - sa98](https://github.com/covalenthq/cqt-staking/pull/125/commits/f7323ce9c348713f40002a94557451135a3b93b8)

**sudeepdino008**

cc: @rogarcia 
Good point from @noslav. Right now if the validator wants to exit the system, they have to ask us to disable the validator and only then they can unstake. This doesn't work al the time though -- the validator is only blocked when the maxDelegationCap limit is crossed. 

I'm still not sure if we should completely incorporate this, and allow validators to exit when they want. At least currently the validators with large delegations have to come to us and ask us to disable. Incorporating this change can surprise us.

**nevillehuang**

@noslav @sudeepdino008 This seems to me like a valid design consideration, given covalent wants to retain large delegation within the system, while still being trusted to rectify the situation if large validators wish to unstake from the system. 

However, is the max delegation cap enforced during delegation to validators? If not this could be a griefing vector by delegates toward validators trying to unstake.

**noslav**

@nevillehuang @sudeepdino008 good points about reversing the argument on its head with the delegators being in control of what the validator should or shouldn't do - ideally this should be a governance process before it makes it on-chain (that being said we're working on getting upto speed on enabling the same for all CQT holders). In the meantime we have decided to move forward with the suggested change as ideally both delegators and validators should be able to come in and go in a permission-less way even with the current iteration of the network (and in the coming future with proposed migration from moonbeam onto ethereum), this translates to increased trust between all parties encouraging positive reinforcement behaviour and actions. Since setting up a validator requires a lot of work and responsibilities including the ones towards your delegators (the validator name is always known to delegators pre and post the fact). We just have to get better at penalizing explicit malicious behaviour, where coming and going within the network shouldn't be one of them for a system that is tending towards a fully permissionless setup.

**noslav**

also check commit [fix check for validator disable below required self-stake](https://github.com/covalenthq/cqt-staking/pull/125/commits/5cc81e06f2063e6583e95bff881476b613e767ab) caught during unit-tests

