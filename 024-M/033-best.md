Calm Gingham Beaver

medium

# Settlement fee of unused markets is still charged in Vault

## Summary

When markets are removed from usage in a vault, their weight is set to 0. However, their fixed market settlement fee is always charged regardless of whether the market is actually being used.

## Vulnerability Detail

The only way to remove a market in Vault.sol is by updating the market weight and leverage to 0 with `updateMarket`. However, the market will still be listed as a market, in which case its fixed settlement fee will be included in the total `settlementFee` amount to be paid whenever a position is changed. 

This results in the market's `settlementFee` being excluded from vault users claim amounts, effectively resulting in a material loss of funds for vault users.

## Impact

Markets can't be removed from use in a vault without incurring a loss of user funds with each claim.

## Code Snippet

Can only update market weight and leverage:

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L170
```solidity
/// @notice Settles, then updates the registration parameters for a given market
/// @param marketId The market id
/// @param newWeight The new weight
/// @param newLeverage The new leverage
function updateMarket(uint256 marketId, uint256 newWeight, UFixed6 newLeverage) external onlyOwner {
    settle(address(0));
    _updateMarket(marketId, newWeight, newLeverage);
}

/// @notice Updates the registration parameters for a given market
/// @param marketId The market id
/// @param newWeight The new weight
/// @param newLeverage The new leverage
function _updateMarket(uint256 marketId, uint256 newWeight, UFixed6 newLeverage) private {
    if (marketId >= totalMarkets) revert VaultMarketDoesNotExistError();

    Registration memory registration = _registrations[marketId].read();
    registration.weight = newWeight;
    registration.leverage = newLeverage;
    _registrations[marketId].store(registration);
    emit MarketUpdated(marketId, newWeight, newLeverage);
}
```

For each market we add the settlementFee regardless of weight.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L454
```solidity
for (uint256 marketId; marketId < totalMarkets; marketId++) {
    ...
    context.settlementFee = context.settlementFee.add(marketParameter.settlementFee);
    ...
}
```

## Tool used

Manual Review

## Recommendation

Include a function to remove markets from use within a vault such that `totalMarkets` is decremented and the `marketId` is somehow marked as unused.