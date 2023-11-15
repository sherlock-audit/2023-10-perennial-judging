Bouncy Corduroy Chimpanzee

medium

# Invalid oracle version can cause the `maker` position to exceed `makerLimit`, temporarily or permanently bricking the Market contract

## Summary

When invalid oracle version happens, positions pending at the oracle version are invalidated with the following pending positions increasing or decreasing in size. When this happens, all position limit checks are not applied (and can't be cancelled/modified), but they are still verified for the final positions in _invariant. This means that many checks are bypassed during such event. There is a protection against underflow due to this problem by enforcing the calculated `closable` value to be 0 or higher. However, exactly the same problem can happen with overflow and there is no protection against it. 

## Vulnerability Detail

For example:

- Latest global maker = maker limit = 1000
- Pending global maker = 500 [t=100]
- Pending global maker = 1000 [t=200]

If oracle version at t = 100 is invalid, then pending global maker = 1500 (at t = 200). However, due to this check in _invariant:
```solidity
if (context.currentPosition.global.maker.gt(context.riskParameter.makerLimit))
    revert MarketMakerOverLimitError();
```
all Market updates will revert except update to reduce maker position by 500+, which might not be even possible in 1 update depending on maker distribution between users. For example, if 5 users have maker = 300 (1500 total), then no single user can update to reduce maker by 500.
This will temporarily brick Market (all updates will revert) until coordinator increases maker limit. If the limit is already close to max possible (2^62-1), then the contract will be bricked permanently (all updates will revert regardless of maker limit, because global maker will exceed 2^62-1 in calculations and will revert when trying to store it).

The same issue can also cause the other problems, such as:
- Bypassing the market utilization limit if long/short is increased above maker
- User unexpectedly becomes liquidatable with too high position (for example: position 500 -> pending 0 -> pending 500 - will make current = 1000 if middle oracle version is invalid)

## Impact

If current maker is close to maker limit, and some user(s) reduce their maker then immediately increase back, and the oracle version is invalid, maker will be above the maker limit and the Market will be temporarily bricked until coordinator increases the maker limit. Even though it's temporary, it still bricked for some time and coordinator is forced to increase maker limit, breaking the intended market config. Furthermore, once the maker limit is increased, there is no guarantee that the users will reduce it so that the limit can be reduced back.

Also, for some low-price tokens, the maker limit can be close to max possible value (2^62-1 is about `4*1e18` or `Fixed6(4*1e12)`). If the token price is about $0.00001, this means such maker limit allows `$4*1e7` or $40M. So, if low-value token with $40M maker limit is used, this issue will lead to maker overflow 2^62-1 and bricking the Market permanently, with all users being unable to withdraw their funds, losing everything.

While this situation is not very likely, it's well possible. For example, if the maker is close to limit, any maker reducing the position will have some other user immediately take up the freed up maker space, so things like global maker change of: 1000->900->1000 are easily possible and any invalid oracle version will likely cause the maker overflowing the limit.

## Code Snippet

`_processPositionGlobal` invalidates the pending position if oracle version is invalid:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L486

This is done by changing the `invalidation` accumulated values of the position. When each position is loaded, it's also adjusted by applying the difference between accumulated invalidation of the latest and the loaded position:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L223-L229

Such invalidation change will update the current position ignoring any position limits. If global maker is increased above maker limit, then any `Market.update` call will revert at the following line:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L586-L587

Alternatively, if the new position size (maker, long or short) is above 2^62-1, the `Market.update` will permanently revert at the following lines when trying to store new position value:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/types/Position.sol#L533-L535

## Tool used

Manual Review

## Recommendation

The same issue for underflow is already resolved by using `closable` and enforcing such pending positions that no invalid oracle can cause the position to be less than 0. This issue can be resolved in the same way, by introducing some `opeanable` value (calculated similar to `closable`, but in reverse - when position is increased, it's increased, when position is decreased, it doesn't change) and enforcing different limits, such that `settled position + openable`:
- can not exceed the max maker
- can not break utilization
- for local position - calculate maxMagnitude amount from `settled + local openable` instead of absolute pending position values for margined/maintained calculations.
