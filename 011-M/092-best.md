Festive Bone Cobra

medium

# Withdrawals in the `PrizeVault` will not work for some vaults due to using `previewWithdraw()` and then `redeem()`

## Summary

`PrizeVault::_withdraw()` withdraws assets by calling `yieldVault.previewWithdraw(_assets - _latentAssets);` followed by `yieldVault.redeem(_yieldVaultShares, address(this), address(this));`, which may pull less assets than required, making the transaction revert.

## Vulnerability Detail

In EIP4626, it is stated that `previewWithdraw()` must return at least the amount of shares burned via `withdraw()`
> MUST return as close to and no fewer than the exact amount of Vault shares that would be burned in a withdraw call in the same transaction. I.e. withdraw should return the same or fewer shares as previewWithdraw if called in the same transaction.

`previewRedeem()` must return at most the assets pulled when calling `redeem()`
> MUST return as close to and no more than the exact amount of assets that would be withdrawn in a redeem call in the same transaction. I.e. redeem should return the same or more assets as previewRedeem if called in the same transaction.

However, there is no strict dependency between the 2 and there may be some difference when mixing the 2 withdrawal flows as is done in `PrizeVault::_withdraw()`.
```solidity
function _withdraw(address _receiver, uint256 _assets) internal {
    ...
    if (_assets > _latentAssets) {
        // The latent balance is subtracted from the withdrawal so we don't withdraw more than we need.
        uint256 _yieldVaultShares = yieldVault.previewWithdraw(_assets - _latentAssets);
        // Assets are sent to this contract so any leftover dust can be redeposited later.
        yieldVault.redeem(_yieldVaultShares, address(this), address(this));
    }
    ...
}
```
`previewWithdraw()` is called to get the amount of shares and then `redeem()` is called with this amount, expecting to receive at least `_assets - _latentAssets` to transfer later to the receiver. However, there is no guarantee that at least the required assets will be pulled from the yield vault, which will make the transaction revert when it tries to transfer the assets to the receiver.

Note1: the same issue is found in `PrizeVault::_depositAndMint()`, where the assets pulled in `mint()` may be more than the balance of the contract , which may be fixed with the same recommendation.
Note2: the yield buffer does not help here as the buffer applies to already deposited assets accruing yield, not assets in the `PrizeVault`. As soon as the first deposit is made, all assets in the `PrizeVault` are deposited and a rounding error will make deposits or withdrawals fail.

## Impact

DoSed withdrawals due to `.safeTransfer()` reverting.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L1054

## Tool used

Manual Review

Vscode

## Recommendation

Cap the transfer amount to the available balance.