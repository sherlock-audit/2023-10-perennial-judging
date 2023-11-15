Main Burgundy Cyborg

medium

# Vault might not settle correctly

## Summary
Due to a logical error in mapping ready() check, vault might not settle correctly.
## Vulnerability Detail
Whenever the vault is about to settle, it will check if the latest mapping, which maps market indexes to their latest id, is ready to be settled or not. According to the comment in code:
 _The latest mapping is ready to be settled when all ids in this mapping are greater than the latest mapping_

which should be equivalent to:

_The latest mapping is not ready to be settled when any id of the latest mapping is greater than or equal this mapping_

The problem is the  implementation in the mapping lib does not achieve this statement.
```solidity
 function ready(Mapping memory self, Mapping memory latestMapping) internal pure returns (bool) {
        for (uint256 id; id < latestMapping._ids.length; id++)
            if (get(self, id) > get(latestMapping, id)) return false;
        return true;
  }
```
According to the above code, the latest mapping wont be settled when any id of the current mapping is greater than one in the latest mapping, which do not make sense.
## Impact
Vault settling will not work correctly.
## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/types/Mapping.sol#L63-L67
## Tool used

Manual Review

## Recommendation
Consider change the code to
```solidity
 function ready(Mapping memory self, Mapping memory latestMapping) internal pure returns (bool) {
        for (uint256 id; id < latestMapping._ids.length; id++)
            if (get(latestMapping, id) >= get(self, id) ) return false;
        return true;
  }
```