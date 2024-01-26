Agreeable Chrome Rhino

medium

# Validators can get prevented from unstaking all their tokens

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
