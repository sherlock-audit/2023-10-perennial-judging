Bouncy Corduroy Chimpanzee

medium

# `MultiInvoker._latest` will return `latestPrice = 0` when latest oracle version is invalid causing liquidation to send 0 fee to liquidator or incorrect order execution

## Summary

There was a slight change of oracle versions handling in 2.1: now each requested oracle version must be commited, either as valid or invalid. This means that now the latest version can be invalid (`price = 0`). This is handled correctly in `Market`, which only uses timestamp from the latest oracle version, but the price comes either from latest version (if valid) or `global.latestPrice` (if invalid).

However, `MultiInvoker` always uses price from `oracle.latest` without verifying if it's valid, meaning it will return `latestPrice = 0` if the latest oracle version is invalid. This is returned from the `_latest` function.

Such latest price = 0 leads to 2 main problems:
- Liquidations orders in MultiInvoker will send 0 liquidation fee to liquidator (will liquidate for free)
- Some `TriggerOrder`s will trigger incorrectly (`canExecuteOrder` will return true when the real price didn't reach the trigger price, or false even if the real prices reached the trigger price)

## Vulnerability Detail

`MultiInvoker._latest` has the following code for latest price assignment:
```solidity
OracleVersion memory latestOracleVersion = market.oracle().latest();
latestPrice = latestOracleVersion.price;
IPayoffProvider payoff = market.payoff();
if (address(payoff) != address(0)) latestPrice = payoff.payoff(latestPrice);
```

This `latestPrice` is what's returned from the `_latest`, it isn't changed anywhere else. Notice that there is no check for latest oracle version validity.

And this is the code for `KeeperOracle._commitRequested`:
```solidity
function _commitRequested(OracleVersion memory version) private returns (bool) {
    if (block.timestamp <= (next() + timeout)) {
        if (!version.valid) revert KeeperOracleInvalidPriceError();
        _prices[version.timestamp] = version.price;
    }
    _global.latestIndex++;
    return true;
}
```

Notice that commits made outside the timeout window simply increase `_global.latestIndex` without assigning `_prices`, meaning it remains 0 (invalid). This means that latest oracle version will return price=0 and will be invalid if commited after the timeout from request time has passed.

Price returned by `_latest` is used when calculating liquidationFee:
```solidity
function _liquidationFee(IMarket market, address account) internal view returns (Position memory, UFixed6, UFixed6) {
    // load information about liquidation
    RiskParameter memory riskParameter = market.riskParameter();
@@@ (Position memory latestPosition, Fixed6 latestPrice, UFixed6 closableAmount) = _latest(market, account);

    // create placeholder order for liquidation fee calculation (fee is charged the same on all sides)
    Order memory placeholderOrder;
    placeholderOrder.maker = Fixed6Lib.from(closableAmount);

    return (
        latestPosition,
        placeholderOrder
@@@         .liquidationFee(OracleVersion(latestPosition.timestamp, latestPrice, true), riskParameter)
            .min(UFixed6Lib.from(market.token().balanceOf(address(market)))),
        closableAmount
    );
}
```

`liquidationFee` calculation in order multiplies order size by `latestPrice`, meaning it will be 0 when price = 0. This liquidation fee is then used in `market.update` for liquidation fee to receive by liquidator:
```solidity
    function _liquidate(IMarket market, address account, bool revertOnFailure) internal isMarketInstance(market) {
@@@     (Position memory latestPosition, UFixed6 liquidationFee, UFixed6 closable) = _liquidationFee(market, account);
        Position memory currentPosition = market.pendingPositions(account, market.locals(account).currentId);
        currentPosition.adjust(latestPosition);

        try market.update(
                account,
                currentPosition.maker.isZero() ? UFixed6Lib.ZERO : currentPosition.maker.sub(closable),
                currentPosition.long.isZero() ? UFixed6Lib.ZERO : currentPosition.long.sub(closable),
                currentPosition.short.isZero() ? UFixed6Lib.ZERO : currentPosition.short.sub(closable),
@@@             Fixed6Lib.from(-1, liquidationFee),
                true
```

This means liquidator will receive 0 fee for the liquidation.

It is also used in `canExecuteOrder`:
```solidity
    function _executeOrder(address account, IMarket market, uint256 nonce) internal {
        if (!canExecuteOrder(account, market, nonce)) revert MultiInvokerCantExecuteError();
...
    function canExecuteOrder(address account, IMarket market, uint256 nonce) public view returns (bool) {
        TriggerOrder memory order = orders(account, market, nonce);
        if (order.fee.isZero()) return false;
@@@     (, Fixed6 latestPrice, ) = _latest(market, account);
@@@     return order.fillable(latestPrice);
    }
```

Meaning `canExecuteOrder` will do comparision with price = 0 instead of real latest price. For example: limit buy order to buy when price <= 1000 (when current price = 1100) will trigger and execute buy at the price = 1100 instead of 1000 or lower.

## Impact

- liquidation done after invalid oracle version via `MultiInvoker` `LIQUIDATE` action will charge and send 0 liquidation fee from the liquidating account, thus liquidator loses these funds.
- some orders with comparison of type -1 (<= price) will incorrectly trigger and will be executed when price is far from reaching the trigger price. This loses user funds due to unexpected execution price of the pending order.

## Code Snippet

`_latest` simply takes `oracle.latest` price, which can be 0, without any check for oracle version validity:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L394-L402

## Tool used

Manual Review

## Recommendation

`_latest` should replicate the process for the latest price from `Market` instead of using price from the oracle's latest version:
- if the latest oracle version is valid, then use its price
- if the latest oracle version is invalid, then iterate all global pending positions backwards and use price of any valid oracle version at the position.
- if all pending positions are at invalid oracles, use market's `global.latestPrice`