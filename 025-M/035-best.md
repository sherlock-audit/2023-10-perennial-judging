Calm Gingham Beaver

high

# Broken market efficiency invariant check

## Summary

The market efficiency invariant check in `Market._invariant` fails to be properly checked in most cases due to an unexpected short circuit.

## Vulnerability Detail

In `Market._invariant` we validate that the market efficiency is above a specified limit as defined by the `riskParameter`.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L618
```solidity
if (
    newOrder.liquidityCheckApplicable(context.marketParameter) &&
    newOrder.efficiency.lt(Fixed6Lib.ZERO) &&
    context.currentPosition.global.efficiency().lt(context.riskParameter.efficiencyLimit)
) revert MarketEfficiencyUnderLimitError();
```

We first validate, `newOrder.liquidityCheckApplicable(context.marketParameter)`, and if the result is false then we short circuit out of this check and don't bother validating the market efficiency. The problem with this is that there are reasonable circumstances in which `liquidityCheckApplicable` will return false. 

To understand, first note that market efficiency is measured as `maker / max(long, short)`. So if we reduce `maker`, we are reducing the efficiency. If the order is decreasing its maker position, and `marketParameter.makerCloseAlways` is set as true, then `liquidityCheckApplicable` returns false.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Order.sol#L108
```solidity
function liquidityCheckApplicable(
    Order memory self,
    MarketParameter memory marketParameter
) internal pure returns (bool) {
    return !marketParameter.closed &&
        ((self.maker.isZero()) || !marketParameter.makerCloseAlways || increasesMaker(self)) &&
        ((self.long.isZero() && self.short.isZero()) || !marketParameter.takerCloseAlways || increasesTaker(self));
}
```

Failing to check the market efficiency when a position change results in reduced market efficiency allows the market to become dangerously inefficient. Furthermore, since this market inefficiency check will be properly executed when orders which increase efficiency are executed, those orders will be reverted as the current market inefficiency will be insufficient, exacerbating the efficiency problem of the market.

## Impact

Core market invariant broken, leading to significant protocol risk including potential market insolvency.

## Code Snippet

See 'Vulnerability Detail' for code snippets.

## Tool used

Manual Review

## Recommendation

Consider removing `liquidityCheckApplicable` check from efficiency invariant check.