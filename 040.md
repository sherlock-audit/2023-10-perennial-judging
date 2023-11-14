Vast Glossy Tapir

medium

# MultiInvoker orders that retrieve all collaterals and have a fee cannot be executed.

## Summary
in `MultiInvoker.sol`
The current order of executing orders is:
1. Execute the order content
2. Execute `chargeFee()` to pay `interfaceFee`

However, if the order type is to retrieve all collateral (`delta==type(int64).min`), the first step will take away all the collateral. When paying the `interfaceFee` in the second step, there is no collateral left, leading to `revert`.

For this type of order, we should first pay the `interfaceFee`, and then take away the remaining collateral.

## Vulnerability Detail
This update adds `side==3`, which is for collateral retrieval, and it allows users to set `delta==type(int64).min` to represent the retrieval of all collateral. 
It also adds the ability for orders to specify the payment of `interfaceFee`. The execution order is as follows:

1. Execute the order content
2. Execute `chargeFee()` to pay `interfaceFee`

The code is as follows:
`_executeOrder()` -> `_update()`
```solidity
   function _update(
        address account,
        IMarket market,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool wrap,
        InterfaceFee memory interfaceFee
    ) internal isMarketInstance(market) {
        Fixed18 balanceBefore =  Fixed18Lib.from(DSU.balanceOf());

        // collateral is transferred here as DSU then an optional interface fee is charged from it
        if (collateral.sign() == 1) _deposit(collateral.abs(), wrap);

        market.update(account, newMaker, newLong, newShort, collateral, false);

        Fixed6 withdrawAmount = Fixed6Lib.from(Fixed18Lib.from(DSU.balanceOf()).sub(balanceBefore));
@>      if (!withdrawAmount.isZero()) _withdraw(account, withdrawAmount.abs(), wrap);

        // charge interface fee
@>      _chargeFee(account, market, interfaceFee);
    }
```
`TriggerOrder.sol`
```solidity
    function execute(TriggerOrder memory self, Position memory currentPosition) internal pure {
        // update position
....

        // update collateral (override collateral field in position since it is not used in this context)
        // Handles collateral withdrawal magic value
        currentPosition.collateral = (self.side == 3) ?
@>          (self.delta.eq(Fixed6.wrap(type(int64).min)) ? Fixed6Lib.MIN : self.delta) :
            Fixed6Lib.ZERO;
    }
}
```

The order content is executed first. 
So if the order content is newly added with `side==3`, it takes the collateral and sets `delta==type(int64).min`. 
This way, the user will take away all the collateral, and by the time payment `interfaceFee` is required, there will be no collateral left. (The _chargeFee() is paid through collateral) 
This causes the order to fail forever.

For example, I set the order to take away all the collateral, my current collateral = 100000, and pay `interfaceFee = 5`. 
The normal logic of this order should be to pay `interfaceFee = 5`, and take away the remaining collateral = (100000 - 5).


## Impact

Orders that retrieve all collaterals and have an `interfaceFee` cannot be executed.

## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L200

## Tool used

Manual Review

## Recommendation

Pay `interfaceFee` first, then execute the order.

```diff
    function _update(
        address account,
        IMarket market,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool wrap,
        InterfaceFee memory interfaceFee
    ) internal isMarketInstance(market) {
        Fixed18 balanceBefore =  Fixed18Lib.from(DSU.balanceOf());

        // collateral is transferred here as DSU then an optional interface fee is charged from it
        if (collateral.sign() == 1) _deposit(collateral.abs(), wrap);

+       _chargeFee(account, market, interfaceFee);

        market.update(account, newMaker, newLong, newShort, collateral, false);

        Fixed6 withdrawAmount = Fixed6Lib.from(Fixed18Lib.from(DSU.balanceOf()).sub(balanceBefore));
        if (!withdrawAmount.isZero()) _withdraw(account, withdrawAmount.abs(), wrap);

        // charge interface fee
-       _chargeFee(account, market, interfaceFee);
    }
```