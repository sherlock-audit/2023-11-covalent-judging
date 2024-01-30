Witty Black Ant

medium

# Function getBSPRoleData always return requiredStake with zero value

## Summary
Function use unitialized variable so user get incorrect data from function
## Vulnerability Detail
function getBSPRoleData() read variable _bspRequiredStake (which has never been initialized) and return value of it as 0. 
## Impact
User receive wrong data from contract
## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L521-L523
## Tool used

Manual Review

## Recommendation
Initialize this variable in initialize() function