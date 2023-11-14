Vast Glossy Tapir

medium

# invoke() early return eth

## Summary
In `MultiInvoker.invoke()`, the return of the remaining `eth` is incorrectly placed inside the loop, resulting in the return of the remaining `eth` after the first `action`.
This results in the second and subsequent `actions` that need to use `eth`, there will be no `eth` available, and `invoke()` will `revert`

## Vulnerability Detail
in `MultiInvoker.invoke()`
If there is any `eth` left after `invocations` is executed, we should return the `eth` to the user to avoid `eth` remaining in the contract.
The code is as follows
```solidity
    function invoke(Invocation[] calldata invocations) external payable {
        for(uint i = 0; i < invocations.length; ++i) {
            Invocation memory invocation = invocations[i];
.......
            } else if (invocation.action == PerennialAction.COMMIT_PRICE) {
                (address oracleProviderFactory, uint256 value, bytes32[] memory ids, uint256 version, bytes memory data, bool revertOnFailure) =
                    abi.decode(invocation.args, (address, uint256, bytes32[], uint256, bytes, bool));

                _commitPrice(oracleProviderFactory, value, ids, version, data, revertOnFailure);
            } else if (invocation.action == PerennialAction.LIQUIDATE) {
                (IMarket market, address account, bool revertOnFailure) = abi.decode(invocation.args, (IMarket, address, bool));

                _liquidate(market, account, revertOnFailure);
            } else if (invocation.action == PerennialAction.APPROVE) {
                (address target) = abi.decode(invocation.args, (address));

                _approve(target);
            }

            // Eth must not remain in this contract at rest
@>          payable(msg.sender).transfer(address(this).balance);
        }
    }
```

As we can see from the code above, the current implementation of returning `eth` is placed inside the loop. 
This way the first `action` is executed and then the `eth` is returned.
If the `action` that requires `eth`, such as `PerennialAction.COMMIT_PRICE`, is the second `action`, by the time it's ready to be executed, but all the `eth` have already been returned, which will cause this `action` to fail to execute.

## Impact

Early return of `eth` will result in `actions` requiring `eth`  after the first `action` not being executed.

## Code Snippet

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-extensions/contracts/MultiInvoker.sol#L166

## Tool used

Manual Review

## Recommendation

```diff
    function invoke(Invocation[] calldata invocations) external payable {
        for(uint i = 0; i < invocations.length; ++i) {
            Invocation memory invocation = invocations[i];
...
            } else if (invocation.action == PerennialAction.APPROVE) {
                (address target) = abi.decode(invocation.args, (address));

                _approve(target);
            }

            // Eth must not remain in this contract at rest
-           payable(msg.sender).transfer(address(this).balance);
        }
+       payable(msg.sender).transfer(address(this).balance);
    }
```