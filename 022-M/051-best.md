Vast Glossy Tapir

medium

# TriggerOrder when delta.isZero should set position to MAGIC_VALUE_FULLY_CLOSED_POSITION

## Summary
This update has modified `delta==0` to represent the closing of the position. 
The current implementation is to set `postion = 0`. 
However, according to the implementation of `market.sol`, 
it should use `MAGIC_VALUE_FULLY_CLOSED_POSITION = type(uint256).max - 1`. 
This is because `market.sol` needs to consider `context.closable`, and passing `0` may not necessarily close it.

## Vulnerability Detail

when `order.delta==0` will pass position == 0
```solidity
    function execute(TriggerOrder memory self, Position memory currentPosition) internal pure {
        // update position
        if (self.side == 0)
                currentPosition.maker = self.delta.isZero() ?
@>              UFixed6Lib.ZERO :
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.maker).add(self.delta));
        if (self.side == 1)
            currentPosition.long = self.delta.isZero() ?
@>              UFixed6Lib.ZERO :
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.long).add(self.delta));
        if (self.side == 2)
            currentPosition.short = self.delta.isZero() ?
@>              UFixed6Lib.ZERO :
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.short).add(self.delta));

        // update collateral (override collateral field in position since it is not used in this context)
        // Handles collateral withdrawal magic value
        currentPosition.collateral = (self.side == 3) ?
            (self.delta.eq(Fixed6.wrap(type(int64).min)) ? Fixed6Lib.MIN : self.delta) :
            Fixed6Lib.ZERO;
    }
```

but if need a full close position,  a better choice is to pass  `MAGIC_VALUE_FULLY_CLOSED_POSITION = UFixed6.wrap(type(uint256).max - 1)`

```solidity
    function _processPositionMagicValue(
        Context memory context,
        UFixed6 currentPosition,
        UFixed6 newPosition
    ) private pure returns (UFixed6) {
        if (newPosition.eq(MAGIC_VALUE_UNCHANGED_POSITION))
            return currentPosition;
@>      if (newPosition.eq(MAGIC_VALUE_FULLY_CLOSED_POSITION)) {
            if (currentPosition.isZero()) return currentPosition;
@>          return context.previousPendingMagnitude.sub(context.closable.min(context.previousPendingMagnitude));
        }
        return newPosition;
    }
```

Due to the existence of `context.closable`, `0` may not necessarily close it.

## Impact

`market.sol` will take `closable` into account. A better choice is `MAGIC_VALUE_FULLY_CLOSED_POSITION`, as `0` may not necessarily close it.

## Code Snippet

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L46-L67

## Tool used

Manual Review

## Recommendation

```diff
    function execute(TriggerOrder memory self, Position memory currentPosition) internal pure {
        // update position
        if (self.side == 0)
            currentPosition.maker = self.delta.isZero() ?
-               UFixed6Lib.ZERO :
+              MAGIC_VALUE_FULLY_CLOSED_POSITION:
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.maker).add(self.delta));
        if (self.side == 1)
            currentPosition.long = self.delta.isZero() ?
-               UFixed6Lib.ZERO :
+              MAGIC_VALUE_FULLY_CLOSED_POSITION:
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.long).add(self.delta));
        if (self.side == 2)
            currentPosition.short = self.delta.isZero() ?
-               UFixed6Lib.ZERO :
+              MAGIC_VALUE_FULLY_CLOSED_POSITION:
                UFixed6Lib.from(Fixed6Lib.from(currentPosition.short).add(self.delta));

        // update collateral (override collateral field in position since it is not used in this context)
        // Handles collateral withdrawal magic value
        currentPosition.collateral = (self.side == 3) ?
            (self.delta.eq(Fixed6.wrap(type(int64).min)) ? Fixed6Lib.MIN : self.delta) :
            Fixed6Lib.ZERO;
    }
}
```