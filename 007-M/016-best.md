Ripe Tartan Woodpecker

medium

# BlockSpecimenProofChain.sol#finalizeSpecimenSession() - multiple flaws during selection of reward winners

## Summary
The ``finalizeSpecimenSession()`` function serves the purpose of finding the most submitted specimen hash for a block and rewarding the submitting operators by meeting a number of submissions and a quorum. Due to the logic implementation there are cases that can lead to disadvantages for operators.

## Vulnerability Detail
First things that come to mind when reviewing the function:
1. ``contributorsN`` indefinitely increases as long as there are hashes with participants, be it only 1. The number of hashes producable by a single operator is limited by the chainData's ``maxSubmissionsPerBlockHeight`` (which does not account for block-height so it is flawed on it's own). Thus a user can submit fraud hashes up until the max limit in hopes of altering the quorum calculation: ``(max * _DIVIDER) / contributorsN > _blockSpecimenQuorum``. E.g if we have a quorum of 50%, per 1 max we need 10 contributors total to NOT reach quorum, so if the max submitted hash has 10 participants, if we have 90-95 total contributors we would pass quorum, but depending on the ````maxSubmissionsPerBlockHeight`` a user can maliciously submit 5 more hashes to prevent other from winning the quorum rewards. This holds even harder for ``_blockSpecimenQuorum`` that's above 50% (>10e17)
2. In the edge-case that 2 hashes meet the same number of total participants, the earlier submitted hash would win the quorum, potentially disincentivizing the submission of later hashes, turning hash submission into something like a race-condition

Using number 1 we can impact number 2, as using the mempool for Moonbeam we can know what hashes have been submitted and in the a hash is off-by-one from another one after it, we can game it and submit it (if we haven't done so yet). 

If a multiple-submit transaction is done atomically, this cannot be prevented by simply disabling the operator for the current epoch 
## Impact
Potential gaming and inconsistency in reward winners allocation

## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L397-L439
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L328-L392

## Tool used

Manual Review

## Recommendation
Restructure how you use ``maxSubmissionsPerBlockHeight`` and maybe even remove it entirely.
As of right now the variable wrongly assumes it covers all participants, but it is a per-participant variable and it does not take per block height into account, contrary to it's name. Allowing an operator to submit multiple different specimen hashes can deem problematic.
