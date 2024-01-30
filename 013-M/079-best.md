Faint Licorice Barracuda

medium

# BlockSpecimenProofChain::submitBlockSpecimenProof Block specimen producer can greatly reduce session duration by submitting fake block specimen in the future

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