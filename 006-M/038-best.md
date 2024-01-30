Sharp Pistachio Perch

medium

# `setValidatorAddress` function doesn't take the `delegated` value into account when setting a new validator address as an delegator address

## Summary
`setValidatorAddress` function checks if `newAddress` is current `validator`'s address or zero.
Therefore `newAddress` could be address of an `delegator` which `validator` can control or has been newly created.

## Vulnerability Detail
When a validator calls `setValidatorAddress` with the delegator address as `newAddress`, `shares` and `stake` are incremented by the value of the delegator.

```solidity
696:     v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
697:     v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
```

At this time, since the ‘delegator’ becomes a ‘validator’, the amount already delegated by the ‘delegator’ must be removed from the ‘delegated’.
However, this was not taken into account in the implementation.

## Impact
Since `delegated` is not changed, the amount that can be delegated later is not correct.

## Code Snippet
https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L689-L711

## Tool used

Manual Review

## Recommendation

In the `setValidatorAddress` function, if `stacked` value of `newAddress` is not zero, then reduce `delegated` value by its value.
```solidity
function setValidatorAddress(uint128 validatorId, address newAddress) external whenNotPaused {
        Validator storage v = _validators[validatorId];
        require(msg.sender == v._address, "Sender is not the validator");
        require(v._address != newAddress, "The new address cannot be equal to the current validator address");
        require(newAddress != address(0), "Invalid validator address");
        require(!v.frozen, "Validator is frozen");

+       uint128 delegatedBefore = v.stakings[newAddress].staked;
        v.stakings[newAddress].shares += v.stakings[msg.sender].shares;
        v.stakings[newAddress].staked += v.stakings[msg.sender].staked;
        delete v.stakings[msg.sender];

        Unstaking[] storage oldUnstakings = v.unstakings[msg.sender];
        uint256 length = oldUnstakings.length;
        require(length <= 300, "Cannot transfer more than 300 unstakings");
        Unstaking[] storage newUnstakings = v.unstakings[newAddress];
        for (uint128 i = 0; i < length; ++i) {
            newUnstakings.push(oldUnstakings[i]);
        }
        delete v.unstakings[msg.sender];
      
+      if (delegatedBefore != 0)
+         delegated -= delegatedBefore;

        v._address = newAddress;
        emit ValidatorAddressChanged(validatorId, newAddress);
    }
   
      
``` 
