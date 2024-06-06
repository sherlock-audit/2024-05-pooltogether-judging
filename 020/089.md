Festive Bone Cobra

medium

# `PrizeVault::depositWithPermit()` still allows griefing attacks, despite the attempts of the code and comments

## Summary

`PrizeVault::depositWithPermit()` only calls `asset.permit()` if the allowance is not the same as the assets to deposit, which will revert if the allowance was non null prior the permit call.

## Vulnerability Detail

Using permits when depositing is generally vulnerable to DoS by frontrunning and using the signature standalone in the asset contract before it is used in the intended function. The code is aware of this issue and implements some measures to mitigate it in `PrizeVault::depositWithPermit()`,
```solidity
// Skip the permit call if the allowance has already been set to exactly what is needed. This prevents
// griefing attacks where the signature is used by another actor to complete the permit before this
// function is executed.
if (_asset.allowance(_owner, address(this)) != _assets) { //@audit DoS if allowance was non 0 and became bigger with frontrun
    IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
}
```
However, this mitigation is incomplete as it will still revert and be DoSed if the `owner` had given the protocol allowance before the permit call. Consider the following scenario, in which a deposit with permit call will be DoSed.
```solidity
asset.allowance(owner, address(this)) = 1
// owner calls depositWithPermit(), which increases the allowance exactly by assets
// attacker frontruns the owner and calls permit on the asset directly, increasing the allowance to 
asset.allowance(owner, address(this)) = 1 + assets
// the code calls permit on the asset as asset.allowance(owner, address(this))  != assets and it reverts due to reusing the same signature
```

## Impact

DoSed `PrizeVault::depositWithPermit()` despite the attempted mitigations.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L545-L547

## Tool used

Manual Review

Vscode

## Recommendation

Permit should only be called if the allowance is smaller than `assets`,
```solidity
if (_asset.allowance(_owner, address(this)) < _assets) { // only call permit if the allowance is smaller
    IERC20Permit(address(_asset)).permit(_owner, address(this), _assets, _deadline, _v, _r, _s);
}
```