Sharp Pistachio Perch

medium

# The `getDelegatorTotalLocked` function does not return the correct value, but only returns the value for one validator.

## Summary
`getDelegatorTotalLocked` function should return the total locked amount for all the validators.
However, it returns an incorrect value, which is just the amount locked to one validator.

## Vulnerability Detail
In the for loop of the `getDelegatorTotalLocked` function, you need to add each validator's locked value to `totalValueLocked`.
However, instead of appending, the validator was initialized on each iteration.
Therefore, if a delegator delegates 'staking' to multiple validators, they will only get the locked value for the last validator.

## Impact
Validator or delegator would read incorrect total locked value and totally confused.

## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L848

## Tool used

Manual Review

## Recommendation
`OperationalStaking.sol` L848
```solidity
-   totalValueLocked = _sharesToTokens(s.shares, v.exchangeRate);
+   totalValueLocked += _sharesToTokens(s.shares, v.exchangeRate);
```
