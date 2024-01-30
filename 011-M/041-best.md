Fast Purple Rabbit

medium

# Validators are incentivized to keep their validatorStake low to get more rewards

## Summary
`maxCapMultiplier` prevents delegators from staking more than 27x the validator's stake. Because validator's rewards are inversely proportional to `totalStaked`, validators are incentivized to keep their stake at `validatorEnableMinStake`, and stake the rest as a delegator, which would cut the staking cap for delegators in half, and increase the rewards for the validator by up to 100% as a consequence.

## Vulnerability Detail

1. An honest validator Alice stakes 350_000 CQT (her whole budget); her delegators stake `(27)*350_000 CQT` ([highest limits at launch](https://www.covalenthq.com/docs/covalent-network/staking/)). When rewards are allocated, Alice will get:

`1/28 * (rewards - validatorCommission) + validatorCommission`

2. Sophisticated validator Bob, with the same budget, instead stakes 175_000 CQT as a validator (`==validatorEnableMinStake`), and stakes the other 175_000 CQT from a different account as a delegator, so his delegators stake only `(27-1)*175_000 CQT`. When rewards are allocated, Bob will get:

`2/28 * (rewards - validatorCommission) + validatorCommission` 

which is twice as much as Alice (neglecting the validator commission).

Moreover, Bob will be able to withdraw a half of his funds in 1 month (`delegatorCoolDown`), instead of 6 (`validatorCoolDown`).
## Impact

1. Sophisticated validators get up to twice as much rewards for the same stake.
2. Non-validators staking cap is half as big in comparison to the "honest validator" scenario.
3. Sophisticated validators are able to withdraw `initialBalance - validatorEnableMinStake` of CQT in 1 month instead of 6.
## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L434
## Tool used

Manual Review

## Recommendation
Introduce incentives for validators to keep their validator stake as high as possible.