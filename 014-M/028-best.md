Bouncy Corduroy Chimpanzee

medium

# Pending keeper and position fees are not accounted for in vault collateral calculation which can be abused to liquidate vault when it's small

## Summary

Vault opens positions in the underlying markets trying to keep leverage at the level set for each market by the owner. However, it uses sum of market collaterals which exclude keeper and position fees. But pending fees are included in account health calculations in the `Market` itself.

When vault TVL is high, this difference is mostly unnoticable. However, if vault is small and keeper fee is high enough, it's possible to intentionally add keeper fees by depositing minimum amounts from different accounts in the same oracle version. This keeps/increases vault calculated collateral, but its pending collateral in underlying markets reduces due to fees, which increases actual vault leverage, so it's possible to increase vault leverage up to maximum leverage possible and even intentionally liquidate the vault.

Even when the vault TVL is not low but keeper fee is large enough, the other issue reported allows to set vault leverage to max (according to margined amount) and then this issue allows to reduce vault collateral even further down to maintained amount and then commit slightly worse price and liquidate the vault.

## Vulnerability Detail

When vault leverage is calculated, it uses collateral equal to sum of collaterals of all markets, loaded as following:
```solidity
// local
Local memory local = registration.market.locals(address(this));
context.latestIds.update(marketId, local.latestId);
context.currentIds.update(marketId, local.currentId);
context.collaterals[marketId] = local.collateral;
```

However, market's `local.collateral` excludes pending keeper and position fees. But pending fees are included in account health calculations in the `Market` itself (when loading pending positions):
```solidity
    context.pendingCollateral = context.pendingCollateral
        .sub(newPendingPosition.fee)
        .sub(Fixed6Lib.from(newPendingPosition.keeper));
...
    if (protected && (
        !context.closable.isZero() || // @audit-issue even if closable is 0, position can still increase
        context.latestPosition.local.maintained(
            context.latestVersion,
            context.riskParameter,
@@@         context.pendingCollateral.sub(collateral)
        ) ||
        collateral.lt(Fixed6Lib.from(-1, _liquidationFee(context, newOrder)))
    )) revert MarketInvalidProtectionError();
...
    if (
@@@     !context.currentPosition.local.margined(context.latestVersion, context.riskParameter, context.pendingCollateral)
    ) revert MarketInsufficientMarginError();

    if (
@@@     !PositionLib.maintained(context.maxPendingMagnitude, context.latestVersion, context.riskParameter, context.pendingCollateral)
    ) revert MarketInsufficientMaintenanceError();
```

This means that small vault deposits from different accounts will be used for fees, but these fees will not be counted in vault underlying markets leverage calculations, allowing to increase vault's actual leverage.

## Impact

When vault TVL is small and keeper fees are high enough, it's possible to intentionally increase actual vault leverage and liquidate the vault by creating many small deposits from different user accounts, making the vault users lose their funds.

## Code Snippet

Vault allocations to markets is calculated using collateral value:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L121-L126

This collateral value is calculated as the sum of collaterals in underlying markets (`local.collateral`):
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L501-L505

`context.collaterals` is loaded as following:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L456-L460

`local.collateral` excludes pending fees. Pending fees are added as seen in Market:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L249-L251

## Tool used

Manual Review

## Recommendation

Consider subtracting pending fees when loading underlying markets data context in the vault.