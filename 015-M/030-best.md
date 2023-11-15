Bouncy Corduroy Chimpanzee

medium

# `TriggerOrder.comparison` can have values from -2 to 2, but only -1 to 1 are implemented

## Summary

`TriggerOrder.comparison` value is described as:
```solidity
int8 comparison;           // -2 = lt, -1 = lte, 0 = eq, 1 = gte, 2 = gt
```

So it can have values from -2 to 2. However, implementation (`fillable`) only implements values of 1 (>=) and -1 (<=):
```solidity
function fillable(TriggerOrder memory self, Fixed6 latestPrice) internal pure returns (bool) {
    if (self.comparison == 1) return latestPrice.gte(self.price);
    if (self.comparison == -1) return latestPrice.lte(self.price);
    return false;
}
```

This means that if user creates an order with values -2, 0 or 2 - it will never execute (`fillable` will return false for these values).

## Vulnerability Detail

User can create advanced orders via `MultiInvoker`, such as take profit or stop loss, which can have different price triggers for execution. These values are checked off-chain by order execution bots and on-chain before order execution:
```solidity
    function _executeOrder(address account, IMarket market, uint256 nonce) internal {
        if (!canExecuteOrder(account, market, nonce)) revert MultiInvokerCantExecuteError();
...
    function canExecuteOrder(address account, IMarket market, uint256 nonce) public view returns (bool) {
        TriggerOrder memory order = orders(account, market, nonce);
        if (order.fee.isZero()) return false;
        (, Fixed6 latestPrice, ) = _latest(market, account);
@@@     return order.fillable(latestPrice);
    }
```

If the user creates an order with `comparison` value of -2 (<), 2 (>), or 0 (=), such order won't be able to execute as `canExecuteOrder` will return false and order execution will always revert.

## Impact

User with `comparison` values of -2, 2 or 0 expects his order to execute when the condition is met, but it never executes, possibly causing loss of funds of loss of profitable opportunity for the user.

## Code Snippet

Description of `comparison` values:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L21

Implementation of `comparison` (`fillable`):
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L40-L44

## Tool used

Manual Review

## Recommendation

Implement correct comparison for all the values.