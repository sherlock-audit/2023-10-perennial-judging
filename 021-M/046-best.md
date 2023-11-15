Vast Glossy Tapir

medium

# vault.claimReward() If have a market without reward token, it may cause all markets to be unable to retrieve rewards.

## Summary
In `vault.claimReward()`, it will loop through all `market` of `vault` to execute `claimReward()`, and transfer `rewards` to `factory().owner()`. 
If one of the markets does not have `rewards`, that is, `rewardToken` is not set, `Token18 reward = address(0)`. 
Currently, the loop does not make this judgment `reward != address(0)`, it will also execute `market.claimReward()`, and the entire method will `revert`. 
This leads to other markets with `rewards` also being unable to retrieve `rewards`.

## Vulnerability Detail
The current implementation of `vault.claimReward()` is as follows:
```solidity
    function claimReward() external onlyOwner {
        for (uint256 marketId; marketId < totalMarkets; marketId++) {
            _registrations[marketId].read().market.claimReward();
            _registrations[marketId].read().market.reward().push(factory().owner());
        }
    }
```
We can see that the method loops through all the `market` and executes `market.claimReward()`, and `reward().push()`.

The problem is, not every market has `rewards` tokens. `market.sol`'s `rewards` are not forcibly set in `initialize()`. The market's `makerRewardRate.makerRewardRate` is also allowed to be 0.
```solidity
contract Market is IMarket, Instance, ReentrancyGuard {
    /// @dev The token that incentive rewards are paid in
@>  Token18 public reward;

    function initialize(IMarket.MarketDefinition calldata definition_) external initializer(1) {
        __Instance__initialize();
        __ReentrancyGuard__initialize();

        token = definition_.token;
        oracle = definition_.oracle;
        payoff = definition_.payoff;
    }
...


library MarketParameterStorageLib {
...
    function validate(
        MarketParameter memory self,
        ProtocolParameter memory protocolParameter,
        Token18 reward
    ) public pure {
        if (self.settlementFee.gt(protocolParameter.maxFeeAbsolute)) revert MarketParameterStorageInvalidError();

        if (self.fundingFee.max(self.interestFee).max(self.positionFee).gt(protocolParameter.maxCut))
            revert MarketParameterStorageInvalidError();

        if (self.oracleFee.add(self.riskFee).gt(UFixed6Lib.ONE)) revert MarketParameterStorageInvalidError();

        if (
@>          reward.isZero() &&
@>          (!self.makerRewardRate.isZero() || !self.longRewardRate.isZero() || !self.shortRewardRate.isZero())
        ) revert MarketParameterStorageInvalidError();
```

This means that `market.sol` can be without `rewards token`.

If there is such a market, the current `vault.claimReward()` will `revert`, causing other markets with `rewards` to also be unable to retrieve `rewards`.

## Impact
If the `vault` contains markets without `rewards`, it will cause other markets with `rewards` to also be unable to retrieve `rewards`.
## Code Snippet

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L209-L214

## Tool used

Manual Review

## Recommendation
```diff
    function claimReward() external onlyOwner {
        for (uint256 marketId; marketId < totalMarkets; marketId++) {
+           if (_registrations[marketId].read().market.reward().isZero()) continue;
            _registrations[marketId].read().market.claimReward();
            _registrations[marketId].read().market.reward().push(factory().owner());
        }
    }
```
