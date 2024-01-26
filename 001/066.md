Wild Bronze Marmot

high

# `validatorMaxStake` can be bypassed by using `setValidatorAddress()`

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
