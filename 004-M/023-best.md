Bouncy Corduroy Chimpanzee

medium

# It is possible to open and liquidate your own position in 1 transaction to overcome efficiency and liquidity removal limits at almost no cost

## Summary

In 2.0 audit the [issue 104](https://github.com/sherlock-audit/2023-07-perennial-judging/issues/104) was fixed but not fully and it's still possible, in a slightly different way. This wasn't found in the fix review contest. The fix introduced margined and maintained amounts, so that margined amount is higher than maintained one. However, when collateral is withdrawn, only the current (pending) position is checked by margined amount, the largest position (including latest settled) is checked by maintained amount. This still allows to withdraw funds up to the edge of being liquidated, if margined current position amount <= maintained settled position amount. So the new way to liquidate your own position is to reduce your position and then do the same as in 2.0 issue.

This means that it's possible to be at almost liquidation level intentionally and moreover, the current oracle setup allows to open and immediately liquidate your own position in 1 transaction, effectively bypassing efficiency and liquidity removal limits, paying only the keeper (and possible position open/close) fees, causing all kinds of malicious activity which can harm the protocol.

## Vulnerability Detail

`Market._invariant` verifies margined amount only for the current position:
```solidity
if (
    !context.currentPosition.local.margined(context.latestVersion, context.riskParameter, context.pendingCollateral)
) revert MarketInsufficientMarginError();
```

All the other checks (max pending position, including settled amount) are for maintained amount:
```solidity
if (
    !PositionLib.maintained(context.maxPendingMagnitude, context.latestVersion, context.riskParameter, context.pendingCollateral)
) revert MarketInsufficientMaintenanceError();
```

The user can liquidate his own position with 100% guarantee in 1 transaction by following these steps:
1. It can be done only on existing settled position
2. Record Pyth oracle prices with signatures until you encounter a price which is higher (or lower, depending on your position direction) than latest oracle version price by any amount.
3. In 1 transaction do the following:
3.1. Reduce your position by `(margin / maintenance)` and make the position you want to liquidate at exactly the edge of liquidation: withdraw maximum allowed amount. Position reduction makes margined(current position) = maintained(settled position), so it's possible to withdraw up to be at the edge of liquidation.
3.2. Commit non-requested oracle version with the price recorded earlier (this price makes the position liquidatable)
3.3. Liquidate your position (it will be allowed, because the position generates a minimum loss due to price change and becomes liquidatable)

Since all liquidation fee is given to user himself, liquidation of own position is almost free for the user (only the keeper and position open/close fee is paid if any).

## Impact

There are different malicious actions scenarios possible which can abuse this issue and overcome efficiency and liquidity removal limitations (as they're ignored when liquidating positions), such as:
- Combine with the other issues for more severe effect to be able to abuse them in 1 transaction (for example, make `closable = 0` and liquidate your position while increasing to max position size of 2^62-1 - all in 1 transaction)
- Open large maker and long or short position, then liquidate maker to cause mismatch between long/short and maker (socialize positions). This will cause some chaos in the market, disbalance between long and short profit/loss and users will probably start leaving such chaotic market, so while this attack is not totally free, it's cheap enough to drive users away from competition.
- Open large maker, wait for long and/or short positions from normal users to accumulate, then liquidate most of the large maker position, which will drive taker interest very high and remaining small maker position will be able to accumulate big profit with a small risk.

## Code Snippet

`Market._invariant` verifies margined amount only for the current position:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L605-L607

All the other checks (max pending position, including settled amount) are for maintained amount:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L609-L611

## Tool used

Manual Review

## Recommendation

If collateral is withdrawn or order increases position, verify `maxPendingMagnitude` with `margined` amount. If position is reduced or remains unchanged AND collateral is not withdrawn, only then `maxPendingMagnitude` can be verified with `maintained` amount.