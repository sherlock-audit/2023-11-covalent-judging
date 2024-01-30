Faint Licorice Barracuda

medium

# BlockSpecimenProofChain::_finalizeWithParticipants Finalization can be bricked if number of validators is greater than 256

## Summary
Finalization can be bricked if there exists more than 256 validators in the system

## Vulnerability Detail
If there exists a validator with id > 255, and the producer submits a block specimen for the session, we can see that this line reverts with an underflow:
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L452

This means that the finalization for that block will be bricked undefinitely, and since it reverts during this function, it is not even possible to mark it as `to audit` and recover.

> Please note that it is not possible currently to remove validators, only to disable them. So the condition is not to have more than 256 validators at a given time, but that since inception more than 256 validators have been created. This makes the bug more likely 

## Impact
The block specimen finalization can be bricked undefinitely.

## Code Snippet

## Tool used

Manual Review

## Recommendation
Please consider reusing validator IDs to keep this bitmask (and if number of validators which can simultaneously exist is guaranteed to be less than 256)