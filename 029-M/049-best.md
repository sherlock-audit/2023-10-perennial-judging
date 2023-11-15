Main Burgundy Cyborg

medium

# Latest version price might be updated incorrectly

## Summary
Latest version price might be updated incorrectly
## Vulnerability Detail
global.latestPrice is the price the market will rely on only when the latest oracle version is invalid. Otherwise, we use the price from the latest oracle version.
The problem is that context.latestVersion.price is always being overwritten by global.lastestPrice even when the oracle version is valid. It could lead to incorrect calculations in invariants

```solidity
function _settle(Context memory context, address account) private {
        context.latestPosition.global = _position.read();
        context.latestPosition.local = _positions[account].read();

        Position memory nextPosition;

        // settle
        while (
            context.global.currentId != context.global.latestId &&
            (nextPosition = _loadPendingPositionGlobal(context, context.global.latestId + 1))
                .ready(context.latestVersion)
        ) _processPositionGlobal(context, context.global.latestId + 1, nextPosition);

        while (
            context.local.currentId != context.local.latestId &&
            (nextPosition = _loadPendingPositionLocal(context, account, context.local.latestId + 1))
                .ready(context.latestVersion)
        ) {
            Fixed6 previousDelta = _pendingPositions[account][context.local.latestId].read().delta;
            _processPositionLocal(context, account, context.local.latestId + 1, nextPosition);
            _checkpointCollateral(context, account, previousDelta, nextPosition);
        }

        // sync
        if (context.latestVersion.timestamp > context.latestPosition.global.timestamp) {
            nextPosition = _loadPendingPositionGlobal(context, context.global.latestId);
            nextPosition.sync(context.latestVersion);
            _processPositionGlobal(context, context.global.latestId, nextPosition);
        }

        if (context.latestVersion.timestamp > context.latestPosition.local.timestamp) {
            nextPosition = _loadPendingPositionLocal(context, account, context.local.latestId);
            nextPosition.sync(context.latestVersion);
            _processPositionLocal(context, account, context.local.latestId, nextPosition);
        }

        // overwrite latestPrice if invalid
        >>>context.latestVersion.price = context.global.latestPrice; // always update with global latest price, which is only a fallback option

        _position.store(context.latestPosition.global);
        _positions[account].store(context.latestPosition.local);
    }
```
## Impact
It could lead to incorrect accounting in invariant check, leading to incorrect pass/revert.
## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L389-L456
## Tool used

Manual Review

## Recommendation
Consider this change
```solidity
        // overwrite latestPrice if invalid
        if (!context.latestVersion.valid) context.latestVersion.price = context.global.latestPrice; 
        _position.store(context.latestPosition.global);
        _positions[account].store(context.latestPosition.local);
    }
```