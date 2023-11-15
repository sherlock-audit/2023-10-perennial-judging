Calm Gingham Beaver

medium

# TriggerOrder.execute withdraws full position if the provided delta is 0

## Summary

`TriggerOrder.update` is used to update a given position according to a `side` and `delta` such that the output is the resulting position after the order. In the case where the delta is 0, i.e. the position should be unchanged, the position is unintentionally updated such that the entire position is withdrawn.

## Vulnerability Detail

In `TriggerOrder.update`, if `delta` is zero, we withdraw the entire position of the given `side`. 

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L49
```solidity
currentPosition.maker = self.delta.isZero() ?
        // @audit should set as currentPosition.maker instead of setting as 0
        UFixed6Lib.ZERO :
        UFixed6Lib.from(Fixed6Lib.from(currentPosition.maker).add(self.delta));
```

This is contrary to the intended effect of leaving the position unchanged.

## Impact

Unexpected position change incurred by user.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Leave the position unchanged if the delta is 0.