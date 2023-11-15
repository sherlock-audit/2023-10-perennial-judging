Calm Gingham Beaver

high

# Can drain market contracts via self-liquidating positions to negative collateral amounts

## Summary

It's possible that positions end up with negative collateral amounts during liquidation since the relevant invariant is skipped. It's also possible that the minMaintenance needed to initially create a deposit is less tokens than the minMaintenance liquidationFee which the liquidator receives for liquidation. As a result, attackers can strategically self-liquidate their positions to negative collateral until the markets are drained.

## Vulnerability Detail

In `Market._invariant` we enforce that the collateral of a position doesn't end up negative.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L630
```solidity
if (collateral.lt(Fixed6Lib.ZERO) && context.pendingCollateral.lt(Fixed6Lib.ZERO))
      revert MarketInsufficientCollateralError();
```

However, during liquidations, we return from this function early and never actually execute this check.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L592
```solidity
if (protected) return; // The following invariants do not apply to protected position updates (liquidations)
```

Positions must exceed a minimum maintenance amount as defined by `riskParameter.minMaintenance` to be created. This also happens to be the minimum value used as the liquidation fee. Maintenance is dependent on the price, so as the price moves up, minMaintenance moves down, and as the price moves down, minMaintenance moves up.

We can exploit this by intentionally creating a position that will be at minMaintenance immediately before a price commitment which we know will bring it below minMaintenance, where we can then liquidate the position receiving greater than the initially deposited collateral amount as liquidation fee.

Attack works as follows:
- Immediately before processing a price commitment, attacker opens a position where it is at minMaintenance but will not be after the price is committed
  - e.g. if price is decreasing, open a long position at minMaintenance, if price is increasing, open a short position at minMaintenance
- Attacker commits price
- Attacker immediately liquidates own position, retrieving minMaintenance as liquidation fee
  - Since minMaintenance is determined by the current price, the amount of collateral deposited was less (same value, different price), than the amount received as the liquidation fee
  - Results in attackers collateral going negative
- The attacker is free to repeat this attack with every price commitment and even with multiple accounts simultaneously until the market is drained

## Impact

Attacker can drain every market of its collateral.

## Code Snippet

See 'Vulnerability Detail' for snippets.

## Tool used

Manual Review

## Recommendation

Ensure that the user has sufficient collateral after a position change regardless of whether it's a liquidation or not.
