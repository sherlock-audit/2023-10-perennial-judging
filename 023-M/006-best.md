Electric Satin Cricket

medium

# Vault.settle() won't be able to rebalance the collateral and position of the vault

## Summary
Vault.settle() is supposed to  rebalance the collateral and position of the vault without a deposit or withdraw but this won't happen because `bool rebalance` is set as false when calling Vault._manage().

## Vulnerability Detail
Vault._manage() has 3 params: `context`, `withdrawAmount`, and `bool rebalance`. 

The value of `bool rebalance` determines whether to rebalance the vault's position or not. 

Now the issue is when Vault.settle() calls _manage(), it sets `bool rebalance` as false, see [here](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L224). 

Now this if statement [here](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L391) will return the execution without rebalancing the collateral and position of the vault.

## Impact
Vault.settle() won't be able to rebalance the collateral and positions of the vault
## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L224

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L391
## Tool used

Manual Review

## Recommendation
`bool rebalance` should be set to `true` and not `false` within Vault.settle(), since its supposed to  rebalance the collateral and position of the vault 