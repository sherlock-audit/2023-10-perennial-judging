Gorgeous Oily Salmon

medium

# Current `KeeperFactory#settle` logic is not entirely correct: Keepers that input lower maxCounts earn more keeper fees than those that input larger maxCounts

## Summary
With the current `KeeperFactory#settle` logic, either:

- Keepers receive less fees than necessary when they input high maxCounts, or
- Keepers receive more fee than necessary when they input low maxCounts(e.g. 1)

## Vulnerability Detail
When [`settle`ing](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L202) accounts that a Market requested a new oracle version for, the caller is allowed to enter a `maxCount` parameter, which means that `maxCount` number of accounts within the callback array will be [settled](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L141-L146).

`settle`ing an account pays the caller a predetermined amount based on the [settleKeepConfig()](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L204) values. So, irrespective of the number of accounts that gets settled in a single `KeeperFactory#settle` call, the caller receives the same amount.

For example, if Alice calls `KeeperFactory#settle` with a maxCount of 10, and Bob calls the function with a maxCount of 1, they both get paid the same amount of keeper fees.

This incentivizes a keeper to call `KeeperFactory#settle` with a maxCount of 1, n times,(rather than inputting a maxCount of n) so that they get more keeper fees. This will lead to significant loss of keeper fees overtime as many users interact with a Market.

Consider the following scenario:

- Let's say keeper fee to be paid when keeper calls `KeeperFactory#settle`(based on settleKeepConfig), is 1 DSU
- 200 unique users update their account within a granularity time period/oracleVersion.
- price for that oracleVersion gets committed
- By calling `KeeperFactory#settle` with a maxCount of 200, keeper would have received 1 DSU as fee
- Keeper uses a contract to call `KeeperFactory#settle` with a maxCount of 1, 200 times in a loop, which will make him receive 200\*1=200 DSU as fees

## Impact
User can enter small maxCount for a market multiple times to use up more keeper fees than keepers that input larger maxCount.
This leads to wastage of keeper fees

## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L202-L217

## Tool used

Manual Review

## Recommendation
Consider implementing any of these:

1. Keepers should be rewarded according to the number of accounts they settled, to make it fair for all keepers.

2. There should be a minimum number of accounts that a keeper is allowed to settle. This will reduce wastage of the fees.
   Within KeeperOracle#settle:

```solidity
contract KeeperOracle{
++  uint256 public minSettleableAccounts;
    ...
++  function updateMinSettleableAccounts(uint256 count)external onlyOwner{
++      minSettleableAccounts=count;
++  }
    ...
    function settle(IMarket market, uint256 version, uint256 maxCount) external onlyFactory {
        EnumerableSet.AddressSet storage callbacks = _localCallbacks[version][market];

        if (_global.latestVersion < version) revert KeeperOracleVersionOutsideRangeError();
--      if (maxCount == 0) revert KeeperOracleInvalidCallbackError();
++      if (maxCount<minimum(minSettleableAccounts,callbacks.length())) revert KeeperOracleInvalidCallbackError();
        if (callbacks.length() == 0) revert KeeperOracleInvalidCallbackError();

        for (uint256 i; i < maxCount && callbacks.length() > 0; i++) {
            address account = callbacks.at(0);
            _settle(market, account);
            callbacks.remove(account);
            emit CallbackFulfilled(SettlementCallback(market, account, version));
        }
    }

}

```
