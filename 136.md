Real Ceramic Koala

medium

# `maxRedeem` doesn't comply with ERC-4626

## Summary
`maxRedeem` function reverts due to division by 0 and hence doesn't comply with ERC4626 

## Vulnerability Detail
The contract's `maxRedeem` function doesn't comply with ERC-4626 which is a mentioned requirement. 
According to the [specification](https://eips.ethereum.org/EIPS/eip-4626#maxredeem), `MUST NOT revert.`  
The `maxRedeem` function will revert in case the totalAsset amount is 0 due to a divison by 0

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L445)
```solidity
uint256 _maxScaledRedeem = _convertToShares(_maxWithdraw, _totalAssets, totalDebt(), Math.Rounding.Up);
```

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L864-L876)
```solidity
    function _convertToShares(
        uint256 _assets,
        uint256 _totalAssets,
        uint256 _totalDebt,
        Math.Rounding _rounding
    ) internal pure returns (uint256) {
        if (_totalAssets >= _totalDebt) {
            return _assets;
        } else {
            return _assets.mulDiv(_totalDebt, _totalAssets, _rounding);
```
### POC

```diff
diff --git a/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol b/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
index 7006f44..9b92c35 100644
--- a/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
+++ b/pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol
@@ -4,6 +4,8 @@ pragma solidity ^0.8.24;
 import { UnitBaseSetup, PrizePool, TwabController, ERC20, IERC20, IERC4626, YieldVault } from "./UnitBaseSetup.t.sol";
 import { IPrizeHooks, PrizeHooks } from "../../../src/interfaces/IPrizeHooks.sol";
 import { ERC20BrokenDecimalMock } from "../../contracts/mock/ERC20BrokenDecimalMock.sol";
+import "forge-std/Test.sol";
+
 
 import "../../../src/PrizeVault.sol";
 
@@ -45,6 +47,34 @@ contract PrizeVaultTest is UnitBaseSetup {
         assertEq(testVault.owner(), address(this));
     }
 
+    function testHash_MaxRedeemRevertDueToDivByZero() public {
+        PrizeVault testVault=  new PrizeVault(
+            "PoolTogether aEthDAI Prize Token (PTaEthDAI)",
+            "PTaEthDAI",
+            yieldVault,
+            PrizePool(address(prizePool)),
+            claimer,
+            address(this),
+            YIELD_FEE_PERCENTAGE,
+            0,
+            address(this)
+        );
+
+        uint amount_ = 1e18;
+        underlyingAsset.mint(address(this),amount_);
+
+        underlyingAsset.approve(address(testVault),amount_);
+
+        testVault.deposit(amount_,address(this));
+
+        // lost the entire deposit amount
+        underlyingAsset.burn(address(yieldVault), 1e18);
+
+        vm.expectRevert();
+        testVault.maxRedeem(address(this));
+
+    }
+
     function testConstructorYieldVaultZero() external {
         vm.expectRevert(abi.encodeWithSelector(PrizeVault.YieldVaultZeroAddress.selector));
 

```

## Impact
Failure to comply with the specification which is a mentioned necessity

## Code Snippet

## Tool used
Manual Review

## Recommendation
Handle the `_totalAssets == 0` condition