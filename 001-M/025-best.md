Bouncy Corduroy Chimpanzee

medium

# `KeeperOracle.request` adds only the first pair of market+account addresses per oracle version to callback list, ignoring all the subsequent ones

## Summary

The new feature introduced in 2.1 is the callback called for all markets and market+account pairs which requested the oracle version. These callbacks are called once the corresponding oracle settles. For this reason, `KeeperOracle` keeps a list of markets and market+account pairs per oracle version to call market.update on them:
```solidity
/// @dev Mapping from version to a set of registered markets for settlement callback
mapping(uint256 => EnumerableSet.AddressSet) private _globalCallbacks;

/// @dev Mapping from version and market to a set of registered accounts for settlement callback
mapping(uint256 => mapping(IMarket => EnumerableSet.AddressSet)) private _localCallbacks;
```

However, currently `KeeperOracle` stores only the market+account from the first request call per oracle version, because if the request was already made, it returns from the function before adding to the list:
```solidity
function request(IMarket market, address account) external onlyAuthorized {
    uint256 currentTimestamp = current();
@@@ if (versions[_global.currentIndex] == currentTimestamp) return;

    versions[++_global.currentIndex] = currentTimestamp;
    emit OracleProviderVersionRequested(currentTimestamp);

    // @audit only the first request per version reaches these lines to add market+account to callback list
    _globalCallbacks[currentTimestamp].add(address(market));
    _localCallbacks[currentTimestamp][market].add(account);
    emit CallbackRequested(SettlementCallback(market, account, currentTimestamp));
}
```

## Vulnerability Detail

According to docs, the same `KeeperOracle` can be used by multiple markets. And every account requesting in the same oracle version is supposed to be called back (settled) once the oracle version settles.

## Impact

The new core function of the protocol doesn't work as expected and `KeeperOracle` will fail to call back markets and accounts if there is more than 1 request in the same oracle version (which is very likely).

## Code Snippet

`KeeperOracle.request` will return early if the request for this oracle version was already made:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L77

The lines to add market+account to callback list will only be reached once per oracle version:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L82-L83

## Tool used

Manual Review

## Recommendation

Move addition to callback list to just before the condition to exit function early:
```solidity
function request(IMarket market, address account) external onlyAuthorized {
    uint256 currentTimestamp = current();
    _globalCallbacks[currentTimestamp].add(address(market));
    _localCallbacks[currentTimestamp][market].add(account);
    emit CallbackRequested(SettlementCallback(market, account, currentTimestamp));
    if (versions[_global.currentIndex] == currentTimestamp) return;

    versions[++_global.currentIndex] = currentTimestamp;
    emit OracleProviderVersionRequested(currentTimestamp);
}
```