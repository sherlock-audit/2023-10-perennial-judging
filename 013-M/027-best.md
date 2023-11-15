Bouncy Corduroy Chimpanzee

medium

# Vault `_maxDeposit` incorrect calculation allows to bypass vault deposit cap

## Summary

Vault has a deposit cap risk setting, which is the max amount of funds users can deposit into the vault. The problem is that `_maxDeposit` function, which calculates max amount of assets allowed to be deposited is incorrect and always includes vault claimable assets even when the vault is at the cap. This allows malicious (or even regular) user to deposit unlimited amount bypassing the vault cap, if the vault has any assets redeemed but not claimed yet. This breaks the core protocol function which limits users risk, for example when the vault is still in the testing phase and owner wants to limit potential losses in case of any problems.

## Vulnerability Detail

`Vault._update` limits the user deposit to `_maxDeposit()` amount:
```solidity
    if (depositAssets.gt(_maxDeposit(context)))
        revert VaultDepositLimitExceededError();
...
function _maxDeposit(Context memory context) private view returns (UFixed6) {
    if (context.latestCheckpoint.unhealthy()) return UFixed6Lib.ZERO;
    UFixed6 collateral = UFixed6Lib.from(totalAssets().max(Fixed6Lib.ZERO)).add(context.global.deposit);
    return context.global.assets.add(context.parameter.cap.sub(collateral.min(context.parameter.cap)));
}
```

When calculating max deposit, the vault's collateral consists of vault assets as well as assets which are redeemed but not yet claimed. However, the formula used to calculate max deposit is incorrect, it is:

`maxDeposit = claimableAssets + (cap - min(collateral, cap))`

As can be seen from the formula, regardless of cap and current collateral, maxDeposit will always be at least claimableAssets, even when the vault is already at the cap or above cap, which is apparently wrong. The correct formula should subtract claimableAssets from collateral (or 0 if claimableAssets is higher than collateral) instead of adding it to the result:

`maxDeposit = cap - min(collateral - min(collateral, claimableAssets), cap)`

Current incorrect formula allows to deposit up to claimable assets amount even when the vault is at or above cap. This can either be used by malicious user (user can deposit up to cap, redeem, deposit amount = up to cap + claimable, redeem, ..., repeat until target deposit amount is reached) or can happen itself when there are claimable assets available and vault is at the cap (which can easily happen by itself if some user forgets to claim or it takes long time to claim).

## Impact

Malicious and regular users can bypass vault deposit cap, either intentionally or just in the normal operation when some users redeem and claimable assets are available in the vault. This breaks core contract security function of limiting the deposit amount and can potentially lead to big user funds loss, for example at the initial stages when the owner still tests the oracle provider/market/etc and wants to limit vault deposit if anything goes wrong, but gets unlimited deposits instead.

## Proof of concept

Bypass of vault cap is demonstrated in the test, add this to `Vault.test.ts`:
```ts
it('bypass vault deposit cap', async () => {
    console.log("start");

    await vault.connect(owner).updateParameter({
    cap: parse6decimal('100'),
    });

    await updateOracle()

    var deposit = parse6decimal('100')
    console.log("Deposit 100")
    await vault.connect(user).update(user.address, deposit, 0, 0)

    await updateOracle()
    await vault.settle(user.address);

    var assets = await vault.totalAssets();
    console.log("Vault assets: " + assets);

    // additional deposit reverts due to cap
    var deposit = parse6decimal('10')
    console.log("Deposit 10 revert")
    await expect(vault.connect(user).update(user.address, deposit, 0, 0)).to.be.reverted;

    // now redeem 50
    var redeem = parse6decimal('50')
    console.log("Redeem 50")
    await vault.connect(user).update(user.address, 0, redeem, 0);

    await updateOracle()
    await vault.settle(user.address);

    var assets = await vault.totalAssets();
    console.log("Vault assets: " + assets);

    // deposit 100 (50+100=150) doesn't revert, because assets = 50
    var deposit = parse6decimal('100')
    console.log("Deposit 100")
    await vault.connect(user).update(user.address, deposit, 0, 0);

    await updateOracle()
    await vault.settle(user.address);

    var assets = await vault.totalAssets();
    console.log("Vault assets: " + assets);

    var deposit = parse6decimal('50')
    console.log("Deposit 50")
    await vault.connect(user).update(user.address, deposit, 0, 0);

    await updateOracle()
    await vault.settle(user.address);

    var assets = await vault.totalAssets();
    console.log("Vault assets: " + assets);
})
```

Console log from execution of the code above:
```solidity
start
Deposit 100
Vault assets: 100000000
Deposit 10 revert
Redeem 50
Vault assets: 50000000
Deposit 100
Vault assets: 150000000
Deposit 50
Vault assets: 200000000
```

The vault cap is set to 100 and is then demonstrated that it is bypassed and vault assets are set at 200 (and can be continued indefinitely)

## Code Snippet

`_maxDeposit` always adds `context.global.assets` (assets redeemed but not yet claimed) to the returned amount, even when the vault is at or above cap:
https://github.com/sherlock-audit/2023-10-perennial/blob/main/perennial-v2/packages/perennial-vault/contracts/Vault.sol#L483

## Tool used

Manual Review

## Recommendation

The correct formula to `_maxDeposit` should be:

`maxDeposit = cap - min(collateral - min(collateral, claimableAssets), cap)`

So the code can be:
```solidity
function _maxDeposit(Context memory context) private view returns (UFixed6) {
    if (context.latestCheckpoint.unhealthy()) return UFixed6Lib.ZERO;
    UFixed6 collateral = UFixed6Lib.from(totalAssets().max(Fixed6Lib.ZERO)).add(context.global.deposit);
    return context.parameter.cap.sub(collateral.sub(context.global.assets.min(collateral)).min(context.parameter.cap));
}
```