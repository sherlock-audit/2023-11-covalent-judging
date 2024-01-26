Dry Iron Coyote

medium

# Discrepancy in Access Control Between Documentation and Implementation in `setValidatorCommissionRate` Function

## Summary

The access control modifier of `onlyStakingManager` should be used instead of `onlyOwner`. There is a mismatch on the implementation and the docs.
## Vulnerability Detail

```solidity
function setValidatorCommissionRate(uint128 validatorId, uint128 amount) external onlyOwner {
	require(validatorId < validatorsN, "Invalid validator");
	require(amount < DIVIDER, "Rate must be less than 100%");
	_validators[validatorId].commissionRate = amount;
	emit ValidatorCommissionRateChanged(validatorId, amount);
}
```

In the documentation, it mentioned that only staking manager should be able to set validator's commission rate. However, in the implementation code, `onlyOwner` is passed in. There is a mismatch in the modifier used. Furthermore, a staking manager might not be the owner and this may prevent him from being able to change the commission rate.

## Impact

The staking manager might not be able to change the validator's commission rate since he might not have been a owner. This misalignment can lead to management inefficiencies and possibly undermine the governance model outlined in the project's documentation.

## Code Snippet

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L362

## Tool used

Manual Review

## Recommendation

Consider changing the access control modifier to `onlyStakingManager`.

```diff
- function setValidatorCommissionRate(uint128 validatorId, uint128 amount) external onlyOwner {
+ function setValidatorCommissionRate(uint128 validatorId, uint128 amount) external onlyStakingManager {

	require(validatorId < validatorsN, "Invalid validator");
	require(amount < DIVIDER, "Rate must be less than 100%");
	_validators[validatorId].commissionRate = amount;
	emit ValidatorCommissionRateChanged(validatorId, amount);
}
```