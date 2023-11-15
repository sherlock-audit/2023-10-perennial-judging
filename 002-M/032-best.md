Bouncy Corduroy Chimpanzee

medium

# `MultiInvoker._latest` calculates incorrect closable for the current oracle version causing some liquidations to revert

## Summary

`closable` is the value calculated as the maximum possible position size that can be closed even if some pending position updates are invalidated due to invalid oracle version. There is one tricky edge case at the current oracle version which is calculated incorrectly in `MultiInvoker` (and also in `Vault`). This happens when pending position is updated in the current active oracle version: it is allowed to set this current position to any value conforming to closable of the **previous** pending (or latest) position. For example:
1. latest settled position = 10
2. user calls update(20) - pending position at t=200 is set to 20. If we calculate `closable` normally, it will be 10 (latest settled position).
3. user calls update(0) - pending position at t=200 is set to 0. This is valid and correct. It looks as if we've reduced position by 20, bypassing the `closable = 10` value, but in reality the only enforced `closable` is the previous one (for latest settled position in the example, so it's 10) and it's enforced as a change from previous position, not from current.

Now, if the step 3 happened in the next oracle version, so
3. user calls update(0) - pending position at t=300 will revert, because user can't close more than 10, and he tries to close 20.

So in such tricky edge case, `MultiInvoker` (and `Vault`) will calculate `closable = 10` and will try to liquidate with position = 20-10 = 10 instead of 0 and will revert, because `Market._invariant` will calculate `closable = 10` (latest = 10, pending = 10, closable = latest = 10), but it must be 0 to liquidate (step 3. in the example above)

In `Vault` case, this is less severe as the market will simply allow to redeem and will close smaller amount than it actually can.

## Vulnerability Detail

When `Market` calculates `closable`, it's calculated starting from latest settled position up to (but not including) current position:
```solidity
// load pending positions
for (uint256 id = context.local.latestId + 1; id < context.local.currentId; id++)
    _processPendingPosition(context, _loadPendingPositionLocal(context, account, id));
```

Pay attention to `id < context.local.currentId` - the loop doesn't include currentId.

After the current position is updated to a new user specified value, only then the current position is processed and closable now includes **new** user position change from the previous position:

```solidity
function _update(
    ...
    // load
    _loadUpdateContext(context, account);
    ...
    context.currentPosition.local.update(collateral);
    ...
    // process current position
    _processPendingPosition(context, context.currentPosition.local);
    ...
    // after
    _invariant(context, account, newOrder, collateral, protected);
```

The `MultiInvoker._latest` logic is different and simply includes calculation of `closable` for all pending positions:

```solidity
for (uint256 id = local.latestId + 1; id <= local.currentId; id++) {

    // load pending position
    Position memory pendingPosition = market.pendingPositions(account, id);
    pendingPosition.adjust(latestPosition);

    // virtual settlement
    if (pendingPosition.timestamp <= latestTimestamp) {
        if (!market.oracle().at(pendingPosition.timestamp).valid) latestPosition.invalidate(pendingPosition);
        latestPosition.update(pendingPosition);

        previousMagnitude = latestPosition.magnitude();
        closableAmount = previousMagnitude;

    // process pending positions
    } else {
        closableAmount = closableAmount
            .sub(previousMagnitude.sub(pendingPosition.magnitude().min(previousMagnitude)));
        previousMagnitude = latestPosition.magnitude();
    }
}
```

The same incorrect logic is in a `Vault`:

```solidity
// pending positions
for (uint256 id = marketContext.local.latestId + 1; id <= marketContext.local.currentId; id++)
    previousClosable = _loadPosition(
        marketContext,
        marketContext.currentAccountPosition = registration.market.pendingPositions(address(this), id),
        previousClosable
    );
```

## Impact

In the following edge case:
- current oracle version = oracle version of the pending position in currentId index
- AND this (current) pending position increases compared to previous pending/settled position

The following can happen:
- liquidation via `MultiInvoker` will revert (medium impact)
- vault's `maxRedeem` amount will be smaller than actual allowed amount, position will be reduced by a smaller amount than they actually can (low impact)

## Code Snippet

`MultiInvoker` calculates `closable` by simply iterating all pending positions:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L412-L433

`Vault` calculates it the same way (iterating all positions):
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/lib/StrategyLib.sol#L164-L170

`Market` calculates `closable` up to (but not including) current position:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L287-L289

and then the current position (after being updated to user values) is processed (closable enforced/calculated):
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L362-L363

## Tool used

Manual Review

## Recommendation

When calculating `closable` in `MultiInvoker` and `Vault`, add the following logic:
- if timestamp of pending position at index currentId equals current oracle version, then add the difference between position size at currentId and previous position size to `closable` (both when that position increases and decreases).

For example, if
- latest settled position = 10
- pending position at t=200 = 20
then
initialize `closable` to `10` (latest)
add (pending-latest) = (20-10) to closable (`closable` = 20)
