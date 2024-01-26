Faint Licorice Barracuda

medium

# OperationalStaking::setValidatorAddress unstaked validator can grief delegator by setting his address as new validator

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