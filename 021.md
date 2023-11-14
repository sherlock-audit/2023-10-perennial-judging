Melted Tawny Ladybug

high

# Keeper payment for the commit of price doesn't consider global callbacks amount

## Summary
Keepers can be no incentivized to commit prices, when there is not 1 global callback for per oracle version as it costs more gas for them, which is not refunded. Because of that it's possible that prices will not be committed, when not only 1 market requested it.
## Vulnerability Detail
When market need new oracle version for the position, then it [requests it from oracle service](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L369).

`KeeperOracle.request` function will store market that did the request to `_globalCallbacks`(currently this function has a problem that i have described in another report, but it will be fixed and this problem will remain). 
When price for oracle version is commited, then all markets that are registered as `_globalCallbacks` for that version [are settled](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L123-L124).

Keeper receive refund for committing oracle price. They call `KeeperFactory.commit` to provide new price and get refund.
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L163-L173
```solidity
    function commit(bytes32[] memory ids, uint256 version, bytes calldata data) external payable {
        bool valid = data.length != 0;
        Fixed6[] memory prices = valid ? _parsePrices(ids, version, data) : new Fixed6[](ids.length);
        uint256 numRequested;

        for (uint256 i; i < ids.length; i++)
            if (IKeeperOracle(address(oracles[ids[i]])).commit(OracleVersion(version, prices[i], valid)))
                numRequested++;

        if (numRequested != 0) _handleCommitKeep(numRequested);
    }
```

So every time, when `IKeeperOracle(address(oracles[ids[i]])).commit` is called, function returns if it was requested price or not. Only requested prices are refunded and in this case `numRequested` is increased. So in case if keeper provided at least 1 requested price, then `_handleCommitKeep(numRequested)` function is called, which will handle refund.

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L221-L224
```solidity
    function _handleCommitKeep(uint256 numRequested)
        internal virtual
        keep(commitKeepConfig(numRequested), msg.data[0:0], 0, "")
    { }
```
As you can see, `numRequested` is passed to `commitKeepConfig` function, which will [increase refund config values according to count of valid prices](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L177-L182).

The difference between `commit` and `settle` function for keeper is that settle refund all executed gas as [it has `keep` modifier](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L204), while commit does that in other way as call of separate function after the job is done. Thus it doesn't track and refund spent gas. It just pays according to the provided config.

So we have seen that payment for the keeper is increased with each requested price. But the problem is that for each price there can be a lot of `_globalCallbacks` registered. And for all of them keeper [should call `settle`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L124). This is not reflected in the refund model(only one market per oracle version is considered), which means that in case if more than 1 market is registered in the callbacks, then it decreases profit for the keeper until it becomes fully unprofitable for them. As result keepers will stop committing prices for such versions. Also it will be possible to block keepers by requesting same version from different markets in order to dos settlement for specific price version. For example, when user sees the price that will liquidate his position, and price for liquidation was requested, then user can do some position changes on other markets to increases number of global callbacks and make keeper not commit price.
## Impact
Keepers are not incentivized to provide prices.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Maybe `KeeperOracle.commit` should return number of callbacks that were settled instead of bool. So `numRequested` then depends on callbacks amount that were executed. As result keeper receives fair payment and happy to provide prices.