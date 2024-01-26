Steep Orchid Mole

high

# Inadequate Access Control on Critical Functions (Unauthorized Access)

## Summary
The contract exposes critical governance functions without proper access control, allowing unauthorized entities to potentially manipulate governance actions, including adding or removing operators and altering configuration parameters.
## Vulnerability Detail
The contract contains several critical functions, such as **addBSPOperator**, **removeBSPOperator**, **addAuditor**, **removeAuditor**, **addGovernor**, and **removeGovernor**, that alter the state of the governance model by adding or removing operators or changing configuration parameters. These functions lack adequate access controls, allowing any caller to execute them without proper authorization checks. This oversight can lead to unauthorized access and manipulation of the governance process.
## Impact
The lack of access control on these critical functions poses a significant security risk. Malicious actors could exploit this to gain unauthorized control over the governance process, manipulate decisions, or disrupt the normal operation of the protocol.
## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L176-L177
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L190-L191
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L203-L204
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L213-L214
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L223-L224
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L233-L234
## Tool used

Manual Review

## Recommendation
Introduce robust access control mechanisms to protect these functions. Use role-based access control (RBAC) patterns, such as **OpenZeppelin's** **AccessControl**, to restrict function execution to authorized addresses only. Each critical function should check if the caller has the appropriate role before proceeding with the execution.

```solidity
// Example using OpenZeppelin's AccessControl for addBSPOperator function
function addBSPOperator(address operator, uint128 validatorId) external onlyRole(BSP_OPERATOR_ROLE) {
    require(operatorRoles[operator] == 0, "Operator already exists");
    ...
}

// Continue similarly for other functions
...
```
