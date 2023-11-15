Bouncy Corduroy Chimpanzee

medium

# `KeeperOracle.commit` will revert and won't work for all markets if any single `Market` is paused.

## Summary

According to protocol design (from KeeperOracle comments), multiple markets may use the same KeeperOracle instance:
```solidity
/// @dev One instance per price feed should be deployed. Multiple products may use the same
///      KeeperOracle instance if their payoff functions are based on the same underlying oracle.
///      This implementation only supports non-negative prices.
```

However, if `KeeperOracle` is used by several `Market` instances, and one of them makes a request and is then paused before the settlement, `KeeperOracle` will be temporarily bricked until `Market` is unpaused. This happens, because `KeeperOracle.commit` will revert in market callback, as `commit` iterates through all requested markets and calls `update` on all of them, and `update` reverts if the market is paused.

This means that pausing of just 1 market will basically stop trading in all the other markets which use the same `KeeperOracle`, disrupting protocol usage. When `KeeperOracle.commit` always reverts, it's also impossible to switch oracle provider from upstream `OracleFactory`, because provider switch still requires the latest version of previous oracle to be commited, and it will be impossible to commit it (both valid or invalid, requested or unrequested).

Additionally, the market's `update` can also revert for some other reasons, for example if maker exceeds the maker limit after invalid oracle as described in the other issue.

And for another problem (although a low severity, but caused in the same lines), if too many markets are authorized to call `KeeperOracle.request`, the markets callback gas usage might exceed block limit, making it impossible to call `commit` due to not enough gas. Currently there is no limit of the amount of Markets which can be added to callback queue.

## Vulnerability Detail

`KeeperOracle.commit` calls back `update` in all markets which called `request` in the oracle version:
```solidity
for (uint256 i; i < _globalCallbacks[version.timestamp].length(); i++)
    _settle(IMarket(_globalCallbacks[version.timestamp].at(i)), address(0));
...
function _settle(IMarket market, address account) private {
    market.update(account, UFixed6Lib.MAX, UFixed6Lib.MAX, UFixed6Lib.MAX, Fixed6Lib.ZERO, false);
}
```

If any `Market` is paused, its `update` function will revert (notice the `whenNotPaused` modifier):
```solidity
    function update(
        address account,
        UFixed6 newMaker,
        UFixed6 newLong,
        UFixed6 newShort,
        Fixed6 collateral,
        bool protect
    ) external nonReentrant whenNotPaused {
```

This means that if any `Market` is paused, all the other markets will be unable to continue trading since `commit` in their oracle provider will revert. It will also be impossible to successfully switch to a new provider for these markets, because previous oracle provider must still commit its latest request before fully switching to a new oracle provider:
```solidity
function _latestStale(OracleVersion memory currentOracleLatestVersion) private view returns (bool) {
    if (global.current == global.latest) return false;
    if (global.latest == 0) return true;

@@@ if (uint256(oracles[global.latest].timestamp) > oracles[global.latest].provider.latest().timestamp) return false;
    if (uint256(oracles[global.latest].timestamp) >= currentOracleLatestVersion.timestamp) return false;

    return true;
}
```

## Impact

One paused market will stop trading in all the markets which use the same oracle provider (`KeeperOracle`).

## Code Snippet

`KeeperOracle.commit` iterates all requested markets and settles them:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L123-L124

`_settle` calls `update` on the market:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L176-L178

`Market.update` has `whenNotPaused` modifier, making it revert when paused:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L87

## Tool used

Manual Review

## Recommendation

1. Consider catching and ignoring revert, when calling `update` for the market in the `_settle` (wrap in try .. catch).
2. Consider adding a limit of the number of markets which are added to callback queue in each oracle version, or alternatively limit the number of authorized markets to call `request`.