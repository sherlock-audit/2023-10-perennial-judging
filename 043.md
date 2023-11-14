Vast Glossy Tapir

medium

# interfaceFee Incorrectly converted  uint40 when stored

## Summary
The `interfaceFee.amount` is currently defined as `uint48` , with a maximum value of approximately `281m`.
However, it is incorrectly converted to `uint40` when saved, `uint40(UFixed6.unwrap(newValue.interfaceFee.amount))`, which means the maximum value can only be approximately `1.1M`. 
If a user sets an order where `interfaceFee.amount` is greater than `1.1M`, the order can be saved successfully
but the actual stored value may be truncated to `0`. 
This is not what the user expects, and the user may think that the order has been set, but in reality, it is an incorrect order. Although a fee of `1.1M` is large, it is not impossible.

## Vulnerability Detail

`interfaceFee.amount` is defined as `uint48`
the legality check also uses `type(uint48).max`, but `uint40` is used when saving.

```solidity
struct StoredTriggerOrder {
    /* slot 0 */
    uint8 side;                // 0 = maker, 1 = long, 2 = short, 3 = collateral
    int8 comparison;           // -2 = lt, -1 = lte, 0 = eq, 1 = gte, 2 = gt
    uint64 fee;                // <= 18.44tb
    int64 price;               // <= 9.22t
    int64 delta;               // <= 9.22t
@>  uint48 interfaceFeeAmount; // <= 281m

    /* slot 1 */
    address interfaceFeeReceiver;
    bool interfaceFeeUnwrap;
    bytes11 __unallocated0__;
}

library TriggerOrderLib {
    function store(TriggerOrderStorage storage self, TriggerOrder memory newValue) internal {
        if (newValue.side > type(uint8).max) revert TriggerOrderStorageInvalidError();
        if (newValue.comparison > type(int8).max) revert TriggerOrderStorageInvalidError();
        if (newValue.comparison < type(int8).min) revert TriggerOrderStorageInvalidError();
        if (newValue.fee.gt(UFixed6.wrap(type(uint64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.price.gt(Fixed6.wrap(type(int64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.price.lt(Fixed6.wrap(type(int64).min))) revert TriggerOrderStorageInvalidError();
        if (newValue.delta.gt(Fixed6.wrap(type(int64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.delta.lt(Fixed6.wrap(type(int64).min))) revert TriggerOrderStorageInvalidError();
@>      if (newValue.interfaceFee.amount.gt(UFixed6.wrap(type(uint48).max))) revert TriggerOrderStorageInvalidError();

        self.value = StoredTriggerOrder(
            uint8(newValue.side),
            int8(newValue.comparison),
            uint64(UFixed6.unwrap(newValue.fee)),
            int64(Fixed6.unwrap(newValue.price)),
            int64(Fixed6.unwrap(newValue.delta)),
@>          uint40(UFixed6.unwrap(newValue.interfaceFee.amount)),
            newValue.interfaceFee.receiver,
            newValue.interfaceFee.unwrap,
            bytes11(0)
        );
    }
```

We can see that when saving, it is forcibly converted to `uint40`, as in `uint40(UFixed6.unwrap(newValue.interfaceFee.amount))`. The order can be saved successfully, but the actual storage may be truncated to `0`.

## Impact
For orders where `interfaceFee.amount` is greater than `1.1M`, the order can be saved successfully, but the actual storage may be truncated to `0`. 
This is not what users expect and may lead to incorrect fee payments when the order is executed. 
Although a fee of `1.1M` is large, it is not impossible.

## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/types/TriggerOrder.sol#L106
## Tool used

Manual Review

## Recommendation

```diff
library TriggerOrderLib {
    function store(TriggerOrderStorage storage self, TriggerOrder memory newValue) internal {
        if (newValue.side > type(uint8).max) revert TriggerOrderStorageInvalidError();
        if (newValue.comparison > type(int8).max) revert TriggerOrderStorageInvalidError();
        if (newValue.comparison < type(int8).min) revert TriggerOrderStorageInvalidError();
        if (newValue.fee.gt(UFixed6.wrap(type(uint64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.price.gt(Fixed6.wrap(type(int64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.price.lt(Fixed6.wrap(type(int64).min))) revert TriggerOrderStorageInvalidError();
        if (newValue.delta.gt(Fixed6.wrap(type(int64).max))) revert TriggerOrderStorageInvalidError();
        if (newValue.delta.lt(Fixed6.wrap(type(int64).min))) revert TriggerOrderStorageInvalidError();
        if (newValue.interfaceFee.amount.gt(UFixed6.wrap(type(uint48).max))) revert TriggerOrderStorageInvalidError();

        self.value = StoredTriggerOrder(
            uint8(newValue.side),
            int8(newValue.comparison),
            uint64(UFixed6.unwrap(newValue.fee)),
            int64(Fixed6.unwrap(newValue.price)),
            int64(Fixed6.unwrap(newValue.delta)),
-          uint40(UFixed6.unwrap(newValue.interfaceFee.amount)),
+          uint48(UFixed6.unwrap(newValue.interfaceFee.amount)),
            newValue.interfaceFee.receiver,
            newValue.interfaceFee.unwrap,
            bytes11(0)
        );
    }
```