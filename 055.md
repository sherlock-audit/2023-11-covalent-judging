Faint Licorice Barracuda

medium

# BlockSpecimenProofChain::finalizeSpecimenSession strict inequality yields wrong quorum ratio

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