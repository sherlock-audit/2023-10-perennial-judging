Melted Tawny Ladybug

medium

# MultiInvoker doesn't pay keepers refund for l1 calldata

## Summary
MultiInvoker doesn't pay keepers refund for l1 calldata, as result keepers can be not incentivized to execute orders.
## Vulnerability Detail
MultiInvoker contract allows users to create orders, which then [can be executed by keepers](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L149). For his job, keeper receives fee from order's creator. This fee payment [is handled by `_handleKeep` function](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L446).

The function will call `keep` modifier and [will craft `KeepConfig`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L473-L478) which contains `keepBufferCalldata`, which is flat fee for l1 calldata of this call.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/root/contracts/attribute/Kept/Kept.sol#L74-L95
```solidity
    modifier keep(
        KeepConfig memory config,
        bytes calldata applicableCalldata,
        uint256 applicableValue,
        bytes memory data
    ) {
        uint256 startGas = gasleft();


        _;


        uint256 applicableGas = startGas - gasleft();
        (UFixed18 baseFee, UFixed18 calldataFee) = (
            _baseFee(applicableGas, config.multiplierBase, config.bufferBase),
            _calldataFee(applicableCalldata, config.multiplierCalldata, config.bufferCalldata)
        );


        UFixed18 keeperFee = UFixed18.wrap(applicableValue).add(baseFee).add(calldataFee).mul(_etherPrice());
        _raiseKeeperFee(keeperFee, data);
        keeperToken().push(msg.sender, keeperFee);


        emit KeeperCall(msg.sender, applicableGas, applicableValue, baseFee, calldataFee, keeperFee);
    }
```

This modifier should calculate amount of tokens that should be refunded to user and then raise it. We are interested not in whole modifier, but in calldata handling. To do that we call `_calldataFee` function. This function [does nothing](https://github.com/sherlock-audit/2023-10-perennial/blob/main/root/contracts/attribute/Kept/Kept.sol#L42-L46) in the `Kept` contract and is overrided in the [`Kept_Arbitrum`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/root/contracts/attribute/Kept/Kept_Arbitrum.sol#L21-L32) and [`Kept_Optimism`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/root/contracts/attribute/Kept/Kept_Optimism.sol#L20-L31).

The problem is that MultiInvoker is only one and it [just extends `Keept`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L19C41-L19C46). As result his `_calldataFee` function will always return 0, which means that calldata fee will not be added to the refund of keeper.
## Impact
Keeper will not be incentivized to execute orders.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
You need to implement 2 versions of MultiInvoker: for optimism(`Kept_Optimism`) and arbitrum(`Kept_Arbitrum`). 