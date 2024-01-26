Shaggy Cedar Cottonmouth

high

# BlockSpecimenProofChain: incorrect deletion of BlockSpecimenProofChain

## Summary
**Mapping in struct**
When a struct in Solidity is deleted, if the fields are value type, they are reset to default value. Bu, if there are reference times like mapping, they will not get deleted when the struct is deleted.

**delete operator functioning for simple data types**
The delete keyword in Solidity sets every field in the struct to its default value. For integers, strings, arrays, and other simple data types, this means they will be set to zero, an empty string, or an empty array, respectively.

**Mapping/dynamic arrays**
However, for mappings, the delete keyword has no effect. This is because mappings are implemented as hash tables and the Ethereum Virtual Machine (EVM) does not keep track of which keys have been used in the mapping. As a result, it doesn't know how to "reset" a mapping. Therefore, when you delete a struct, the mapping within it will still retain its old data.

This can lead to potential security issues.For example, let’s say we have a struct that contains sensitive data within a mapping. when delete the struct, it is assumed that all data within it will be erased, but, actually the data in the mapping will still persist, potentially leading to unintended access or misuse. 

## Vulnerability Detail
```solidity
struct BlockProperties {
        mapping(bytes32 => address[]) participants; // specimen hash => operators who submitted the specimen hash
        bytes32[] specimenHashes; // raw specimen hashes
    }
```
Now, lets look at the data structure for **blockProperties**. It has a mapping inside to store specimen hash to operators array. Also, specimenHashes is a dynamic array and hence should be hash tables.

```solidity
delete session.blockProperties[agreedBlockHash] 
```

So. calling delete as above will not clear the data stored on the hash maps and could potentially be abused by someone mocking to generate the same agreedBlockHash and getting access to the blockProperties data that were assumed to be deleted.

## Impact
Data presumed to be deleted is not actually deleted and can be restored back, which can be abused by a skilful attacker

## Code Snippet
The below code does not serve the intended purpose.

```solidity
 delete session.blockProperties[agreedBlockHash]; // release gas
```

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/BlockSpecimenProofChain.sol#L462

## Tool used

Manual Review

## Recommendation
In order to clear the blockProperties, a function should be written that will iterate over the keys of map and delete every single entry on the dynamic array. 

The logic will have to also iterate over specimenHashes array to delete each entry explicitly.