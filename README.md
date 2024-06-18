# Issue H-1: after the TWAB limit is reached, back-running a call to `withdraw` with a liquidation of yield is extremely profitable for the liquidator at the expense of all the other users in the vault 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/19 

The protocol has acknowledged this issue.

## Found by 
0xSpearmint1
## Summary
after the TWAB limit is reached, back-running a call to `withdraw` with a liquidation of yield is extremely profitable for the liquidator at the expense of all the other users in the vault

## Vulnerability Detail
The price of liquidating yield is inversely proportional to the `elapsedTime` 

```solidity
function _computePrice() internal view returns (uint192) {
        uint256 elapsedTime = block.timestamp - lastAuctionAt;
        if (elapsedTime == 0) {
            return type(uint192).max;
        }
        uint192 price = uint192((targetAuctionPeriod * lastAuctionPrice) / elapsedTime);

        if (price < MIN_PRICE) {
            price = MIN_PRICE;
        }

        return price;
    }
```

Once the `TWAB_SUPPLY_LIMIT` is reached, all liquidation attempts will revert due to the `_enforceMintLimit()` in `transferTokensOut()`

A liquidator can take advantage of this by the following steps

1. `TWAB_SUPPLY_LIMIT` is reached for a vault, at this point any future liquidations will revert, but `elapsedTime` keeps accruing

2. Once price = MIN_PRICE, due to a large `elapsedTime`, the liquidator will withdraw some assets from the vault, this will burn some shares and reduce the totalSupply

3. The liquidator will immediately backrun the `withdraw` call with a liquidation at price = MIN_PRICE for a huge profit at the expense of the other vault users

The result will be the attacker exchanging very few prizeTokens for all the yield.


A key point to note is that if some other user withdraws before the price = MIN_PRICE, the liquidator can backrun that users Tx with a deposit to reach the `TWAB_SUPPLY_LIMIT` and prevent other liquidations.

TWAB_SUPPLY_LIMIT = type(uint96).max 

This is easily achievable for ERC20's with a large totalSupply and low price for example SHIBE or LADYS

The protocol stated in the README
>The protocol supports any standard ERC20 that does not have reentrancy or fee on transfer.

Therefore it should support tokens like SHIBE and LADYS ( they fit the criteria ) but the PrizeVaults do not

## Impact
The attacker has effectively stolen the yield from other users. 

The attacker has reduced the chances of winning for all other users of that vault.

## Proof of concept
Replace the existing `BaseIntegration.t.sol` in /2024-05-pooltogether/pt-v5-vault/test/integration/BaseIntegration.t.sol with the following 
<details>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Test } from "forge-std/Test.sol";
import { console2 } from "forge-std/console2.sol";

import { IERC20, IERC4626 } from "openzeppelin/token/ERC20/extensions/ERC4626.sol";

import { PrizePool } from "pt-v5-prize-pool/PrizePool.sol";
import { TwabController } from "pt-v5-twab-controller/TwabController.sol";

import { ERC20PermitMock } from "../contracts/mock/ERC20PermitMock.sol";
import { PrizePoolMock } from "../contracts/mock/PrizePoolMock.sol";
import { Permit } from "../contracts/utility/Permit.sol";

import { PrizeVaultWrapper, PrizeVault } from "../contracts/wrapper/PrizeVaultWrapper.sol";

import { YieldVault } from "../contracts/mock/YieldVault.sol";

abstract contract BaseIntegration is Test, Permit {

    /* ============ events ============ */

    event Deposit(address indexed sender, address indexed owner, uint256 assets, uint256 shares);
    event Transfer(address indexed from, address indexed to, uint256 value);
    event ClaimerSet(address indexed claimer);
    event LiquidationPairSet(address indexed tokenOut, address indexed liquidationPair);
    event YieldFeeRecipientSet(address indexed yieldFeeRecipient);
    event YieldFeePercentageSet(uint256 yieldFeePercentage);
    event MockContribute(address prizeVault, uint256 amount);
    event ClaimYieldFeeShares(address indexed recipient, uint256 shares);
    event TransferYieldOut(address indexed liquidationPair, address indexed tokenOut, address indexed recipient, uint256 amountOut, uint256 yieldFee);
    event Sponsor(address indexed caller, uint256 assets, uint256 shares);

    /* ============ variables ============ */

    address public vitalik = 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045;
    address public attacker = 0xEf57e88a27f8958D72fd9F983D28349374e6C110;

    address internal owner;
    uint256 internal ownerPrivateKey;

    address internal alice;
    uint256 internal alicePrivateKey;

    address internal bob;
    uint256 internal bobPrivateKey;

    PrizeVaultWrapper public prizeVault;
    string public vaultName = "PoolTogether Test Vault";
    string public vaultSymbol = "pTest";
    uint256 public yieldBuffer = 1e5;

    IERC4626 public yieldVault;
    ERC20PermitMock public underlyingAsset;
    uint8 public assetDecimals;
    uint256 public approxAssetUsdExchangeRate;
    ERC20PermitMock public prizeToken;

    address public claimer;
    PrizePoolMock public prizePool;

    uint32 public drawPeriodSeconds = 1 days;
    TwabController public twabController;

    /// @dev A low gas estimate of what it would cost in gas to manipulate the state into losing 1 wei of assets
    /// in a rounding error. (This should be a unachievable low estimate such that an attacker would never be able
    /// to cause repeated rounding errors for less).
    uint256 lowGasEstimateForStateChange = 100_000;

    /// @dev A low gas price estimate for the chain.
    uint256 lowGasPriceEstimate = 7 gwei;

    /// @dev Defined in 1e18 precision. (2e18 would be 2 ETH per USD)
    uint256 approxEthUsdExchangeRate = uint256(1e18) / uint256(3000); // approx 1 ETH for $3000

    /// @dev Override to true if vault is not expected to have yield
    bool ignoreYield = false;

    /// @dev Override to true if vault cannot feasibly lose assets
    bool ignoreLoss = false;

    /// @dev Some yield vaults have predictable precision loss (such as Sonne which reduces precision of assets with their exchange rate).
    /// This optional variable can be set if this precision loss is acknowledged and appropriate countermeasures will be taken (ex. deploying
    /// a PrizeVault with a higher yield buffer to match the reduced precision).
    uint8 assetPrecisionLoss = 0;

    /* ============ setup ============ */

    function setUpUnderlyingAsset() public virtual returns (IERC20 asset, uint8 decimals, uint256 approxAssetUsdExchangeRate);

    function setUpYieldVault() public virtual returns (IERC4626) {
        return new YieldVault(
            address(underlyingAsset),
            "Test Yield Vault",
            "yvTest"
        );
    }

    function setUpFork() public virtual;

    function beforeSetup() public virtual;

    function afterSetup() public virtual;

    function setUp() public virtual {
        beforeSetup();

        (owner, ownerPrivateKey) = makeAddrAndKey("Owner");
        (alice, alicePrivateKey) = makeAddrAndKey("Alice");
        (bob, bobPrivateKey) = makeAddrAndKey("Bob");

        underlyingAsset = new ERC20PermitMock("SHIBE");

        yieldVault = setUpYieldVault();

        

        prizeToken = new ERC20PermitMock("POOL");

        twabController = new TwabController(1 hours, uint32(block.timestamp));

        prizePool = new PrizePoolMock(prizeToken, twabController);

        claimer = address(0xe291d9169F0316272482dD82bF297BB0a11D267f);

        // adjust yield buffer based-on mitigated precision loss
        yieldBuffer = 1e5 * (10 ** assetPrecisionLoss);

        prizeVault = new PrizeVaultWrapper(
            vaultName,
            vaultSymbol,
            yieldVault,
            PrizePool(address(prizePool)),
            claimer,
            bob,
            9e8,
            yieldBuffer, // yield buffer
            address(this)
        );
        prizeVault.setLiquidationPair(address(this));

        // Fill yield buffer if non-zero:
        if (yieldBuffer > 0) {
            dealAssets(address(prizeVault), yieldBuffer);
        }

        afterSetup();
    }

    /* ============ prank helpers ============ */

    address private currentPrankee;

    /// @notice should be used instead of vm.startPrank so we can keep track of pranks within other pranks
    function startPrank(address prankee) public {
        currentPrankee = prankee;
        vm.startPrank(prankee);
    }

    /// @notice should be used instead of vm.stopPrank so we can keep track of pranks within other pranks
    function stopPrank() public {
        currentPrankee = address(0);
        vm.stopPrank();
    }

    /// @notice pranks the address and sets the prank back to the msg sender at the end
    modifier prankception(address prankee) {
        address prankBefore = currentPrankee;
        vm.stopPrank();
        vm.startPrank(prankee);
        _;
        vm.stopPrank();
        if (prankBefore != address(0)) {
            vm.startPrank(prankBefore);
        }
    }

    /* ============ helpers to override ============ */

    /// @dev The max amount of assets than can be dealt.
    function maxDeal() public virtual returns (uint256);

    /// @dev May revert if the amount requested exceeds the amount available to deal.
    function dealAssets(address to, uint256 amount) public virtual;

    /// @dev Some yield sources accrue by time, so it's difficult to accrue an exact amount. Call the 
    /// function multiple times if the test requires the accrual of more yield than the default amount.
    function _accrueYield() internal virtual;

    /// @dev Simulates loss on the yield vault such that the value of it's shares drops
    function _simulateLoss() internal virtual;

    /// @dev Each integration test must override the `_accrueYield` internal function for this to work.
    function accrueYield() public returns (uint256) {
        uint256 assetsBefore = prizeVault.totalPreciseAssets();
        _accrueYield();
        uint256 assetsAfter = prizeVault.totalPreciseAssets();
        if (yieldVault.balanceOf(address(prizeVault)) > 0) {
            // if the prize vault has any yield vault shares, check to ensure yield has accrued
            require(assetsAfter > assetsBefore, "yield did not accrue");
        } else {
            if (underlyingAsset.balanceOf(address(prizeVault)) > 0) {
                // the underlying asset might be rebasing in some setups, so it's possible time passing has caused an increase in latent balance
                require(assetsAfter >= assetsBefore, "assets decreased while holding latent balance");
            } else {
                // otherwise, we should expect no change on the prize vault
                require(assetsAfter == assetsBefore, "assets changed with zero yield shares");
            }
        }
        return assetsAfter - assetsBefore;
    }

    function checkIgnoreYield() internal returns (bool) {
        if (ignoreYield) {
            console2.log("skipping test since yield is ignored...");
            return true;
        }
        return false;
    }

    /// @dev Each integration test must override the `_simulateLoss` internal function for this to work.
    /// @return The loss the prize vault has incurred as a result of yield vault loss (if any)
    function simulateLoss() public returns (uint256) {
        uint256 assetsBefore = prizeVault.totalPreciseAssets();
        _simulateLoss();
        uint256 assetsAfter = prizeVault.totalPreciseAssets();
        if (yieldVault.balanceOf(address(prizeVault)) > 0) {
            // if the prize vault has any yield vault shares, check to ensure some loss has occurred
            require(assetsAfter < assetsBefore, "loss not simulated");
        } else {
            if (underlyingAsset.balanceOf(address(prizeVault)) > 0) {
                // the underlying asset might be rebasing in some setups, so it's possible time passing has caused an increase in latent balance
                require(assetsAfter >= assetsBefore, "assets decreased while holding latent balance");
                return 0;
            } else {
                // otherwise, we should expect no change on the prize vault
                require(assetsAfter == assetsBefore, "assets changed with zero yield shares");
            }
        }
        return assetsBefore - assetsAfter;
    }

    function checkIgnoreLoss() internal returns (bool) {
        if (ignoreLoss) {
            console2.log("skipping test since loss is ignored...");
            return true;
        }
        return false;
    }

    /* ============ Integration Test Scenarios ============ */

    //////////////////////////////////////////////////////////
    /// Basic Asset Tests
    //////////////////////////////////////////////////////////

    /// @dev Tests if the asset meets a minimum precision per dollar (PPD). If the asset
    /// is below this PPD, then it is possible that the yield buffer will not be able to sustain
    /// the rounding errors that will accrue on deposits and withdrawals.
    ///
    /// Also passes if the yield buffer is zero (no buffer is needed if rounding errors are impossible).
    function testAssetPrecisionMeetsMinimum() public {
        if (yieldBuffer > 0) {
            uint256 minimumPPD = 1e6; // USDC is the benchmark (6 decimals represent $1 of value)
            uint256 assetPPD = (1e18 * (10 ** (assetDecimals - assetPrecisionLoss))) / approxAssetUsdExchangeRate;
            assertGe(assetPPD, minimumPPD, "asset PPD > minimum PPD");
        }
    }

    /// @notice Test if the attacker can cause rounding error loss on the prize vault by spending
    /// less in gas than the rounding error loss will cost the vault.
    ///
    /// @dev This is an important measure to test since it's possible that some yield vaults can essentially
    /// rounding errors to other holders of yield vault shares. If an attacker can cheaply cause repeated
    /// rounding errors on the prize vault, they can potentially profit by being a majority shareholder on
    /// the underlying yield vault.
    function testAssetRoundingErrorManipulationCost() public {
        uint256 costToManipulateInEth = lowGasPriceEstimate * lowGasEstimateForStateChange;
        uint256 costToManipulateInUsd = (costToManipulateInEth * 1e18) / approxEthUsdExchangeRate;
        uint256 costOfRoundingErrorInUsd = ((10 ** assetPrecisionLoss) * 1e18) / approxAssetUsdExchangeRate;
        // 10x threshold is set so an attacker would have to spend at least 10x the loss they can cause on the prize vault.
        uint256 multiplierThreshold = 10;
        assertLt(costOfRoundingErrorInUsd * multiplierThreshold, costToManipulateInUsd, "attacker can cheaply cause rounding errors");
    }

    //////////////////////////////////////////////////////////
    /// Deposit Tests
    //////////////////////////////////////////////////////////

    /// @notice test deposit
    function testDeposit() public {
        uint256 amount = 10 ** assetDecimals;
        uint256 maxAmount = maxDeal() / 2;
        if (amount > maxAmount) amount = maxAmount;
        dealAssets(alice, amount);

        uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
        uint256 totalSupplyBefore = prizeVault.totalSupply();

        startPrank(alice);
        underlyingAsset.approve(address(prizeVault), amount);
        prizeVault.deposit(amount, alice);
        stopPrank();

        uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
        uint256 totalSupplyAfter = prizeVault.totalSupply();

        assertEq(prizeVault.balanceOf(alice), amount, "shares minted");
        assertApproxEqAbs(
            totalAssetsBefore + amount,
            totalAssetsAfter,
            10 ** assetPrecisionLoss,
            "assets accounted for with possible rounding error"
        );
        assertEq(totalSupplyBefore + amount, totalSupplyAfter, "supply increased by amount");
    }

    /// @notice test multi-user deposit
    function testMultiDeposit() public {
        address[] memory depositors = new address[](3);
        depositors[0] = alice;
        depositors[1] = bob;
        depositors[2] = address(this);

        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 amount = (10 ** assetDecimals) * (i + 1);
            uint256 maxAmount = maxDeal() / 2;
            if (amount > maxAmount) amount = maxAmount;
            dealAssets(depositors[i], amount);

            uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
            uint256 totalSupplyBefore = prizeVault.totalSupply();

            startPrank(depositors[i]);
            underlyingAsset.approve(address(prizeVault), amount);
            prizeVault.deposit(amount, depositors[i]);
            stopPrank();

            uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
            uint256 totalSupplyAfter = prizeVault.totalSupply();

            assertEq(prizeVault.balanceOf(depositors[i]), amount, "shares minted");
            assertApproxEqAbs(
                totalAssetsBefore + amount,
                totalAssetsAfter,
                10 ** assetPrecisionLoss,
                "assets accounted for with possible rounding error"
            );
            assertEq(totalSupplyBefore + amount, totalSupplyAfter, "supply increased by amount");
        }
    }

    /// @notice test multi-user deposit w/yield accrual in between
    function testMultiDepositWithYieldAccrual() public {
        if (checkIgnoreYield()) return;

        address[] memory depositors = new address[](2);
        depositors[0] = alice;
        depositors[1] = bob;

        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 amount = (10 ** assetDecimals) * (i + 1);
            uint256 maxAmount = maxDeal() / 2;
            if (amount > maxAmount) amount = maxAmount;
            dealAssets(depositors[i], amount);

            uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
            uint256 totalSupplyBefore = prizeVault.totalSupply();

            startPrank(depositors[i]);
            underlyingAsset.approve(address(prizeVault), amount);
            prizeVault.deposit(amount, depositors[i]);
            stopPrank();

            uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
            uint256 totalSupplyAfter = prizeVault.totalSupply();

            assertEq(prizeVault.balanceOf(depositors[i]), amount, "shares minted");
            assertApproxEqAbs(
                totalAssetsBefore + amount,
                totalAssetsAfter,
                10 ** assetPrecisionLoss,
                "assets accounted for with possible rounding error"
            );
            assertEq(totalSupplyBefore + amount, totalSupplyAfter, "supply increased by amount");

            if (i == 0) {
                // accrue yield
                accrueYield();
            }
        }
    }

    //////////////////////////////////////////////////////////
    /// Withdrawal Tests
    //////////////////////////////////////////////////////////

    /// @notice test withdraw
    /// @dev also tests for any active deposit / withdraw fees
    function testWithdraw() public {
        uint256 amount = 10 ** assetDecimals;
        uint256 maxAmount = maxDeal() / 2;
        if (amount > maxAmount) amount = maxAmount;
        dealAssets(alice, amount);

        uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
        uint256 totalSupplyBefore = prizeVault.totalSupply();

        startPrank(alice);
        underlyingAsset.approve(address(prizeVault), amount);
        prizeVault.deposit(amount, alice);
        prizeVault.withdraw(amount, alice, alice);
        stopPrank();

        uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
        uint256 totalSupplyAfter = prizeVault.totalSupply();

        assertEq(prizeVault.balanceOf(alice), 0, "burns all user shares on full withdraw");
        assertEq(underlyingAsset.balanceOf(alice), amount, "withdraws full amount of assets");
        assertApproxEqAbs(
            totalAssetsBefore,
            totalAssetsAfter,
            2 * 10 ** assetPrecisionLoss,
            "no assets missing except for possible rounding error"
        ); // 1 possible rounding error for deposit, 1 for withdraw
        assertEq(totalSupplyBefore, totalSupplyAfter, "supply same as before");
    }

    /// @notice test withdraw with yield accrual
    function testWithdrawWithYieldAccrual() public {
        if (checkIgnoreYield()) return;

        uint256 amount = 10 ** assetDecimals;
        uint256 maxAmount = maxDeal() / 2;
        if (amount > maxAmount) amount = maxAmount;
        dealAssets(alice, amount);

        uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
        uint256 totalSupplyBefore = prizeVault.totalSupply();

        startPrank(alice);
        underlyingAsset.approve(address(prizeVault), amount);
        prizeVault.deposit(amount, alice);
        uint256 yield = accrueYield(); // accrue yield in between
        prizeVault.withdraw(amount, alice, alice);
        stopPrank();

        uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
        uint256 totalSupplyAfter = prizeVault.totalSupply();

        assertEq(prizeVault.balanceOf(alice), 0, "burns all user shares on full withdraw");
        assertEq(underlyingAsset.balanceOf(alice), amount, "withdraws full amount of assets");
        assertApproxEqAbs(
            totalAssetsBefore + yield,
            totalAssetsAfter,
            2 * 10 ** assetPrecisionLoss,
            "no assets missing except for possible rounding error"
        ); // 1 possible rounding error for deposit, 1 for withdraw
        assertEq(totalSupplyBefore, totalSupplyAfter, "supply same as before");
    }

    /// @notice test all users withdraw
    function testWithdrawAllUsers() public {
        address[] memory depositors = new address[](3);
        depositors[0] = alice;
        depositors[1] = bob;
        depositors[2] = address(this);

        uint256[] memory amounts = new uint256[](3);

        // deposit
        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 amount = (10 ** assetDecimals) * (i + 1);
            uint256 maxAmount = maxDeal() / 2;
            if (amount > maxAmount) amount = maxAmount;
            amounts[i] = amount;
            dealAssets(depositors[i], amount);

            startPrank(depositors[i]);
            underlyingAsset.approve(address(prizeVault), amount);
            prizeVault.deposit(amount, depositors[i]);
            stopPrank();
        }

        // withdraw
        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
            uint256 totalSupplyBefore = prizeVault.totalSupply();

            startPrank(depositors[i]);
            prizeVault.withdraw(amounts[i], depositors[i], depositors[i]);
            stopPrank();

            uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
            uint256 totalSupplyAfter = prizeVault.totalSupply();

            assertEq(prizeVault.balanceOf(depositors[i]), 0, "burned all user's shares on withdraw");
            assertEq(underlyingAsset.balanceOf(depositors[i]), amounts[i], "withdrew full asset amount for user");
            assertApproxEqAbs(
                totalAssetsBefore,
                totalAssetsAfter + amounts[i],
                10 ** assetPrecisionLoss,
                "assets accounted for with no more than 1 wei rounding error"
            );
            assertEq(totalSupplyBefore - amounts[i], totalSupplyAfter, "total supply decreased by amount");
        }
    }

    /// @notice test all users withdraw during lossy state
    function testWithdrawAllUsersWhileLossy() public {
        if (checkIgnoreLoss()) return;

        address[] memory depositors = new address[](3);
        depositors[0] = alice;
        depositors[1] = bob;
        depositors[2] = address(this);

        // deposit
        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 amount = (10 ** assetDecimals) * (i + 1);
            uint256 maxAmount = maxDeal() / 2;
            if (amount > maxAmount) amount = maxAmount;
            dealAssets(depositors[i], amount);

            startPrank(depositors[i]);
            underlyingAsset.approve(address(prizeVault), amount);
            prizeVault.deposit(amount, depositors[i]);
            stopPrank();
        }

        // cause loss on the yield vault
        simulateLoss();

        // ensure prize vault is in lossy state
        assertLt(prizeVault.totalPreciseAssets(), prizeVault.totalDebt());

        // verify all users can withdraw a proportional amount of assets
        for (uint256 i = 0; i < depositors.length; i++) {
            uint256 shares = prizeVault.balanceOf(depositors[i]);
            uint256 totalAssetsBefore = prizeVault.totalPreciseAssets();
            uint256 totalSupplyBefore = prizeVault.totalSupply();
            uint256 totalDebtBefore = prizeVault.totalDebt();
            uint256 expectedAssets = (shares * totalAssetsBefore) / totalDebtBefore;

            startPrank(depositors[i]);
            uint256 assets = prizeVault.redeem(shares, depositors[i], depositors[i]);
            stopPrank();

            uint256 totalAssetsAfter = prizeVault.totalPreciseAssets();
            uint256 totalSupplyAfter = prizeVault.totalSupply();

            assertEq(assets, expectedAssets, "assets received proportional to shares / totalDebt");
            assertEq(prizeVault.balanceOf(depositors[i]), 0, "burned all user's shares on withdraw");
            assertEq(underlyingAsset.balanceOf(depositors[i]), assets, "withdrew assets for user");
            assertApproxEqAbs(
                totalAssetsBefore,
                totalAssetsAfter + assets,
                10 ** assetPrecisionLoss,
                "assets accounted for with no more than 1 wei rounding error"
            );
            assertEq(totalSupplyBefore - shares, totalSupplyAfter, "total supply decreased by shares");
        }
    }

    /// @notice test liquidation of assets
    function testAssetLiquidation() public {
        if (checkIgnoreYield()) return;

        // Deposit
        uint256 amount = 1000 * (10 ** assetDecimals);
        uint256 maxAmount = maxDeal() / 2;
        if (amount > maxAmount) amount = maxAmount;
        dealAssets(alice, amount);

        startPrank(alice);
        underlyingAsset.approve(address(prizeVault), amount);
        prizeVault.deposit(amount, alice);
        stopPrank();

        // Yield Accrual
        uint256 approxYield = accrueYield();
        uint256 availableYield = prizeVault.availableYieldBalance();
        assertApproxEqAbs(
            approxYield,
            availableYield,
            10 ** assetPrecisionLoss,
            "approx yield equals available yield balance"
        );
        uint256 availableAssets = prizeVault.liquidatableBalanceOf(address(underlyingAsset));
        assertGt(availableAssets, 0, "assets are available for liquidation");
        
        // Liquidate
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), availableAssets);
        assertEq(underlyingAsset.balanceOf(bob), availableAssets, "liquidator is transferred expected assets");
        assertApproxEqAbs(prizeVault.availableYieldBalance(), availableYield - availableAssets, 10 ** assetPrecisionLoss, "available yield decreased (w / 1 wei rounding error)");
        assertApproxEqAbs(prizeVault.liquidatableBalanceOf(address(underlyingAsset)), 0, 10 ** assetPrecisionLoss, "no more assets can be liquidated (w/ 1 wei rounding error)");
    }

    /// @notice test liquidatable balance of when lossy
    function testNoLiquidationWhenLossy() public {
        if (checkIgnoreYield() || checkIgnoreLoss()) return;

        // Deposit
        uint256 amount = 1000 * (10 ** assetDecimals);
        uint256 maxAmount = maxDeal() / 2;
        if (amount > maxAmount) amount = maxAmount;
        dealAssets(alice, amount);

        startPrank(alice);
        underlyingAsset.approve(address(prizeVault), amount);
        prizeVault.deposit(amount, alice);
        stopPrank();

        // Yield Accrual
        uint256 approxYield = accrueYield();
        uint256 availableYield = prizeVault.availableYieldBalance();
        assertApproxEqAbs(
            approxYield,
            availableYield,
            10 ** assetPrecisionLoss,
            "approx yield equals available yield balance"
        );
        uint256 availableAssets = prizeVault.liquidatableBalanceOf(address(underlyingAsset));
        assertGt(availableAssets, 0, "assets are available for liquidation");
        uint256 availableShares = prizeVault.liquidatableBalanceOf(address(prizeVault));
        assertGt(availableShares, 0, "shares are available for liquidation");

        // Loss Occurs
        simulateLoss();

        // No liquidations can occur if assets < debt
        availableYield = prizeVault.availableYieldBalance();
        assertEq(availableYield, 0, "no available yield after loss");
        availableAssets = prizeVault.liquidatableBalanceOf(address(underlyingAsset));
        assertEq(availableAssets, 0, "no available asset liquidation after loss");
        availableShares = prizeVault.liquidatableBalanceOf(address(prizeVault));
        assertEq(availableShares, 0, "no available share liquidation after loss");
    }
    
}

```
</details>

Now add the following `BackrunWithdrawWithHugelyProfitableLiquidationPOC.sol` to /2024-05-pooltogether/pt-v5-vault/test/integration/BackrunWithdrawWithHugelyProfitableLiquidationPOC.sol

The following foundry test test__BackrunWithdrawWithHugelyProfitableLiquidationPOC() illustrates the attack scenario.

Run it with the following command line input

```terminal
forge test --mt test__BackrunWithdrawWithHugelyProfitableLiquidationPOC -vv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";
import { BaseIntegration } from "./BaseIntegration.t.sol";



import { IERC20, IERC4626 } from "openzeppelin/token/ERC20/extensions/ERC4626.sol";

import { PrizePool } from "pt-v5-prize-pool/PrizePool.sol";
import { TwabController } from "pt-v5-twab-controller/TwabController.sol";

import { ERC20PermitMock } from "../contracts/mock/ERC20PermitMock.sol";
import { PrizePoolMock } from "../contracts/mock/PrizePoolMock.sol";
import { Permit } from "../contracts/utility/Permit.sol";

import { PrizeVaultWrapper, PrizeVault } from "../contracts/wrapper/PrizeVaultWrapper.sol";

import { YieldVault } from "../contracts/mock/YieldVault.sol";

contract BackrunWithdrawWithHugelyProfitableLiquidationPOC is Test, BaseIntegration {
    function setUp() override public {
        super.setUp();


        // Setup users
        underlyingAsset.mint(alice, type(uint128).max);
        vm.startPrank(alice);
        underlyingAsset.approve(address(prizeVault), type(uint128).max);

        underlyingAsset.mint(bob, type(uint128).max);
        vm.startPrank(bob);
        underlyingAsset.approve(address(prizeVault), type(uint128).max);

        underlyingAsset.mint(attacker, type(uint128).max);
        vm.startPrank(attacker);
        underlyingAsset.approve(address(prizeVault), type(uint128).max);
    }

    function setUpUnderlyingAsset() public override returns (IERC20 asset, uint8 decimals, uint256 approxAssetUsdExchangeRate) {}

    function setUpFork() public override {}

    function beforeSetup() public override {}

    function afterSetup() public override {}

    /* ============ helpers to override ============ */

    /// @dev The max amount of assets than can be dealt.
    function maxDeal() public override returns (uint256) {}

    /// @dev May revert if the amount requested exceeds the amount available to deal.
    function dealAssets(address to, uint256 amount) public override {}

    /// @dev Some yield sources accrue by time, so it's difficult to accrue an exact amount. Call the 
    /// function multiple times if the test requires the accrual of more yield than the default amount.
    function _accrueYield() internal override {
        vm.startPrank(alice);
        IERC20(underlyingAsset).transfer(address(yieldVault), 10000000);
    }

    /// @dev Simulates loss on the yield vault such that the value of it's shares drops
    function _simulateLoss() internal override {}






    function test__BackrunWithdrawWithHugelyProfitableLiquidationPOC() public {
        // This is to represent thousands of users depositing over time
        vm.startPrank(alice);
        prizeVault.deposit(79228162514264337593443950335, alice);

        // This represents the liquidator depositing in the vault at some point
        vm.startPrank(attacker);
        prizeVault.deposit(100000000, attacker);

        // Yield accrual
        _accrueYield();

        vm.stopPrank();

        // Honest liquidator sees the yield accrue and sends a Tx to liquidate it, but it will revert
        uint256 availableAssets = prizeVault.liquidatableBalanceOf(address(underlyingAsset));
        console.log("availableAssets to liquidate =", availableAssets);

        vm.expectRevert();
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), availableAssets);

        // Attacker waits a few days so that elapsed time accumulates
        skip(3 days);

        // Attacker withdraws their deposit
        vm.startPrank(attacker);
        prizeVault.withdraw(prizeVault.balanceOf(attacker), attacker, attacker);

        vm.stopPrank();

        // Attacker backruns the withdraw with a liquidation at a much lower price since so much time passed
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), availableAssets);
    }
}
```

Console Output

```terminal
Ran 1 test for test/integration/BackrunWithdrawWithHugelyProfitableLiquidationPOC.sol:BackrunWithdrawWithHugelyProfitableLiquidationPOC
[PASS] test__BackrunWithdrawWithHugelyProfitableLiquidationPOC() (gas: 597935)
Logs:
  availableAssets to liquidate = 989999

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.15ms (798.19µs CPU time)

Ran 1 test suite in 13.31ms (2.15ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L744-L750

## Tool used

Manual Review

## Recommendation
This will require a complex solution to prevent introducing other vulnerabilities, possibly redesigning the mint limit



## Discussion

**nevillehuang**

Same root cause for all issues and duplicates highlighted, that is a TWAB_SUPPLY_LIMIT must be triggered via large deposits, a potential fix would be to support a higher TWAB_SUPPLY_LIMIT

# Issue H-2: Vault portion calculation in `PrizePool::getVaultPortion()` is incorrect as `_startDrawIdInclusive` has been erased 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/96 

## Found by 
0x73696d616f
## Summary

The vault portion calculation in `PrizePool::getVaultPortion()` is incorrect because it fetches a `_startDrawIdInclusive` that has been overwritten in the total accumulator but not in the vaults accumulator or the donation accumulator.

## Vulnerability Detail

To illustrate the issue, consider the grand prize, where odds are `1 / g = 0.00273972602e18` with `g == 365 days`. The [estimated](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1014) prize frequency is `1e18 / 0.00273972602e18 == 365.000000985` rounded up to the nearest integer, `366`. If `lastAwardedDrawId_` is `366`, then `startDrawIdInclusive` is `366 - 366 + 1 == 1`. 
This means that it will fetch the total, vault and donation accumulators since the opening of `drawId == 1`. 

However, when `lastAwardedDrawId_ == 366`, the current open `drawId` is `367`, which is written to index 0 in the circular buffer, overwriting `drawId == 1`. But, in the donation and vault cases, the accumulator still has `drawId == 1`, as the buffer will likely not have been overwritten (draw periods take 1 day, it is highly likely the individual accumulators are not written to every day, but the total accumulator is, or users may exploit this on purpose). As the buffer is circular, this will keep happening every time after the buffer has been filled and indexes start being overwritten.

Thus, due to this, the vault portion will be bigger than it should, as the total accumulator is calculated between 365 days (from 2 to 366, as 1 has been erased), but the vault and donation accumulators for 366 days (between 1 and 366). Users will have bigger than expected chances of winning prizes and in the worst case, they may even not be able to claim prizes at all, in case the erased `drawId` had a significant contribution to the prizes through a donation, and then it underflows when doing `totalSupply = totalContributed - totalDonated;` in [PrizePool::getVaultShares()](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1129).

Add the following test to `PrizePool.t.sol` and log the vault and total portions in `PrizePool::getVaultShares()`. The contribution of the vault will be bigger than the total.
```solidity
function testAccountedBalance_POC() public {
  for (uint i = 0; i < 366; i++) {
    contribute(100e18);
    awardDraw(1);
    mockTwab(address(this), msg.sender, 0);
  }
  address otherVault = makeAddr("otherVault");
  vm.prank(otherVault);
  prizePool.contributePrizeTokens(otherVault, 0);
  uint256 prize = claimPrize(msg.sender, 0, 0);
}
```

## Impact

Users get better prize chances than supposed and/or it can lead to permanent reverts when trying to claim prizes.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1014

## Tool used

Manual Review

Vscode

## Recommendation

When calculating `PrizePool::computeRangeStartDrawIdInclusive()`, the `rangeSize` should be capped to `365` to ensure the right `startDrawIdInclusive` is fetched, not one that has already been erased. In the scenario above, if the range is capped at 365, when `lastAwardedDrawId_ == 366`, `startDrawIdInclusive` would be `366 - 365 + 1 == 2`, which has not been erased as the current `drawId` is `367`, having only overwritten `drawId == 1`, but 2 still has the information. 

# Issue H-3: Draw auction rewards likely exceed the available rewards, resulting in overpaying rewards or running into an `InsufficientReserve` error 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/125 

## Found by 
0x73696d616f, MiloTruck, berndartmueller, hash, infect3d, trachev, zraxx
## Summary

The sum of the calculated `startDraw` and `finishDraw` reward fractions likely exceeds `1.0`, which can lead to overpaying rewards or an `InsufficientReserve` error.

## Vulnerability Detail

The `finishDraw` function calculates the rewards for starting and finishing a draw and distributes them to the respective participants. The total rewards (`availableRewards`) are based on the available prize pool reserve ([upper capped by `maxRewards`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L507)). Those rewards are then split into the start draw rewards and the finish draw rewards. Internally, the portion each participant receives is denominated in fractions ([0.00, 1.00]). No more than `availableRewards` (i.e., the upper cap or the total available prize pool reserve)can be distributed.

However, it is very likely that the sum of all fractions exceeds `1.0`, which would result in utilizing more rewards than `availableRewards`. Consequently, if the reserve has been larger than the `maxRewards` upper cap, more rewards would be paid out than `availableRewards`. Or, if the reserve has been smaller than `maxRewards`, which means the entire reserve is supposed to get used for rewards, using more than `1.0 * availableRewards` would [result in an `InsufficientReserve` error](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L439).

Basically, in the former case, rewards are overpaid, and in the latter case, the draw can not be awarded due to the `InsufficientReserve` error.

The following test case demonstrates this issue by having two `startDraw` calls, both of which are supposed to receive `~0.1e18` rewards, which would already exceed the `0.1e18` available reserve. Please note that due to mocking the calls to the prize pool, the actual `InsufficientReserve` error is not triggered. However, by inspecting the emitted events, it can be seen that the rewards are overpaid.

```solidity
function test_finishDraw_multipleStarts_with_exceeding_reserve() public {
    vm.warp(1 days + auctionDuration);

    mockReserve(.1e18, 0);

    // 1st start draw
    mockRng(99, 0x1234);
    uint firstStartReward = 99999999999999999; // ~0.1e18
    vm.expectEmit(true, true, true, true);
    emit DrawStarted(address(this), alice, 1, auctionDuration, firstStartReward, 99, 1);
    drawManager.startDraw(alice, 99);

    // RNG fails
    mockRngFailure(99, true);

    // 2nd start draw
    vm.warp(1 days + auctionDuration + auctionDuration);
    vm.roll(block.number + 1);
    mockRng(100, 0x1234);
    uint secondStartReward = 99999999999999999; // ~0.1e18 - Would already exceed the available reserve of 0.1e18
    vm.expectEmit(true, true, true, true);
    emit DrawStarted(address(this), bob, 1, auctionDuration, secondStartReward, 100, 2);
    drawManager.startDraw(bob, 100);

    mockFinishDraw(0x1234);
    vm.warp(1 days + auctionDuration*3);
    uint finishReward = 99999999999999999; // ~0.1e18
    mockAllocateRewardFromReserve(alice, firstStartReward);
    mockAllocateRewardFromReserve(bob, secondStartReward);
    mockAllocateRewardFromReserve(bob, finishReward);
    mockReserveContribution(.1e18);
    assertEq(drawManager.finishDrawReward(), finishReward, "finish draw reward matches");
    drawManager.finishDraw(bob);
}
```

Please copy and paste the above test case into the `pt-v5-draw-manager/test/DrawManager.t.sol` file and run it with `forge test -vv --match-test "test_finishDraw_multipleStarts_with_exceeding_reserve"`.

## Impact

Either rewards for `startDraw` and `finishDraw` are overpaid, or the draw can not be awarded.

## Code Snippet

[DrawManager.finishDraw#L314-L352](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L314-L352)

```solidity
314: function finishDraw(address _rewardRecipient) external returns (uint24) {
...    // [...]
333:   uint256 availableRewards = _computeAvailableRewards();
334:   (uint256[] memory startDrawRewards, UD2x18[] memory startDrawFractions) = _computeStartDrawRewards(
335:     prizePool.drawClosesAt(startDrawAuction.drawId),
336:     availableRewards
337:   );
338:   (uint256 _finishDrawReward, UD2x18 finishFraction) = _computeFinishDrawReward(
339:     startDrawAuction.closedAt,
340:     block.timestamp,
341:     availableRewards
342:   );
343:   uint256 randomNumber = rng.randomNumber(startDrawAuction.rngRequestId);
344:   uint24 drawId = prizePool.awardDraw(randomNumber);
345:
346:   lastStartDrawFraction = startDrawFractions[startDrawFractions.length - 1];
347:   lastFinishDrawFraction = finishFraction;
348:
349:   for (uint256 i = 0; i < _startDrawAuctions.length; i++) {
350:     _reward(_startDrawAuctions[i].recipient, startDrawRewards[i]);
351:   }
352:   _reward(_rewardRecipient, _finishDrawReward);
```

## Tool used

Manual Review

## Recommendation

Consider ensuring that the sum of all fractions does not exceed `1.0` to prevent overpaying rewards or running into an `InsufficientReserve` error.



## Discussion

**trmid**

The suggested mitigation is a good addition to the draw auction logic, however the difference in behaviour between the fix and the current behaviour is minimal:

**Current Behaviour:**

When there is not enough reserve to pay for the `finishDraw`, the draw will not be awarded and the prize liquidity will gracefully roll over to the next draw (the prize pool is well-equipped to handle skipped draws). The only loss here is on the agent that started the draw, who will not receive the rewards for doing so.

**Behaviour After Fix**

When there is not enough reserve to pay for the `finishDraw`, the auction price will remain at the value of whatever is left until the end of the auction incase a benevolent actor decides to push it through at a loss. If no one pushes it through, the draw will be skipped the same as before and the agent that started the draw will still receive no rewards.

The only difference is the amount of time that a benevolent actor has to push the draw through at a loss. No core protocol functionality is broken since the prize pool is only expected to award prizes if there are enough incentives to do so, and it's built to handle skipped draws without loss.

**nevillehuang**

@trmid I see users that was supposed to be awarded in the original draw as losing their funds as well (since they lose their chance to win), not just the agent that started the draw.

Was this behavior documented as expected behavior? If not I would lean towards agreeing that high severity is appropriate. 

# Issue M-1: `DrawManager.startDraw()` can be called after prize pool shutdown, even though `finishDraw()` always reverts 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/26 

## Found by 
MiloTruck, trachev
## Summary

After the prize pool is shutdown, `startDraw()` can still be called even though `finishDraw()` will always revert. This causes a loss of funds for draw bots that do so as draw rewards are never paid out.

## Vulnerability Detail

`PrizePool.awardDraw()` has the `notShutdown` modifier, which means that it cannot be called once `block.timestamp` exceeds [`shutdownAt()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L957-L963):

[PrizePool.sol#L455](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L455)

```solidity
  function awardDraw(uint256 winningRandomNumber_) external onlyDrawManager notShutdown returns (uint24) {
```

However, `DrawManager.startDraw()` does not check if the prize pool has been shutdown. The only time-based condition in `startDraw()` is whether the current `drawId` to award has closed:

[DrawManager.sol#L222-L224](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L222-L224)

```solidity
    uint24 drawId = prizePool.getDrawIdToAward(); 
    uint48 closesAt = prizePool.drawClosesAt(drawId);
    if (closesAt > block.timestamp) revert DrawHasNotClosed();
```

As such, it is possible for draw bots to call `startDraw()` after the pool has shutdown, even though it is no longer possible to call `finishDraw()` since `PrizePool.awardDraw()` will revert.

When draw bots call `startDraw()` to start awarding a draw, they pay gas costs + the RNG fee for requesting randomness. As such, when `startDraw()` is called after the pool is shutdown, draw bots will never be refunded for gas costs + the RNG fee as draw rewards are paid in `finishDraw()`.

## Impact

When draw bots call `startDraw()` after the prize pool is shutdown, they experience a loss of funds as `finishDraw()` cannot be called, so draw rewards are never paid out to cover the cost of calling `startDraw()`.

Note that the likelihood of this happening is not low - draw bots will most likely call [`canStartDraw()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L275-L291), which also does not check if the prize pool has shutdown, to determine if they should call `startDraw()`.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L455

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L222-L224

## Tool used

Manual Review

## Recommendation

In `startDraw()`, consider checking if `block.timestamp` is greater than [`shutdownAt()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L957-L963) and reverting if so.

# Issue M-2: Draws can be retried even if a random number is available or the current draw has finished 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/27 

## Found by 
MiloTruck
## Summary

`RngWitnet.isRequestFailed()` does not accurately represent whether a random number is available, making it possible for `startDraw()` to be called in conditions that causes a loss of funds for draw bots.

## Vulnerability Detail

`RngWitnet.isRequestFailed()` determines if a request has failed by checking the response status of a specific block number is equal to `WitnetV2.ResponseStatus.Error`:

[RngWitnet.sol#L115-L118](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L115-L118)

```solidity
    function isRequestFailed(uint32 _requestId) onlyValidRequest(_requestId) public view returns (bool) {
        (uint256 witnetQueryId,,) = witnetRandomness.getRandomizeData(requests[_requestId]);
        return witnetRandomness.witnet().getQueryResponseStatus(witnetQueryId) == WitnetV2.ResponseStatus.Error;
    }
```

Note that `requests[_requestId]`, stores the block number at which the request at `_requestId` was made.

However, `isRequestFailed()` does not accurately reflect if a random number is available. In Witnet, `isRandomized()` (which is used in [`isRequestComplete()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L108-L110)) returns `true` and `fetchRandomnessAfter()` (which is used in [`randomNumber()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L120-L125)) returns a valid random number as long as there is a successful request at or after the requested block number:

[WitnetRandomnessV2.sol#L334-L336](https://github.com/witnet/witnet-solidity-bridge/blob/2602a43e10b6dae3e681e567ce7de1c0204f4d5e/contracts/apps/WitnetRandomnessV2.sol#L334-L336)

```solidity
    /// @notice Returns `true` only if a successfull resolution from the Witnet blockchain is found for the first 
    /// @notice non-errored randomize request posted on or after the given block number.
    function isRandomized(uint256 _blockNumber)
```

[WitnetRandomnessV2.sol#L128-L136](https://github.com/witnet/witnet-solidity-bridge/blob/2602a43e10b6dae3e681e567ce7de1c0204f4d5e/contracts/apps/WitnetRandomnessV2.sol#L128-L136)

```solidity
    /// @notice Retrieves the result of keccak256-hashing the given block number with the randomness value 
    /// @notice generated by the Witnet Oracle blockchain in response to the first non-errored randomize request solved 
    /// @notice after such block number.
    /// @dev Reverts if:
    /// @dev   i.   no `randomize()` was requested on neither the given block, nor afterwards.
    /// @dev   ii.  the first non-errored `randomize()` request found on or after the given block is not solved yet.
    /// @dev   iii. all `randomize()` requests that took place on or after the given block were solved with errors.
    /// @param _blockNumber Block number from which the search will start
    function fetchRandomnessAfter(uint256 _blockNumber)
```

For example, assume there are two requests:

- Request 1 made at `block.number = 100` failed.
- Request 2 made at `block.number = 101` was successful.

If `isRandomized()` and `fetchRandomnessAfter()` was called for block number 100, they would return `true` and a valid random number respectively. By extension, `RngWitnet.isRequestComplete()` would return `true` for request 1. However, `RngWitnet.isRequestFailed()` also returns `true` for request 1, forming a contradiction.

As such, it is possible for `RngWitnet.isRequestFailed()` to return `true` for a given `_requestId`, even when [`RngWitnet.isRequestComplete()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L108-L110) returns `true` and [`RngWitnet.randomNumber()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L120-L125) returns a valid random number.

In `DrawManager.startDraw()`, if `RngWitnet.isRequestFailed()` returns `true`, draw bots are allowed to call `startDraw()` again to submit a new request for the current draw to "retry" the randomness request:

[DrawManager.sol#L238-L239](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L238-L239)

```solidity
      if (!rng.isRequestFailed(lastRequest.rngRequestId)) { // if the request failed
        revert AlreadyStartedDraw();
```

Since `isRequestFailed()` might wrongly return `true` as described above, it is possible for draw bots to retry a randomness request even when a random number is actually available.

Additionally, it is possible for `startDraw()` to be called after a draw has concluded with `finishDraw()`. For example:

- `startDraw()` is called at `block.number = 100`.
- Witnet receives a randomness request from another user/protocol at `block.number = 101`.
- The randomness request at `block.number = 100` fails. 
- The randomness request at `block.number = 101` succeeds.
- `finishDraw()` is called, which works as there is a successful request after `block.number = 100`.
- Now, if `startDraw()` is called, it will pass as `isRequestFailed()` returns `true` for the request at `block.number = 100`.

Note that it is not feasible for draw bots to check if `finishDraw()` has been called before calling `startDraw()`, as another draw bot can front-run their transaction calling `startDraw()` to call `finishDraw()`.

## Impact

`startDraw()` can be called to retry the current draw even when (a) a random number is already available, or (b) when the current draw has already finished. (a) causes a loss of funds to other draw bots that called `startDraw()` in the current draw as their rewards are diluted, while (b) causes a loss of funds to the draw bots that call `startDraw()` after the draw has finished as they never receive rewards.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L115-L118

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L238-L239

## Tool used

Manual Review

## Recommendation

In `startDraw()`, consider checking `isRequestComplete()` to know if a random number is available:

```diff
-     if (!rng.isRequestFailed(lastRequest.rngRequestId)) { // if the request failed
+     if (rng.isRequestComplete(lastRequest.rngRequestId)) { // if the request failed
        revert AlreadyStartedDraw();
```

# Issue M-3: `drawTimeoutAt()` causes the prize pool to shutdown one draw earlier 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/28 

## Found by 
MiloTruck, hash
## Summary

The shutdown timestamp returned by `drawTimeoutAt()` is one draw period early, causing the protocol to shut down one draw earlier than expected.

## Vulnerability Detail

In `PrizePool.sol`, the prize pool shuts down if the number of unawarded draws in a row is equal to `drawTimeout`:

[PrizePool.sol#L283-L284](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L283-L284)

```solidity
  /// @notice The maximum number of draws that can be missed before the prize pool is considered inactive.
  uint24 public immutable drawTimeout;
```

`drawTimeout` is used in `drawTimeoutAt()`, which determines the timestamp at which the pool shuts down:

[PrizePool.sol#L973-L975](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L973-L975)

```solidity
  function drawTimeoutAt() public view returns (uint256) { 
    return drawClosesAt(_lastAwardedDrawId + drawTimeout);
  }
```

As seen from above, the pool shuts down at the close time of `drawId = _lastAwardedDrawId + drawTimeout`. However, this causes the pool to shut down one draw earlier than expected as draws can only be awarded after their close time in `awardDraw()`:

[PrizePool.sol#L460-L465](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L460-L465)

```solidity
    uint24 awardingDrawId = getDrawIdToAward();
    uint48 awardingDrawOpenedAt = drawOpensAt(awardingDrawId);
    uint48 awardingDrawClosedAt = awardingDrawOpenedAt + drawPeriodSeconds;
    if (block.timestamp < awardingDrawClosedAt) {
      revert AwardingDrawNotClosed(awardingDrawClosedAt);
    }
```

To illustrate the problem:

- Assume the following:
  - `drawTimeout = 1`, which means the pool should shut down after one draw has been missed.
  - No draws have been awarded yet, so `_lastAwardedDrawId = 0`.
- The first draw to award is always `drawId = 1`.
- When `awardDraw()` is called before `drawId = 2`, the check in `awardDraw()` will revert as `block.timestamp` is less than the close time of `drawId = 1`.
- When `awardDraw()` is called during or after `drawId = 2` (ie. `block.timestamp` is greater than the close time of `drawId = 1`):
  - `_lastAwardedDrawId + drawTimeout = 0 + 1 = 1`
  - `drawTimeoutAt()` returns the close time of `drawId = 1`, so the pool has already shut down.
  - `awardDraw()` reverts due to the `notShutdown` modifier.

As seen from above, `awardDraw()` can never be called when `drawTimeout = 1`, even though a draw was never missed. This demonstrates how `drawTimeoutAt()` returns a timestamp one draw period early.
 
## Impact

When `drawTimeout = 1`, the prize pool will never award prizes to any vaults and will immediately shut down, causing a loss of yield for depositors. Furthermore, this breaks core functionality of the protocol as the only incentive for users to deposit into vaults is the chance of winning a huge prize.

When `drawTimeout > 1`, the prize pool will shut down when `drawTimeout - 1` consecutive draws have been missed, which is one draw early.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L283-L284

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L973-L975

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L460-L465

## Tool used

Manual Review

## Recommendation

Modify `drawTimeoutAt()` to return the close time of `_lastAwardedDrawId + drawTimeout + 1`:

```diff
  function drawTimeoutAt() public view returns (uint256) { 
-   return drawClosesAt(_lastAwardedDrawId + drawTimeout);
+   return drawClosesAt(_lastAwardedDrawId + drawTimeout + 1);
  }
```

# Issue M-4: Distributing liquidity based on the last `grandPrizePeriodDraws` days post-shutdown is problematic 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/36 

The protocol has acknowledged this issue.

## Found by 
MiloTruck, trachev, volodya
## Summary

After a prize pool shuts down, its funds are distributed based on draws before it shut down. This could cause a loss of funds for prize pools that contribute new liquidity to the pool post-shutdown.

## Vulnerability Detail

After a vault is shutdown, all existing and new liquidity is distributed based on a user and vault's time-weighted prize contribution during the last `grandPrizePeriodDraws` before shutdown:

[PrizePool.sol#L873-L895](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L873-L895)

```solidity
  function computeShutdownPortion(address _vault, address _account) public view returns (ShutdownPortion memory) {
    uint24 drawIdPriorToShutdown = getShutdownDrawId() - 1;
    uint24 startDrawIdInclusive = computeRangeStartDrawIdInclusive(drawIdPriorToShutdown, grandPrizePeriodDraws);

    (uint256 vaultContrib, uint256 totalContrib) = _getVaultShares(
      _vault,
      startDrawIdInclusive,
      drawIdPriorToShutdown
    );

    (uint256 _userTwab, uint256 _vaultTwabTotalSupply) = getVaultUserBalanceAndTotalSupplyTwab(
      _vault,
      _account,
      startDrawIdInclusive,
      drawIdPriorToShutdown
    );

    if (_vaultTwabTotalSupply == 0) {
      return ShutdownPortion(0, 0);
    }

    return ShutdownPortion(vaultContrib * _userTwab, totalContrib * _vaultTwabTotalSupply);
  }
```

As seen from above, the draw ID used as the ending draw is `getShutdownDrawId() - 1`, which is the draw before the pool shut down.

However, this method of allocating liquidity is problematic when considering that new liquidity can be contributed to the prize pool, even after it is shutdown.

Firstly, if a vault shuts down before any liquidity is contributed through `contributePrizeTokens()`, all future liquidity contributed to the prize pool will be permanently stuck. For example:

- A prize pool and vault are newly deployed.
- Assume that `drawTimeout = 1`, which means the prize pool shuts down after one draw is missed.
- In the first draw period, the prize pool does not generate any yield, and therefore, it does not contribute any liquidity.
- The first draw is missed as there is no prize liquidity, so the pool shuts down.
- In the second draw period, the prize pool generates some yield and calls `contributePrizeTokens()` to add it to the prize pool.
- When users of the vault attempt to withdraw the yield from the prize pool through `withdrawShutdownBalance()`:
  - In `computeShutdownPortion()`, `totalContrib = 0` as no liquidity was contributed before the pool shut down.
  - In `shutdownBalanceOf()`, `shutdownPortion.denominator = 0`, so the withdrawable balance for all users is `0`.
- As such, since all users cannot withdraw any balance from the prize pool, the new prize liquidity is effectively stuck forever.

Secondly, prize pools that did not contribute any liquidity before the pool shut down, but start doing so post-shutdown will always lose their yield. For example:
- Assume two vaults are deployed, called vault A and B.
- Before the pool shuts down:
  - Vault A calls `contributePrizeTokens()` and contributes some prize liquidity.
  - Vault B does not generate any yield, so it does not contribute any prize liquidity.
- After the pool shuts down, vault B starts generating yield and consistently calls `contributePrizeTokens()`.
- However, since vault B did not contribute any liquidity before the pool shut down, all contributions from vault B are only withdrawable by users of vault A.
- This causes a loss of yield for users in vault B.

Note that since vaults call `contributePrizeTokens()` without checking if the pool has shut down and the liquidation of yield is performed automatically by liquidation bots, it is not unlikely for a vault to contribute liquidity to a pool after it is shut down.

## Impact

When a vault contributes to the prize pool post-shutdown but did not do so before shutdown, all users of the vault lose their yield, causing a loss of funds.

In the case where the prize pool did not receive any liquidity before shutdown, all funds contributed post-shutdown will be permanently stuck in the pool, causing a loss of yield for all users.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Consider a different method of allocating funds in the prize pool after shutdown. For example, the ending draw in `computeShutdownPortion()` could include the latest draw.



## Discussion

**trmid**

The prize pool can enter shutdown mode for two reasons:

1. the draw timeout has been reached due to the termination of the RNG service
2. the TWAB timestamps have reached their max limit

The second is guaranteed to happen at the end of life for a TWAB controller and was a major consideration in the design of the shutdown logic. With this consideration in mind, the computed shutdown balances cannot look at TWAB past the shutdown timestamp, and therefore must use past TWAB to allocate balances on a best-effort basis. While the shutdown allocations can be unfair to edge cases, it provides simple rules for distributions that consider the historical weight of a vault's contributions while remaining within the limitations of a shutdown environment.

The shutdown logic is not designed to support contributions from new vaults that have never contributed before. It's a bit like trying to open an account at an institution that just declared bankruptcy. While it may not be ideal for everyone, prize vault owners that don't like the shutdown distribution can change the liquidation strategy on the vault to redirect any yield to a different distribution system.

# Issue M-5: Price formula in `TpdaLiquidationPair._computePrice()` does not account for a jump in liquidatable balance 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/38 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, MiloTruck
## Summary

The linearly decreasing auction formula in `TpdaLiquidationPair._computePrice()` does not account for sudden increases in the vault's liquidatable balance, causing the price returned to be much lower than it should be.

## Vulnerability Detail

The price paid by liquidation bots to liquidate the asset balance in a vault is calculated in `TpdaLiquidationPair._computePrice()` as shown:

[TpdaLiquidationPair.sol#L191-L195](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L191-L195)

```solidity
        uint256 elapsedTime = block.timestamp - lastAuctionAt;
        if (elapsedTime == 0) {
            return type(uint192).max;
        }
        uint192 price = uint192((targetAuctionPeriod * lastAuctionPrice) / elapsedTime);
```

As seen from above, the price paid decreases linearly over time from `targetAuctionPeriod * lastAuctionPrice` to 0. This allows the liquidation pair to naturally find a price that liquidation bots are willing to pay for the current liquidatable balance in the vault, which is calculated as shown:

[TpdaLiquidationPair.sol#L184-L186](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L184-L186)

```solidity
    function _availableBalance() internal returns (uint256) {
        return ((1e18 - smoothingFactor) * source.liquidatableBalanceOf(address(_tokenOut))) / 1e18;
    }
```

The vault's liquidatable balance is determined by `liquidatableBalanceOf()`:

[PrizeVault.sol#L687-L709](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L687-L709)

```solidity
    function liquidatableBalanceOf(address _tokenOut) external view returns (uint256) {
        uint256 _totalDebt = totalDebt();
        uint256 _maxAmountOut;
        if (_tokenOut == address(this)) {
            // Liquidation of vault shares is capped to the mint limit.
            _maxAmountOut = _mintLimit(_totalDebt);
        } else if (_tokenOut == address(_asset)) {
            // Liquidation of yield assets is capped at the max yield vault withdraw plus any latent balance.
            _maxAmountOut = _maxYieldVaultWithdraw() + _asset.balanceOf(address(this));
        } else {
            return 0;
        }

        // The liquid yield is limited by the max that can be minted or withdrawn, depending on
        // `_tokenOut`.
        uint256 _availableYield = _availableYieldBalance(totalPreciseAssets(), _totalDebt);
        uint256 _liquidYield = _availableYield >= _maxAmountOut ? _maxAmountOut : _availableYield;

        // The final balance is computed by taking the liquid yield and multiplying it by
        // (1 - yieldFeePercentage), rounding down, to ensure that enough yield is left for
        // the yield fee.
        return _liquidYield.mulDiv(FEE_PRECISION - yieldFeePercentage, FEE_PRECISION);
    }
```

Apart from the yield vault suddenly gaining yield, there are two reasons why the vault's liquidatable balance might suddenly increase:

- (1) The liquidatable balance is no longer restricted by the mint limit after someone withdraws from the vault. For example:
  - Assume `_tokenOut == address(this)`.
  - The vault is currently at the mint limit, so `_maxAmountOut = _mintLimit(_totalDebt) = 0`.
  - A user withdraws from the vault, causing the mint limit to decrease. Assume that `_maxAmountOut` is now `1000e18`, which means 1000 more shares can be minted.
  - Since `_liquidYield` is restricted by `_maxAmountOut`, it jumps from `0` to `1000e18`, causing a sudden increase in the liquidatable balance.
- (2) The vault's owner calls `setYieldFeePercentage()` to decrease `yieldFeePercentage`. As seen from above, this will cause the returned value from `liquidatableBalanceOf()` to suddenly increase.

However, since the vault's liquidatable balance is not included in price calculation and the price decreases linearly, `TpdaLiquidationPair._computePrice()` cannot accurately account for instantaneous increases in `liquidatableBalanceOf()`. 

Using (1) to illustrate this:

- Assume that the vault's token is USDT and the prize token is USDC.
- Since the vault is currently at its mint limit, the liquidatable balance is 0 USDT.
- Over time, the price calculated in `_computePrice()` decreases to near-zero to match the liquidatable balance.
- A user suddenly withdraws, which increases the mint limit and causes the liquidatable balance to jump to 1000 USDT.
- However, the price returned by `_computePrice()` will still remain close to 0 USDC. 
- This allows a liquidation bot to swap 1000 USDT in the vault for 0 USDC.

As seen from above, when a sudden increase in liquidatable balance occurs, the price in `_computePrice()` could be too low due to the linearly decreasing auction. 

## Impact

When `liquidatableBalanceOf()` suddenly increases due to an increase in the mint limit or a reduction in `yieldFeePercentage`, liquidation bots can swap the liquidatable balance in the vault for less than its actual value. This causes a loss of funds for the vault, and by extension, all users in the vault.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L191-L195

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L184-L186

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L687-L709

## Tool used

Manual Review

## Recommendation

Consider using an alternative auction formula in `_computePrice()`, ideally one that takes into account the current liquidatable balance in the vault.



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> that is the reason of the MIN_PRICE state variable



**trmid**

If the yield source is expected to have large jumps in yield (by design or by the prize vault owner's actions) the prize vault owner can either:

1. set a suitable `smoothingFactor` on the liquidation pair to mitigate the effect within a reasonable efficiency target
2. pause liquidations during times of anticipated flux and set a new liquidation pair when ready (this would be good practice if suddenly lowering the yield fee percentage).

# Issue M-6: The claimer's fee will be stolen by the winner 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/73 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, cu5t0mPe0, jovi, ydlee
## Summary

The user calls `claimPrizes` to collect prizes for the winner, earning some rewards in the process. However, during this process, the winner can steal the fees collected by the user without paying any gas fees.

## Vulnerability Detail

The user calls `claimPrizes` to collect prizes for the winner, thereby earning a reward fee.

Throughout the process, the winner can set hooks before and after calling `prizePool.claimPrize`. The winner can set an `afterClaimPrize` hook and then call `claimer.claimPrizes` again within this `afterClaimPrize` hook, thereby earning reward fees without using any gas. As a result, the user can only collect the reward fee for one winner, while the remaining reward fees for other winners will all be collected by the first winner.

Consider the following scenario:

1. User A calls `claimer.claimPrizes` to claim prizes, setting the winners as [B, C, D, E, F] (B, C, D, E, F are all winners).
2. User B notices in the mempool that User A is about to call `claimer.claimPrizes` to claim prizes for others. User B then calls `setHooks` to set the `afterClaimPrize` function, which implements a logic to loop call `claimer.claimPrizes` with winners set as [C, D, E, F].
3. User A's transaction is executed, but since User B has already claimed the rewards for C, D, E, and F, User A's attempts to claim prizes for C, D, E, and F will revert. However, due to the `try-catch`, User A's transaction will continue executing. In the end, User A will only receive the reward fee for User B.

This is essentially the same attack method as the previous PoolTogether vulnerability. The last audit did not completely fix it. The difference this time is that the hook uses `afterClaimPrize`.

https://code4rena.com/reports/2024-03-pooltogether#m-01-the-winner-can-steal-claimer-fees-and-force-him-to-pay-for-the-gas

## Impact

The user will lose the reward fee

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L110-L117

## Tool used

Manual Review

## Recommendation

If the hook's purpose is for managing whitelists and blacklists, PoolTogether can create a template contract that users can manage. This way, the hook can only be set to the template contract and cannot perform additional operations.

If users are allowed to perform any operation, there is no good solution to this problem. The only options are to limit the gas, use reentrancy locks, or remove the after hook altogether.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> if the attacker can front-run one prize then he is better off front-running the whole claimPrizes call and earn even more reward



**trmid**

The risks of front-running claims is a known issue from previous audits: https://code4rena.com/reports/2023-07-pooltogether#m-24-claimerclaimprizes-can-be-front-runned-in-order-to-make-losses-for-the-claim-bot

Although the method described here is slightly more complex, the end result is still very similar to the more straightforward approach of frontrunning the entire tx.

It is also worth noting that the prize hooks have a gas limit of 150k gas, which would cause this strategy to fail when trying to frontrun a large number of prizes.

**nevillehuang**

I believe medium severity is appropriate here, this issue wasn't highlighted as a known issue/risk, and watsons has highlighted another vector that allows stealing of claimer fees

# Issue M-7: `PUSH0` opcode Is Not Supported on Linea yet 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/79 

The protocol has acknowledged this issue.

## Found by 
0xSpearmint1, aman, elhaj, volodya
## Summary

## Vulnerability Detail
 - The current codebase is compiled with Solidity `version 0.8.24`, which includes the `PUSH0` opcode in the compiled bytecode. According to the [README](https://github.com/sherlock-audit/2024-05-pooltogether?tab=readme-ov-file#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed), the protocol will be deployed on the Linea network.
 
 -  This presents an issue because Linea does not yet support the `PUSH0` opcode, which can lead to unexpected behavior or outright failures when deploying and running the smart contracts.[see here](https://docs.linea.build/developers/quickstart/ethereum-differences#evm-opcodes)
 
## Impact
- Deploying the protocol on `Linea` with the current Solidity version (0.8.24) may result in unexpected behavior or failure due to the unsupported `PUSH0` opcode.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L2C1-L3C1
## Tool used
 Manual Review
## Recommendation
- for Linea you may consider to use version 0.8.19 to compile .



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> contract will simply not deploy so no risk/loss__ see bullet 24 of invalid findings in sherlock rules



**nevillehuang**

Valid medium, since if current contract is deployed as is, it will have issues and as per noted in contest details, so the contest details will override sherlock rule 24 since it is stated as follows:

> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity. High severity will be applied only if the issue falls into the High severity category in the judging guidelines.

# Issue M-8: Potential ETH Loss Due to transfer Usage in Requestor Contract on `zkSync` 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/80 

## Found by 
0x73696d616f, 0xAadi, MiloTruck, elhaj, trachev
## Summary
- The [Requestor](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/Requestor.sol#L36) contract uses `transfer` to send `ETH` which has the risk that it will not work if the gas cost increases/decrease(low Likelihood), but it is highly likely to fail on `zkSync` due to gas limits. This may make users' `ETH` irretrievable.

## Vulnerability Detail

- Users (or bots) interact with [RngWitnet](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L132) to request a random number and start a draw in the `DrawManager` contract. To generate a random number, users must provide some `ETH` that will be sent to [WitnetRandomness](https://github.com/witnet/witnet-solidity-bridge/blob/6cf9211928c60e4d278cc70e2a7d5657f99dd060/contracts/apps/WitnetRandomnessV2.sol#L372) to generate the random number.
```js
    function startDraw(uint256 rngPaymentAmount, DrawManager _drawManager, address _rewardRecipient) external payable returns (uint24) {
            (uint32 requestId,,) = requestRandomNumber(rngPaymentAmount);
            return _drawManager.startDraw(_rewardRecipient, requestId);
    }
 ```
- The `ETH` sent with the transaction may or may not be used (if there is already a request in the same block, it won't be used). Any remaining or unused `ETH` will be sent to [Requestor](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/Requestor.sol#L36), so the user can withdraw it later.
- The issue is that the `withdraw` function in the `Requestor` contract uses `transfer` to send `ETH` to the receiver. This may lead to users being unable to withdraw their funds .
```js
 function withdraw(address payable _to) external onlyCreator returns (uint256) {
        uint256 balance = address(this).balance;
 >>       _to.transfer(balance);
        return balance;
 }
```

- The protocol will be deployed on different chains including `zkSync`, on `zkSync` the use of `transfer` can lead to issues, as seen with [921 ETH Stuck in zkSync Era](https://medium.com/coinmonks/gemstoneido-contract-stuck-with-921-eth-an-analysis-of-why-transfer-does-not-work-on-zksync-era-d5a01807227d).since it has a fixed amount of gas `23000` which won't be anough in some cases even to send eth to an `EOA`, It is explicitly mentioned in their docs to not use the `transfer` method to send `ETH` [here](https://docs.zksync.io/build/quick-start/best-practices.html#use-call-over-send-or-transfer).

>  notice that in case `msg.sender` is  a contract that have some logic on it's receive or fallback function  the `ETH` is definitely not retrievable. since this contract can only withdraw eth to it's [own addres](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L99-L102) which will always revert.

## Impact

- Draw Bots' `ETH` may be irretrievable or undelivered, especially on zkSync, due to the use of `.transfer`.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/Requestor.sol#L27-L30
## Tool used
Manual Review , Foundry Testing
## Recommendation
- recommendation to use `.call()` for `ETH` transfer.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> invalid__ only affect bots that deliberately chose to set code inside the fallback function



**trmid**

The zkSync [doc link](https://docs.zksync.io/build/quick-start#use-call-over-send-or-transfer) provided in the issue does not seem to have information on this behaviour anymore. Is there an updated link on the issue?

**nevillehuang**

request poc

**sherlock-admin4**

PoC requested from @elhajin

Requests remaining: **1**

**elhajin**

- here the correct quote from the docs :
[Use call over .send or .transfer](https://docs.zksync.io/build/developer-reference/best-practices#use-call-over-send-or-transfer)




# Issue M-9: Withdrawals in the `PrizeVault` will not work for some vaults due to using `previewWithdraw()` and then `redeem()` 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/92 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, 0xmystery, AuditorPraise
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



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> work as expected__ SR mixed previewRedeem values of prizeVault and yieldVault and recommandation break EIP4626 for prizeVault



**trmid**

The 4626 spec is somewhat loose in the relationship between shares, assets, and the functions that convert them. Like the issue states, there is no *guaranteed* relationship between the withdrawal and redeem accounting. The spec even states that:

> Details such as accounting and allocation of deposited tokens are intentionally not specified, as Vaults are expected to be treated as black boxes on-chain and inspected off-chain before use.

The prize vault requires that these accounting functions are related and use the same accounting as part of the "Dust Collection Strategy" described: https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L38 

Any discrepancy between the assets returned from `redeem` when passing in the shares computed from `previewWithdraw` would indicate that there exists either some fee or non-symmetrical accounting which would break the entire integration (not just the `_withdraw` function). As stated in the prize vault, yield sources with fees on deposit / withdraw flows are not supported: https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L66

Given that the 4626 spec is so loosely defined that there could exist an implementation with no correlation at all between withdraw and redeem, it is not unreasonable to expect deployers of a prize vault to verify that the yield source they are integrating is indeed a "normal" integration that also contains no fees.

**nevillehuang**

Valid medium, since it was mentioned as the following

> PrizeVaults are expected to strictly comply with the ERC4626 standard.

Since the word "MUST" is used, based on the following sherlock guidelines:

> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity. High severity will be applied only if the issue falls into the High severity category in the judging guidelines.

# Issue M-10: Estimated prize draws in TieredLiquidityDistributor are off due to rounding down when calculating the sum, leading to incorrect prizes 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/95 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f
## Summary

Estimated prize draws in `TieredLiquidityDistributor` are off due to rounding down when calculating the expected prize count on each tier, leading to an incorrect next number of tiers and distribution of rewards.

## Vulnerability Detail

`ESTIMATED_PRIZES_PER_DRAW_FOR_5_TIERS` and the other tiers are precomputed initially in `TieredLiquidityDistributor::_sumTierPrizeCounts()`. In [here](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L574), it goes through all tiers and calculates the expected prize count per draw for each tier. However, when doing this [calculation](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L113-L115) in `TierCalculationLib.tierPrizeCountPerDraw()`, `uint32(uint256(unwrap(sd(int256(prizeCount(_tier))).mul(_odds))));`, it rounds down the prize count of each tier. This will lead to an incorrect prize count calculation, for example:

```solidity
grandPrizePeriodDraws == 8 days
numTiers == 5
prizeCount(tier 0) == 4**0 * 1 / 8 == 1 * 1 / 8 == 0.125 = 0
prizeCount(tier 1) == 4**1 * (1 / 8)^sqrt((1 + 1 - 3) / (1 - 3)) == 4 * 0.2298364718 ≃ 0.92 = 0
prizeCount(tier 2) == 4**2 * 1 == 16
prizeCount(tier 3) == 4**3 * 1 == 64
total = 80
```
However, if we multiply the prize counts by a constant and then divide the sum in the end, the total count would be 81 instead, getting an error of 1 / 81 ≃ 1.12 %

## Impact

The estimated prize count will be off, which affects the calculation of the next number of tiers in [PrizePool::computeNextNumberOfTiers()](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L472). This modifies the whole rewards distribution for the next draw.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L574
https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L113-L115

## Tool used

Manual Review

Vscode

## Recommendation

Add some precision to the calculations by multiplying, for example, by `1e5` each count and then dividing the sum by `1e5`. `uint32(uint256(unwrap(sd(int256(prizeCount(_tier)*1e5)).mul(_odds))));` and `return prizeCount / 1e5;`.



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> rounding isn't an issue by itself it looks like a design choice



**nevillehuang**

Request PoC to facilitate discussion

Sponsor comments:
> Quality assurance at best, rounding errors are perfectly acceptable here and have no significant impact

What is the maximum impact here?

**sherlock-admin4**

PoC requested from @0x73696d616f

Requests remaining: **3**

**0x73696d616f**

Hi @nevillehuang, some context first:
Rewards are assigned based on the max tier for a draw, starting at tier 4. 
So users can claim rewards on tiers 0, 1, 2, 3. Tier 5 has tiers 0, 1, 2, 3, 4. And so on.
Each prize tier is claimed 4^t numbers of times for each user, where t is the tier.
Each tier has certain odds of occuring, according to the formula [here](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L24-L26).
The prize is distributed by awarding each tier pro-rata `tierShares` and canary tiers (the last 2) receive `canaryShares`.
So, the more tiers, the more the total prize pool is distributed between tiers, decreasing the prize of each tier.

Now some specific example:
When we go from max tier 4 to tier 5, the prize pool is dilluted to the new tier, decreasing the rewards for users for each tier (there are more shares).
Going to tier 5 also means that there is 1 tier that has [less](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L591) odds of being claimed. Tier 1 is no longer awarded every draw.
Thus, as we have dilluted the prize in the tiers, and added a tier with less odds of occuring, the total rewards expected are now less.
This mechanism works to offset luck in the draw. If a certain draw is claimed too many times compared to the expected value, it will likely increase the max tier in the next draw, and decrease the expected rewards of the next draw.

And finally, the issue:
The expected number of prizes collected on a certain max tier, ex 4, is calculated by summing the expected prizes collected on each tier (this is well explained in the issue).
As the expected number will be off, the expected claim count will also be off.
In the issue example, tier 5 has an incorrect expected value of 80 instead of 81.
If the number of claimed prizes is 81, [> 80](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L546), it will advance to tier 5.
However, if the expected value was correctly calculated as 81, we would not advance to tier 5, but remain in tier 4.
Consequently, the next draw will award less rewards, on average, than it should, as it incorrectly increased the max tier.
Here is the calculation of the expected rewards being decreased for the next round, based on the numbers above:
```solidity
prizePool == 1000
numShares = 100
canaryTierShares = 5

// tier 4
totalShares = 210
tierValue = 1000*100/210 = 476
canaryValue = 1000*5/210 = 24

476*0.125 + 476*1 + 24 = 560

// tier 5
totalShares = 310
tierValue = 1000*100/310 = 323
canaryValue = 1000*5/310 = 16

323*0.125 + 323*0.23 + 323*1 + 16 = 454
```


**trmid**

The existing algorithm returns the expected number of tiers based on claims. The calculation *informs* the rest of the system on the number of tiers. As long as the tiers can go up and down based on the network conditions, there is no unexpected consequence of the calculation rounding down.

# Issue M-11: When `PrizePool` reserves are low and more than expected number of prizes are won, race conditions are created to claim prizes. Some users' prize funds will be locked due to `PrizePool::claimPrize()` call reverting. 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/112 

The protocol has acknowledged this issue.

## Found by 
Nihavent, elhaj, evmboi32
### When `PrizePool` reserves are low and more than expected number of prizes are won, race conditions are created to claim prizes. Some users' prize funds will be locked due to `PrizePool::claimPrize()` call reverting

## Summary

The determination of winners for a given draw and tier are statistically independant events. That is, if Alice winning a prize for a given draw and tier does not make it more or less likely that Bob will win a prize in the same draw and tier. Consequently, there may be more (or less) than expected winners in any given draw. 
This can be an issue because the `tierLiquidity.prizeSize` for a given tier and draw is based on the expected number of winners, not the actual number of winners. In the case where there is more than the expected number of winners there may not be enough liquidity in the reserve to cover the extra prize(s). 
Under these circumstances, there is a race for users' to call `PrizePool::claimPrize()` before the liqudiity in the tier runs out. Once tier liquidity runs out, there will be user(s) who are geneuine winners for a given draw and tier (`PrizePool::isWinner()` returns true), but calls to `PrizePool::claimPrize()` for their address will revert, resulting in their prize-funds being locked.

As an analogy, the protocol is currently like a lottery where there can be more than 1 grand prize winner (tier 0). In the event there is 2 grand prize winners, it allows both users to make a claim on the full grand prize instead of allocating an equal portion to each winner. The issue is present in each tier level with varying frequencies and impacts.

## Vulnerability Detail

The expected number of prizes awarded for any given draw and tier is given by 4^t where t is the tier number. For example, in tier 1, there is 4 expected prizes. In reality, this means over a large number of draws the average number of tier 1 prizes awarded per draw will approach 4.

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L39-L41

The way this is implemented, in tier t, is by allowing users to generate 4^t different `userSpecificRandomNumber` in the `PrizePool::isWinner()` function. This means, users get 4^t chances to generate a random number in the winning zone (ie. in tier t, users are 4^t more likely to be able to claim a prize than in tier 0).

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1022-L1029

```javascript
    uint256 userSpecificRandomNumber = TierCalculationLib.calculatePseudoRandomNumber(
      lastAwardedDrawId_,
      _vault,
      _user,
      _tier,
@>    _prizeIndex,
      _winningRandomNumber
    );
```

The user wins a prize for a given vault, draw, tier, and prize index if their user-specific uniformly distributed psuedo-random number falls within the `winning zone`:

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L68-L70 

Due to the statistical independence of each user's random numbers generated, there is theoretically no limit to the number of prizes that can be won in a given draw and tier. This results in the possibility of the required liquidity in a tier exceeding the available liquidity in that tier. 

The issue is made possible because the `tierLiquidity.prizeSize` is updated when a user calls `PrizePool::claimPrize()` via `TieredLiquidityDistributor::_getTier()` and `TieredLiquidityDistributor::_computePrizeSize()`. As shown below, the prize size returned is based on dividing the `remainingTierLiquidity` by `prizeCount` which is the <u>expected prize count (4^t)</u>, not the <u>actual prize count</u> for that draw and tier.

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L411-L429

```javascript
  function _computePrizeSize(
    uint8 _tier,
    uint8 _numberOfTiers,
    uint128 _tierPrizeTokenPerShare,
    uint128 _prizeTokenPerShare
  ) internal view returns (uint104) {
    uint256 prizeCount = TierCalculationLib.prizeCount(_tier);
    uint256 remainingTierLiquidity = _getTierRemainingLiquidity(
      _tierPrizeTokenPerShare,
      _prizeTokenPerShare,
      _numShares(_tier, _numberOfTiers)
    );

    uint256 prizeSize = convert(
@>    convert(remainingTierLiquidity).mul(tierLiquidityUtilizationRate).div(convert(prizeCount))
    );

    return prizeSize > type(uint104).max ? type(uint104).max : uint104(prizeSize);
  }
```

Crucially, note that in the event there are less than the expected number of winners in a draw and tier, the protocol does not 'save' these extra prizes in reserves to protect against this issue ocurring in the future. It simply inflates that tier's liquidity for future draws. As shown above, in future draws `tierLiquidity.prizeSize` will be recalculated based on `remainingTierLiquidity` and `prizeCount`.

Also note there is no protection against this issue by ensuring the reserves exceed a sufficient size. At any point, the  `drawManager` can allocate any amount of the reserves to any address as a reward:

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L434-L449

The risk of at least one prize being unclaimable is present in any draw where a tier's prize size `tierLiquidity.prizeSize` is greater than the reserve amount. Note that unless the value of the grand prize is kept in the reserve, the issue will be present in every single draw.


## Impact

When `tierLiquidity.prizeSize` exceeds `TieredLiquidityDistributor._reserve`, and there are more winners than the expected number of winners, a race to call `PrizePool::claimPrize()` begins. In tier t, the first 4^t prize claims will be successful, with each subsequent call reverting.

Addresses that were entitled to a prize, but were not the first 4^t winning addresses to claim will <b><u>forfeit their prize</b></u>.

## Code Snippet

The POC below was adapted from a test in `PrizePool.t.sol` and shows a scenario where five users can make a prize claim in tier 1 for a given draw. There is not enough liquidity for all users to claim, so the fifth claim reverts.

Paste the following import and test in PrizePool.t.sol.

```javascript

import { InsufficientLiquidity } from "../src/abstract/TieredLiquidityDistributor.sol";

  function test_ClaimPrizesRevertsScenarioOne() public {
    uint users = 50;
    uint balance = 1e18;
    uint totalSupply = users * balance;
    uint8 t = 1; // Show revert in tier 1
    uint256 yieldAmt = 100e18;

    // Vault contributes yield to prize pool
    contribute(yieldAmt);

    // Draw is awarded with this random number
    uint256 random = uint256(keccak256(abi.encode(1234)));
    awardDraw(random);

    // Based on this seed, addresses, and TWAB values, Users 10, 15, 16, 18, and 41 win a prize in tier 1
    address user10 = makeAddr(string(abi.encodePacked(uint(10))));
    address user15 = makeAddr(string(abi.encodePacked(uint(15))));
    address user16 = makeAddr(string(abi.encodePacked(uint(16))));
    address user18 = makeAddr(string(abi.encodePacked(uint(18))));
    address user41 = makeAddr(string(abi.encodePacked(uint(41))));
    address user42 = makeAddr(string(abi.encodePacked(uint(42))));

    // Mock user TWAB
    mockTwabForUser(address(this), user10, t, balance);
    mockTwabForUser(address(this), user15, t, balance);
    mockTwabForUser(address(this), user16, t, balance);
    mockTwabForUser(address(this), user18, t, balance);
    mockTwabForUser(address(this), user41, t, balance);
    mockTwabForUser(address(this), user42, t, balance);

    // Mock vault TWAB
    mockTwabTotalSupply(address(this), t, totalSupply);

    // 5 Different users are able to claim prizes for this draw in tier 1
    assertEq(prizePool.isWinner(address(this), user10, 1, 0), true);
    assertEq(prizePool.isWinner(address(this), user15, 1, 2), true);
    assertEq(prizePool.isWinner(address(this), user16, 1, 3), true);
    assertEq(prizePool.isWinner(address(this), user18, 1, 0), true);
    assertEq(prizePool.isWinner(address(this), user41, 1, 1), true);

    // There is not enough liquidity for all 5 winners
    assertGt(5 * prizePool.getTierPrizeSize(1), prizePool.getTierRemainingLiquidity(1) + prizePool.reserve());

    // Race condition is has been created, first 4 users can claim
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user10, 1, 0, user10, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user15, 1, 2, user15, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user16, 1, 3, user16, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user18, 1, 0, user18, 0, address(this)));

    // Fifth user's claim reverts due to insufficient liquditiy
    vm.expectRevert(abi.encodeWithSelector(InsufficientLiquidity.selector, prizePool.getTierPrizeSize(1)));
    prizePool.claimPrize(user41, 1, 1, user41, 0, address(this));
  }
```

## Tool used

Manual Review

## Recommendation

Without changing the core logic of winner selection, my recommended mitigation falls into two categories:

  1. Waiting until the end of the claim period before calculating `tierLiquidity.prizeSize` and distributing pool tokens. This means the payout values for a given draw and tier will be based on the actual number of winners, not the expected number of winners. As an example, if two addresses were to win the grand prize on the same draw, they would each recieve half of the total grand prize.
  This implementation would be more like a lottery where the total prize pool in each tier is guaranteed, but not the value of each winner's payout, this is only determined after the number of successful winners has been determined.

  2. Managing the size of the reserve pool (`TieredLiquidityDistributor._reserve`) by taking steps to keep the reserve pool above some 'minimum reserve buffer':

- Requiring the reserve stays above the 'minimum reserve buffer' after each call to `prizePool::allocateRewardFromReserve()`
- Bootstrapping reserve pool to the 'minimum reserve buffer' by temporarily increasing `TieredLiquidityDistributor.reserveShares` upon PrizePool deployment, or if reserve dips below 'minimum buffer'
- For draws where the actual number of prize claims is less than the expected number, transfer the extra prizes to the reserve until the reserve pool exceed some 'minimum reserve buffer'.
  
    Note that the size of 'minimum reserve buffer' depends on the protocol's risk appetite for this issue arrising. To reduce the risk significantly, ideally the size of the reserve would exceed the liquidity available in tier 0. However the significant tradeoff is user's would recieve less yield because the reserve would need to grow in accordance with the grand prize pool. 





## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> the PoC do not take the _reserve into account which exists for that specific purpose 



**trmid**

This is a well known limitation and design choice of the prize pool. As mentioned in the issue, the prize pool operates on a statistical model, awarding prizes based on tier odds. The prize pool offers two different tools for mitigating this variance:

1. The reserve can be used to backstop "over-awarded" prizes with extra liquidity. This can be managed manually through the `contributeReserve` function, or automatically through the cut of contributions captured by the `reserveShares`. However, in recent deployments, the reserve is no longer needed for this purpose due to a new parameter:
2. The `tierUtilizationRatio` can be configured such that the prizes only use a portion of the available yield in each prize tier. When configured correctly, this alone is enough to ensure that there will be enough liquidity for every prize within reasonable statistical variance. Although the absolute possibility of an over-awarded prize is not completely removed by this, the chance of a race condition can be brought so low such that there is no reasonable incentive for claimers to try and win the race. It is also worth noting that the most likely prizes to be over awarded are the smallest prizes in the system, therefore lowering impact of the condition even further.

While the issue is valid, the prize pool already offers multiple configuration parameters to mitigate it such that any properly configured prize pool will not be significantly impacted.

**nevillehuang**

@trmid Was this design choice/limitation documented? If not I am leaning towards valid medium severity based on your comments above.

# Issue M-12: Unfair Manipulation of Winning Chances Due to Stolen Yield on `Blast` 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/114 

## Found by 
0xSpearmint1, elhaj, jo13
## Summary
- The `PoolTogether` protocol on [Blast](https://docs.blast.io/about-blast) will use `WETH` as it's prize token as mentioned by sponsor, which generates yield over time. This yield can be stolen and used to manipulate the vaults' winning chances unfairly. By contributing the stolen yield to their own vault, users can inflate their vault's portion, increasing their chances of winning prizes while decreasing the chances for other legitimate users. This undermines the fairness of the prize distribution.
## Vulnerability Detail

- The `PoolTogether` protocol will be deployed on [Blast](https://docs.blast.io/about-blast), and as the sponsor mentioned, `WETH` will be the prize token.
- `WETH` on [Blast](https://docs.blast.io/about-blast) is a rebasing token that automatically generates yield over time. `WETH` holders have three Yield Modes for rebasing: `Void`, `Automatic`, and `Claimable`. The **Default** mode is `Automatic`, which means the balance will increase over time by the yield generated.
- The [prizePool]() will hold `WETH` as the prize token and will generate yield over time. However, this yield can be stolen by anyone, as they can contribute it to their own vault by calling the `contributePrizeTokens()` function, passing their vault and the amount of yield generated.
```js
function contributePrizeTokens(address _prizeVault, uint256 _amount) public returns (uint256) {
  >>  uint256 _deltaBalance = prizeToken.balanceOf(address(this)) - accountedBalance();
    if (_deltaBalance < _amount) {
      revert ContributionGTDeltaBalance(_amount, _deltaBalance);
    }
    uint24 openDrawId_ = getOpenDrawId();
   >> _vaultAccumulator[_prizeVault].add(_amount, openDrawId_);
   >> _totalAccumulator.add(_amount, openDrawId_);
    emit ContributePrizeTokens(_prizeVault, openDrawId_, _amount);
    return _deltaBalance;
  }

```
- The idea is that this yield is generated from all vault contributions. However, a malicious user can exploit this feature to their advantage. While this might be considered a missing feature (not benefiting from yield) in different protocol contexts, within the `PoolTogether` on [Blast](https://docs.blast.io/about-blast) scenario, it has a detrimental effect. The malicious user's ability to exploit the yield generation artificially inflates their vault's portion compared to other vaults. **This act will decrease other vaults portions consequently increases the malicious user vault portion, thus increase his chances of winning prizes and decrease others chances**.
- For the malicious user's vault, the increase in their contribution portion will lead to a higher `winning zone` thus more chances to win. Conversely, for other vaults, the decrease in their contribution portions will lead to a smaller  `winning zone` reducing their chances of winning. 

- This manipulation undermines the fairness of the prize distribution. Not only will the yield be stolen from all vaults contributions and not be spread to them proportionally, but it will also affect their winning chances by decreasing them. This gives the malicious user an unfair advantage by increasing their likelihood of winning at the expense of others.

### Scenario

**Normal Case**

- Let's say from draw 4 to draw 40, we have:
  - `vault1.contributionBetween = 100`
  - `vault2.contributionBetween = 100`
  - `totalContributionBetween = 200`
- To calculate their portions:
  - `v1.portion = 100 / 200 = 0.5`
  - `v2.portion = 100 / 200 = 0.5`

> Assuming the TWAB balance is 500 for both vaults and they both have 1 user for easy calculation(so users twab also 500) , and ignoring the odds scaling:
  - `winningZone(vault1[user]) = 500 * 0.5 = 250`
  - `winningZone(vault2[user]) = 500 * 0.5 = 250`

**Yield Stolen Case**

- Now let's assume that a malicious user continuously steals yield during this period (from draw 4 to draw 40) 
> `Notice` that the yield will be generated for all the PrizePool balance (probably more than 200 which totalContribution in this period):
  - When we get the contribution of `vault3` (malicious vault) for this period (from draw 4 to 40), we get:
    - `vault3.contribution = 50`

- Now let's recalculate the vaults' portions again:
  - `vault1.contributionBetween = 100`
  - `vault2.contributionBetween = 100`
  - `vault3.contributionBetween = 50`
  - `totalContributionBetween = 250`
- To calculate their portions:
  - `v1.portion = 100 / 250 = 0.4`
  - `v2.portion = 100 / 250 = 0.4`
  - `v3.portion = 50 / 250 = 0.2`

> Assuming the TWAB balance is 500 for all vaults:
  - `winningZone(vault1[user]) = 500 * 0.4 = 200`
  - `winningZone(vault2[user]) = 500 * 0.4 = 200`
  - `winningZone(vault3[maliciousUser) = 500 * 0.2 = 100`

- Notice how the malicious user now has a chance to win without any contribution from it's vault (only stolen yield), and how legitimate users vaults' chances have decreased.

## Impact
- Malicious actors can steal yield generated by the prize pool, inflating their vault's portion and unfairly increasing their win chances. This decreases legitimate players' portion and winning probabilities.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L1037-L1043 
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L412-L422
## Tool used
manual review
## Recommendation
- For `Blast`, configure the yield mode to `Claimable` in the constructor so it doesn't affect the `balanceOf()`. Expose a public function that claims this yield and contributes it to the `DONATOR` address. This way, the yield is not wasted, and all contributors benefit from it. Contributors will be incentivized to call this function since it increases the prize size.

```js
 // import those :
 enum YieldMode {
  AUTOMATIC,
  VOID,
  CLAIMABLE
}
interface IERC20Rebasing {
  function configure(YieldMode) external returns (uint256);
  function claim(address recipient, uint256 amount) external returns (uint256);
  function getClaimableAmount(address account) external view returns (uint256);
}
```
```diff

contract prizePool {

 constructor() {

   // you can check by address of by chainid if you are in blast : 
+  if (address(prizeToken) == 0x4300000000000000000000000000000000000004){
+        IERC20Rebasing(address(prizeToken)).configure(YieldMode.CLAIMABLE); //configure claimable yield for WETH
+    }
+  }

+ function claimYield() external returns (uint256){
+    IERC20Rebasing weth = IERC20Rebasing(address(prizeToken));
+    uint amount = weth.getClaimableAmount(address(this));
+     weth.claim(address(this),amount);
+     contributePrizeTokens(DONATOR, amount);
+  }
}
```



## Discussion

**trmid**

The blast docs [here](https://docs.blast.io/building/guides/eth-yield#overview) state that:

> Smart contract accounts have three Yield Modes for their rebasing mode:
> 
> - Void (DEFAULT): ETH balance never changes; no yield is earned
> - Automatic: native ETH balance rebases (increasing only)
> - Claimable: ETH balance never changes; yield accumulates separately

The docs say the default for EOAs is rebasing, but the default for the contracts like the prize pool will be void.

**elhajin**

 @trmid that's not true that's the case for native ETH not  WETH here a quote from docs in case of WETH: 

> Similar to ETH, WETH and USDB on Blast is also rebasing and follows the same yield mode configurations.
**However, unlike ETH where contracts have Disabled yield by default, WETH and USDB accounts have Automatic yield by default for both EOAs and smart contracts**

**nevillehuang**

@elhajin You are right based on this [link here](https://docs.blast.io/building/guides/weth-yield). Seems like this is a valid issue.

# Issue M-13: `Claimers` can receive less `feePerClaim` than they should if some prizes are already claimed or if reverts because of a reverting hook 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/124 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, hash, infect3d, jovi, zraxx
## Summary

If a claimer propose an array of prizes to claim, but some of these prizes have already been claimed, or some claims revert, then the actual `feePerClaim` received will be less **compared to what it should really be, as expected by the VRGDA algorithm**

This happens because `_computeFeePerClaim` can undervaluate the value of `feePerClaim`, as it compute failed claims as successful ones.

This poses an issue, as `feePerClaim` is then used as an input by `_vault.claimPrize(_winners[w], _tier, _prizeIndices[w][p], _feePerClaim, _feeRecipient)`, which will transfer the undervaluated fee to the claimer.


## Vulnerability Detail
Auctions for claiming prizes are based on the [VRGDA algorithm](https://www.paradigm.xyz/2022/08/vrgda)
In simple terms, this algorithm update price depending on either the numbers of claim is behind or ahead of time schedule.
In order to have a schedule, a target of claim per time unit is defined.
Just to give an idea, let's simplify that to the extreme (we will see complete formula afterward) and say : `price(t) = claim/expected * targetPrice(t)` 
E.g: if 10 claims per 2 hours are exepected, then at t=1h, 5 claims should be concluded.
If only 4 were claimed, then we can calculate that (4 claims)/(5 expected) < 1, price will be lower that target.
if 6 were claimed, then we will have (6 claims)/(5 expected) > 1, price will be greater than target.

The formula that has been implemented into `LinearVRGDALib` is the following:

```haskell
price = p0 * e^ (k * (t - n+1/r))   ; with k = ln(maxFee/minFee) * t_target
```

With:
- `n` the number of claim already completed
- `r` the expected rate per hour
- `k` the decay constant (speed at which price will change)
- `p0` the target price (or fee in our case)

The more `k *(t - n+1/r) > 0`, the more `price > p0`
When `t = n+1/r` <=> `(k * (t - n+1/r)) = 0`, then `price = p0`
The more `k * (t - n+1/r) < 0`, the more `price < p0`

We understand that the more whe are behind schedule in term of expected claim, the higher the fees earned by claimer will be.
And the more we are ahead of schedule, the lower the fee for claimers will be (as their is no urgency)

### Scenario
Now let's see what happens in this scenario:
1. For now, 0 prizes have already been claimed from the prize pool, `n = 0`
2. Alice has 5 prizes to claim, she build her claim array
3. It seems that 2 of the prizes Alice was going to claim are claimed right before, now `n = 2`
4. Alice tx is executed with the 5 prizes

Now let's see what happen from a code perspective.
The code above is the entry point for claiming prizes. As we can see L113, the `feePerClaim` is computed based on the [number of claims to count](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L126-L136) (`_countClaims`) and the number of [already claimed prizes](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-prize-pool/src/PrizePool.sol#L302-L302) (`prizePool::claimCount()`)
Then, the computed `feePerClaim` value is given to `_claim` which actually claim the prizes into the Prize Pool.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
```solidity
File: pt-v5-claimer/src/Claimer.sol

090:   function claimPrizes(
091:     IClaimable _vault,
092:     uint8 _tier,
093:     address[] calldata _winners,
094:     uint32[][] calldata _prizeIndices,
095:     address _feeRecipient,
096:     uint256 _minFeePerClaim
097:   ) external returns (uint256 totalFees) {
...:
...:          /* some code */
...:
112:     if (!feeRecipientZeroAddress) {
113:       feePerClaim = SafeCast.toUint96(_computeFeePerClaim(_tier, _countClaims(_winners, _prizeIndices), prizePool.claimCount()));
114:       if (feePerClaim < _minFeePerClaim) {
115:         revert VrgdaClaimFeeBelowMin(_minFeePerClaim, feePerClaim);
116:       }
117:     }
118:
119:     return feePerClaim * _claim(_vault, _tier, _winners, _prizeIndices, _feeRecipient, feePerClaim); 
120:   }
```

Now, let's see how `_computeFeePerClaim` actually compute `feePerClaim`.
We see above L230-241 that a fee is calculated for each of the claims of the array, starting at `_claimedCount` (The number of prizes already claimed) based on the VRGDA formula L309. The returned value (which is stored into `feePerClaim`) is the averaged fee as shown L241.
And as we explained earlier, the higher the number of claim we make, the lower the earned fee are. So, a higher value of `_claimedCount + i` will give lower fees.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236
```solidity
File: pt-v5-claimer/src/Claimer.sol
205:   /// @param _claimCount The number of claims to check
206:   /// @param _claimedCount The number of prizes already claimed
207:   /// @return The total fees for the claims
208:   function _computeFeePerClaim(
209:     uint8 _tier,
210:     uint256 _claimCount,
211:     uint256 _claimedCount
212:   ) internal view returns (uint256) {
...:
...:        /* some code */
...:
227:     uint256 elapsed = block.timestamp - (prizePool.lastAwardedDrawAwardedAt());
228:     uint256 fee;
229: 
230:     for (uint256 i = 0; i < _claimCount; i++) {
231:       fee += _computeFeeForNextClaim(
232:         targetFee,
233:         decayConstant,
234:         perTimeUnit,
235:         elapsed,
236:         _claimedCount + i,         
237:         _maxFee
238:       );
239:     }
240: 
241:     return fee / _claimCount;
242:   }
...:
...:        /* some code */
...:
301:   function _computeFeeForNextClaim(
302:     uint256 _targetFee,
303:     SD59x18 _decayConstant,
304:     SD59x18 _perTimeUnit,
305:     uint256 _elapsed,
306:     uint256 _sold,
307:     uint256 _maxFee
308:   ) internal pure returns (uint256) {
309:     uint256 fee = LinearVRGDALib.getVRGDAPrice(
310:       _targetFee,
311:       _elapsed,
312:       _sold,
313:       _perTimeUnit,
314:       _decayConstant
315:     );
316:     return fee > _maxFee ? _maxFee : fee;
317:   }
318: 
```

What we can see from this, is that the computation will be executed for `_claimCount = 2` and up to `i = 5`, so as if there has been 7 claimed prizes, while in reality only 5 prizes are claimed, leading in an undervaluation of the fees to award.
As you probably have infered, the computation should have been made for `i = 3` to be correct.

## Impact
The `feePerClaim` computation is incorrect as the VRGDA is calculated for more claims that will really happen, leading to less fee earned by claimers at the time of the call.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236

## Tool used
Manual Review

## Recommendation
The `PrizePool` contract expose a function to check if a prize has already been claimed: `wasClaimed`
This can be used to countClaims based on the actual true number of claimable prizes from the array.

This isn't a "perfect" solution though, as there are still issues when not already claimed prizes revert because of reverting prize hooks. In that case, VRGDA will still count the claim as happening, but we can consider this less likely to happen.

```diff
  function _countClaims(
    address[] calldata _winners,
    uint32[][] calldata _prizeIndices
  ) internal pure returns (uint256) {
    uint256 claimCount;
    uint256 length = _winners.length;
    for (uint256 i = 0; i < length; i++) {
-     claimCount += _prizeIndices[i].length;
+	  numPrize = _prizeIndices[i].length;
+	  for(uint256 j = 0; j < numPrize; j++) {
+     	bool wasClaimed = wasClaimed(_vault, _winner, _drawId,_tier, _prizeIndex);
+     	if(!wasClaimed) {
+		 claimCount += 1;
+		}
+     }
    }
    return claimCount;
  }
```



## Discussion

**trmid**

The prize pool must be passed the fee during each individual claim call so it can give the claimer their reward. The fees are calculated before in bulk and then split evenly for each claim in the batch. As the issue demonstrates, it is possible for some of these claims to revert, resulting in less fees collected than if the claimer excluded the reverting claims.

It's important for the claimers to simulate the results before claiming to ensure their claims will result in the expected rewards. Claimers compete to perform claims at the lowest costs, so the bots that simulate results will have higher success rates and be able to claim at lower margins compared to those who don't.

The impact of this issue is very low for competitive bots since claim simulations are commonly available and encouraged.

# Issue M-14: The RNG finish draw auction rewards are overpaid due to missing to account for the time it takes to fulfill the Witnet randomness request 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/126 

The protocol has acknowledged this issue.

## Found by 
berndartmueller
## Summary

The rewards for finishing the draw and submitting the previously requested randomness result are slightly overpaid due to the incorrect calculation of the elapsed auction time, wrongly including the time it takes to fulfill the Witnet randomness request.

## Vulnerability Detail

The `DrawManager.finishDraw` function calculates the rewards for finishing the draw, i.e., providing the previously requested randomness result, via the `_computeFinishDrawReward` function in lines [`338-342`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L338-L342). The calculation is based on the difference between the timestamp when the start draw auction was closed (`startDrawAuction.closedAt`) and the current block timestamp. Specifically, a parabolic fractional dutch auction (PFDA) is used to incentivize shorter auction durations, resulting in faster randomness submissions.

However, the requested randomness result at block X is not immediately available to be used in the `finishDraw` function. According to the Witnet documentation, [the randomness result is available after 5-10 minutes](https://docs.witnet.io/smart-contracts/witnet-randomness-oracle/code-examples). This time delay is currently included when determining the elapsed auction time because `startDrawAuction.closedAt` marks the time when the randomness request was made, not when the result was available to be submitted.

Consequently, the protocol **always** overpays rewards for the finish draw. Over the course of many draws, this can lead to a significant overpayment of rewards.

It should be noted that the PoolTogether documentation about the draw auction timing [clearly outlines the timeline for the randomness auction](https://dev.pooltogether.com/protocol/design/draw-auction/#auction-timing) and states that finish draw auction price will rise **once the random number is available**:

> _The RNG auction for any given draw starts at the beginning of the following draw period and must be completed within the auction duration. Following the start RNG auction, there is an “unknown” waiting period while the RNG service fulfills the request._
>
> _**Once the random number is available**, the finished RNG auction will rise in price until it is called to award the draw with the available random number. Once the draw is awarded for a prize pool, it will distribute the fractional reserve portions based on the auction results that contributed to the closing of that draw._

## Impact

The protocol overpays rewards for the finish draw.

## Code Snippet

[pt-v5-draw-manager/src/DrawManager.sol#L339](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L339)

```solidity
311: /// @notice Called to award the prize pool and pay out rewards.
312: /// @param _rewardRecipient The recipient of the finish draw reward.
313: /// @return The awarded draw ID
314: function finishDraw(address _rewardRecipient) external returns (uint24) {
315:   if (_rewardRecipient == address(0)) {
316:     revert RewardRecipientIsZero();
317:   }
318:
319:   StartDrawAuction memory startDrawAuction = getLastStartDrawAuction();
...    // [...]
338:   (uint256 _finishDrawReward, UD2x18 finishFraction) = _computeFinishDrawReward(
339: ❌  startDrawAuction.closedAt,
340:     block.timestamp,
341:     availableRewards
342:   );
343:   uint256 randomNumber = rng.randomNumber(startDrawAuction.rngRequestId);
```

## Tool used

Manual Review

## Recommendation

Consider determining the auction start for the finish draw reward calculation based on the timestamp when the randomness result was made available. This timestamp is accessible by using the [`WitnetRandomnessV2.fetchRandomnessAfterProof`](https://github.com/witnet/witnet-solidity-bridge/blob/4d1e464824bd7080072eb330e56faca8f89efb8f/contracts/apps/WitnetRandomnessV2.sol#L196) function ([instead of `fetchRandomnessAfter`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-rng-witnet/src/RngWitnet.sol#L124)). The `_witnetResultTimestamp` return value can be used to better approximate the auction start, hence, paying out rewards more accurately.



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> this is exactly the `_firstFinishDrawTargetFraction` goal



**trmid**

The suggested mitigation is flawed since it would return the timestamp that the RNG was generated on the Witnet network, not the timestamp that it was relayed to the requesting network.

The auction should be set with an appropriate ~~`_firstFinishDrawTargetFraction`~~ `_auctionTargetTime` value such that the estimated auction value is reached after the 5-10 min expected relay time. In addition, the `maxRewards` parameter on the `DrawManager` contract helps to ensure a reasonable max limit to the total auction payout in the case of an unexpected outage.

**nevillehuang**

@berndartmueller Could you take a look at the above comment? Does an appropriately admin set `_firstFinishDrawTargetFraction` mitigate this issue? 

**berndartmueller**

Hey @nevillehuang!

> _The suggested mitigation is flawed since it would return the timestamp that the RNG was generated on the Witnet network, not the timestamp that it was relayed to the requesting network._

This statement is mostly true, **but**, the timestamp can also be the `block.timestamp` at the time when the RNG request was relayed to the requesting network. Specifically, when reporters use the `reportResult` function, the timestamp is set to [`block.timestamp`](https://github.com/witnet/witnet-solidity-bridge/blob/9783dca1f228d1d132214494546fa120f70da79e/contracts/core/defaults/WitnetOracleTrustableBase.sol#L581).

If reporters use one of the [two other reporting methods](https://github.com/witnet/witnet-solidity-bridge/blob/9783dca1f228d1d132214494546fa120f70da79e/contracts/core/defaults/WitnetOracleTrustableBase.sol#L597-L679), the timestamp is set to the time when the RNG was generated on the Witnet network (although it's not verified that reporters set this value exactly, the timestamp could also be set to [`block.timestamp`](https://github.com/witnet/witnet-solidity-bridge/blob/9783dca1f228d1d132214494546fa120f70da79e/contracts/core/defaults/WitnetOracleTrustableBase.sol#L610-L614)). To be fair, [on-chain Optimism transactions to the Witnet contract](https://optimistic.etherscan.io/txs?a=0x77703aE126B971c9946d562F41Dd47071dA00777&p=1) show that the `reportResultBatch` function is predominantly used.

However, reporters are free to use whatever relaying function they use, so it's possible that the timestamp is the time when the request was made available to the requesting network. Either way, IMO, this makes it a better approximation of when the random number is available to PoolTogether's `finishDraw`, than [using the time when `startDraw` was called](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L339) and the RNG requested (which neglects the time it needs for Witnet actors to relay the request to the Witnet chain, generate the random number and relay it back to the requesting chain)

> _The auction should be set with an appropriate _firstFinishDrawTargetFraction value such that the estimated auction value is reached after the 5-10 min expected relay time. In addition, the maxRewards parameter on the DrawManager contract helps to ensure a reasonable max limit to the total auction payout in the case of an unexpected outage._

`_firstFinishDrawTargetFraction` is provided once in the `constructor` and stored in `lastFinishDrawFraction`. And `lastFinishDrawFraction` is [updated in every `finishDraw` call](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L347). While it's true that this value is used to better estimate the expected rewards and the relay time, it's also not perfect. A single occurrence of a longer than usual RNG relay time is enough to set `lastFinishDrawFraction` to an outlier value (i.e., inflated value, but upper-capped by `maxRewards`).

Long story short, using the recommendation in this submission, in conjunction with providing a reasonable `_firstFinishDrawTargetFraction ` value by the admin, further improves the accuracy of the rewards calculation (i.e., prevents overpaying).

**trmid**

Removed the "disputed" tag since this seems like a valid issue for the auctions, but may not need additional mitigation.

I realized I quoted the wrong parameter in my statement above:

> The auction should be set with an appropriate _firstFinishDrawTargetFraction value such that the estimated auction value is reached after the 5-10 min expected relay time.

I meant to say `_auctionTargetTime` here instead of `_firstFinishDrawTargetFraction`.

# Issue M-15: Witnet is not available on some networks listed 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/127 

The protocol has acknowledged this issue.

## Found by 
0x73696d616f, 0xSpearmint1, AuditorPraise, trachev, volodya
## Summary

Witnet is not deployed on Blast, Linea, ZkSync, Zerion, so draws will not be possible there. Vaults can still be deployed and yield earned, so it will fail when trying to start or complete draws.

## Vulnerability Detail

The sponsor intends to know concerns when deploying the protocol to the list of blockchains in the [readme](https://audits.sherlock.xyz/contests/225). 
> We're interested to know if there will be any issues deploying the code as-is to any of these chains

One of these issues is that Witnet is not supported on Blast, Linea, ZkSync or Zerion, so an alternative rng source must be used.

Prize vaults may still be deployed, accruing yield and sending it to the prize pool, but the prize pool will not be able to award draws and will only refund users when it shutdowns.

## Impact

DoSed yield until the prize pool shutdown mechanism starts after the timeout.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L938
https://docs.witnet.io/smart-contracts/supported-chains

## Tool used

Manual Review

Vscode

## Recommendation

Use an alternative RNG source on the non supported chains.



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> from project docs: ""The startDraw auction process may be slightly different on each chain and depends on the RNG source that is being used for the prize pool.""



**trmid**

We are in active communication with the Witnet team to ensure that a deployment is available before deploying a prize pool on the network. In the unlikely case that Witnet will not be available on the target network, an alternate RNG source can be integrated.

**nevillehuang**

@trmid If the contracts are deployed as it is, then the inscope rng-witnet contract cannot be deployed, so per the following contest details:

> We're interested to know if there will be any issues deploying the code as-is to any of these chains, and whether their opcodes fully support our application.

and the following sherlock guidelines:

> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity. High severity will be applied only if the issue falls into the High severity category in the judging guidelines.

I believe this is a valid medium, if not it should have been made known as a known consideration

# Issue M-16: `DrawManager.canStartDraw` does not consider retried RNG requests when determining if a new draw auction can be started 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/129 

## Found by 
0x73696d616f, berndartmueller, dany.armstrong90, hash, trachev, ydlee
## Summary

Inconsistent checks in the `DrawManager.canStartDraw` function, neglecting to consider retried RNG requests, might lead to wrongly assuming that a new draw auction cannot be started.

## Vulnerability Detail

The `DrawManager.canStartDraw` function checks if the `startDraw` function can be called. However, the checks are not consistent with the `startDraw` function. Specifically, the check in line `289` to determine if the draw has expired is different than the auction duration check in the `startDraw` function in lines [`250-251`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L250-L251). The latter uses the last RNG request's `closedAt` timestamp to determine the elapsed **auction** time, to consider any retried failed RNG requests, while the former checks if the **draw** has expired, not considering retried RNG requests.

As a result, if for a given draw a RNG request has been retried, and thus the total elapsed time from the draw close until now (`block.timestamp`) might exceed the auction duration, off-chain actors calling the `canStartDraw` function might wrongly assume that the draw auction can not be started, even though such a call would succeed.

## Impact

As `canStartDraw` is also [called internally by the `startDrawReward` function](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L296-L298) and both functions are likely to be used by off-chain actors to determine if a new draw auction an be started, this might lead to wrongly assuming that a new draw auction cannot be started, even though it should be possible. As a result, the current draw might not get awarded.

## Code Snippet

[DrawManager.canStartDraw()](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L289)

```solidity
275: /// @notice Checks if the start draw can be called.
276: /// @return True if start draw can be called, false otherwise
277: function canStartDraw() public view returns (bool) {
278:   uint24 drawId = prizePool.getDrawIdToAward();
279:   uint48 drawClosesAt = prizePool.drawClosesAt(drawId);
280:   StartDrawAuction memory lastStartDrawAuction = getLastStartDrawAuction();
281:   return (
282:     (
283:       // if we're on a new draw
284:       drawId != lastStartDrawAuction.drawId ||
285:       // OR we're on the same draw, but the request has failed and we haven't retried too many times
286:       (rng.isRequestFailed(lastStartDrawAuction.rngRequestId) && _startDrawAuctions.length <= maxRetries)
287:     ) && // we haven't started it, or we have and the request has failed
288:     block.timestamp >= drawClosesAt && // the draw has closed
289: ❌  _computeElapsedTime(drawClosesAt, block.timestamp) <= auctionDuration // the draw hasn't expired
290:   );
291: }
```

## Tool used

Manual Review

## Recommendation

Consider using the last request's `closedAt` timestamp instead of `drawClosesAt` to determine if the auction has expired to consider failed RNG requests that have been retried by calling `startDraw` again.

# Issue M-17: User's might be able to claim their prizes even after shutdown 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/133 

## Found by 
Rhaydden, hash
## Summary
User's might be able to claim their prizes even after shutdown due to `lastObservationAt` and draw period difference

## Vulnerability Detail
There is no shutdown check kept on the `claimPrize` function. Hence user's can calim their prizes even after the pool has been shutdown if the draw has not been finalized

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524)
```solidity
  function claimPrize(
    address _winner,
    uint8 _tier,
    uint32 _prizeIndex,
    address _prizeRecipient,
    uint96 _claimReward,
    address _claimRewardRecipient
  ) external returns (uint256) {
    /// @dev Claims cannot occur after a draw has been finalized (1 period after a draw closes). This prevents
    /// the reserve from changing while the following draw is being awarded.
    uint24 lastAwardedDrawId_ = _lastAwardedDrawId;
    if (isDrawFinalized(lastAwardedDrawId_)) {
      revert ClaimPeriodExpired();
    }
```

The remaining balance after a shutdown is supposed to be allocated to user's based on their (vault prize contribution + twab contribution) via the `withdrawShutdownBalance` and is not supposed to be based on a random number ie. via `claimPrize`

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L938-L941)
```solidity
  function withdrawShutdownBalance(address _vault, address _recipient) external returns (uint256) {
    if (!isShutdown()) {
      revert PrizePoolNotShutdown();
    }
```

In case the draw period is different from the TWAB's period length, it is not necessary that the shutdown due to the `lastObservationAt` occurs at the end of a draw. In such a case, it will allow user's who are winners of the draw to claim their prizes and also withdraw their share of the `shutdown balance` hence stealing funds from others 

### POC
```diff
diff --git a/pt-v5-prize-pool/test/PrizePool.t.sol b/pt-v5-prize-pool/test/PrizePool.t.sol
index 99fe6b5..5ce7ad6 100644
--- a/pt-v5-prize-pool/test/PrizePool.t.sol
+++ b/pt-v5-prize-pool/test/PrizePool.t.sol
@@ -75,6 +75,7 @@ contract PrizePoolTest is Test {
   uint256 RESERVE_SHARES = 10;
 
   uint24 grandPrizePeriodDraws = 365;
+  uint periodLength;
   uint48 drawPeriodSeconds = 1 days;
   uint24 drawTimeout; // = grandPrizePeriodDraws * drawPeriodSeconds; // 1000 days;
   uint48 firstDrawOpensAt;
@@ -112,27 +113,26 @@ contract PrizePoolTest is Test {
 
   ConstructorParams params;
 
-  function setUp() public {
-    drawTimeout = 30; //grandPrizePeriodDraws;
-    vm.warp(startTimestamp);
+    function setUp() public {
+      // at end drawPeriod == 2 day, and period length in twab = 1 day
 
+    periodLength = 1 days;
+    drawPeriodSeconds = 2 days;
+    
+    // the last draw should be ending at lastObservation timestamp + 1 day
+    startTimestamp = 1000 days;
+    firstDrawOpensAt =  uint48((type(uint32).max / periodLength ) % 2 == 0 ? startTimestamp + 1 days : startTimestamp + 2 days);
+
+    
+
+    drawTimeout = 25854; // to avoid shutdown by drawTimeout when warping
+
+    vm.warp(startTimestamp + 1);
     prizeToken = new ERC20Mintable("PoolTogether POOL token", "POOL");
-    twabController = new TwabController(uint32(drawPeriodSeconds), uint32(startTimestamp - 1 days));
+    twabController = new TwabController(uint32(periodLength), uint32(startTimestamp));
 
-    firstDrawOpensAt = uint48(startTimestamp + 1 days); // set draw start 1 day into future
     initialNumberOfTiers = MINIMUM_NUMBER_OF_TIERS;
 
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_OFFSET, ()),
-      abi.encode(firstDrawOpensAt)
-    );
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_LENGTH, ()),
-      abi.encode(drawPeriodSeconds)
-    );
-
     drawManager = address(this);
     vault = address(this);
     vault2 = address(0x1234);
@@ -142,7 +142,7 @@ contract PrizePoolTest is Test {
       twabController,
       drawManager,
       tierLiquidityUtilizationRate,
-      drawPeriodSeconds,
+      uint48(drawPeriodSeconds),
       firstDrawOpensAt,
       grandPrizePeriodDraws,
       initialNumberOfTiers, // minimum number of tiers
@@ -155,6 +155,51 @@ contract PrizePoolTest is Test {
     prizePool = newPrizePool();
   }
 
+  function testHash_CanClaimPrizeAfterShutdown() public {
+    uint secondLastDrawStart = startTimestamp + (type(uint32).max / periodLength ) * periodLength - 3 days;
+
+    vm.warp(secondLastDrawStart + 1);
+    address user1 = address(100);
+    //address user2 = address(200);
+
+    // mint tokens to the user in twab
+    twabController.mint(user1, 10e18);
+    //twabController.mint(user2, 5e18);
+
+    // contribute prize tokens to the vault
+    prizeToken.mint(address(prizePool),100e18);
+    prizePool.contributePrizeTokens(address(this),100e18);
+
+    // move to the next draw and award this one
+    vm.warp(secondLastDrawStart + drawPeriodSeconds + 1);
+    prizePool.awardDraw(100);
+    uint drawId = prizePool.getOpenDrawId();
+
+    //currently not shutdown. but shutdown will occur in the middle of this draw allowing both prize claiming and the shutdown withdrawal
+    uint shutdownTimestamp = prizePool.shutdownAt();
+    assert(shutdownTimestamp == secondLastDrawStart + drawPeriodSeconds + periodLength);
+
+    vm.warp(shutdownTimestamp);
+
+    // call to store the shutdown data before the prize is claimed
+    prizePool.shutdownBalanceOf(address(this),user1);
+
+    /**
+     * address _winner,
+    uint8 _tier,
+    uint32 _prizeIndex,
+    address _prizeRecipient,
+    uint96 _claimReward,
+    address _claimRewardRecipient
+     */
+    prizePool.claimPrize(user1,1,0,user1,0,address(0));
+
+    // no withdrawing shutdown balance will revert due to the amount being withdrawn earlier via the claimPrize function
+    vm.prank(user1);
+    vm.expectRevert();
+    prizePool.withdrawShutdownBalance(address(this),user1);
+  }
+
   function testConstructor() public {
     assertEq(prizePool.firstDrawOpensAt(), firstDrawOpensAt);
     assertEq(prizePool.drawPeriodSeconds(), drawPeriodSeconds);

```

## Impact
User's can double claim assets when vault shutdowns eventually

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524

## Tool used
Manual Review

## Recommendation
Add a notShutdown modifier to the claimPrize function

# Issue M-18: `maxDeposit` doesn't comply with ERC-4626 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/134 

## Found by 
0x73696d616f, Tri-pathi, hash
## Summary
`maxDeposit` doesn't comply with ERC-4626 since depositing the returned amount can cause reverts

## Vulnerability Detail
The contract's `maxDeposit` function doesn't comply with ERC-4626 which is a mentioned requirement. 
According to the [specification](https://eips.ethereum.org/EIPS/eip-4626#maxdeposit), `MUST return the maximum amount of assets deposit would allow to be deposited for receiver and not cause a revert ....`  
The `deposit` function will revert in case the deposit is a lossy deposit ie. totalPreciseAsset function returns less than the totalDebt after the deposit. It is possible for this to occur due to rounding inside the preview redeem function of the yieldVault in the absence / depletion of yield buffer

### POC
Add the following test inside `pt-v5-vault/test/unit/PrizeVault/PrizeVault.t.sol`

```solidity
    function testHash_MaxDepositRevertLossy() public {
        PrizeVault testVault=  new PrizeVault(
            "PoolTogether aEthDAI Prize Token (PTaEthDAI)",
            "PTaEthDAI",
            yieldVault,
            PrizePool(address(prizePool)),
            claimer,
            address(this),
            YIELD_FEE_PERCENTAGE,
            0,
            address(this)
        );

        underlyingAsset.mint(address(yieldVault),100e18);
        YieldVault(address(yieldVault)).mint(address(100),99e18); // 99 shares , 100 assets

        uint maxDepositReturned = testVault.maxDeposit(address(this));

        uint amount_ = 99e18;
        underlyingAsset.mint(address(this),amount_);
        underlyingAsset.approve(address(testVault),amount_);

        assert(maxDepositReturned > amount_);

        vm.expectRevert();
        testVault.deposit(amount_,address(this));

    }
```

## Impact
Failure to comply with the specification which is a mentioned necessity

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L991-L992

## Tool used
Manual Review

## Recommendation
Consider the yieldBuffer balance too inside the `maxDeposit` function



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> deposit can only incur rounding issue if yield buffer is depleted but if the buffer is depleted reverts on deposit are expected__ see L112-114 of PrizeVault.sol



**nevillehuang**

Valid medium since it was mentioned as:

> Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
> PrizeVaults are expected to strictly comply with the ERC4626 standard.

Sherlock rules states
> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity

# Issue M-19: `maxRedeem` doesn't comply with ERC-4626 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/136 

## Found by 
hash
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



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> if `totalAssets` is 0 the first if statement will be executed thus no revert



**nevillehuang**

Valid medium since it was mentioned as:

> Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
> PrizeVaults are expected to strictly comply with the ERC4626 standard.

Sherlock rules states
> The protocol team can use the README (and only the README) to define language that indicates the codebase's restrictions and/or expected functionality. Issues that break these statements, irrespective of whether the impact is low/unknown, will be assigned Medium severity

# Issue M-20: User's might be able to add already errored witnet requests on L2's 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/154 

The protocol has acknowledged this issue.

## Found by 
hash
## Summary
User's might be able to add already errored witnet requests on L2's 

## Vulnerability Detail
`block.number` of L2's like Arbitrum return Ethereum's block number but [lagged](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#ethereum-block-numbers-within-arbitrum). Currently it is synced approximately every minute but is recommended to be only relied on a scale of hours. 
The current best time possible for witnet to report is 3 minutes (usually reported in 5-10 mins) and the wit/2 upgrade will (most likely) [decrease block time](https://discord.com/channels/492453503390842880/1154045538900115456/1247216363961843733). 
In such a case blocks could occur such that the rngRequest made at `block.number` is reported in the same block with `ERROR` as the status. 
Since no checks are made for the request status inside `startDraw` for the new request id, a user can add such a request and earn additional reward without any useful work

## Impact
User can claim reward without actually contributing to obtaining the random number

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L219-L225

## Tool used
Manual Review

## Recommendation
Check if the status is `ERROR` before adding a request



## Discussion

**sherlock-admin3**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> invalid__ I deployed a contract on arbitrum and the delay is at most 1 block from mainnet



**trmid**

Not sure I understand the exploit here, but it sounds like this block number check would prevent any RNG reuse if it's somehow returned in the same block number: https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L225

**nevillehuang**

@10xhash Could you take a look at the above comment? This seems invalid given you cannot re-use a random number within the same block. 

**10xhash**

It is not about re-use. If witnet returns in the same block.number and its status is `ERROR`, then that request could still be added because no-check is kept on the status of the request 

**nevillehuang**

@10xhash Would'nt this be OOS?

> The core protocol integration is the random number generator, Witnet. This is a trusted integration and you can assume they are always responsive.

Also is there a reference code logic I can take a look so I can determine the possibility of the status returning Error?

**10xhash**

`Error` is a possible state of the query in witnet

https://github.com/witnet/witnet-solidity-bridge/blob/4d1e464824bd7080072eb330e56faca8f89efb8f/contracts/data/WitnetOracleDataLib.sol#L69-L79

**trmid**

I don't see why this should be treated differently than the normal retry logic. If Witnet returns an Error response, then that request has failed and a new RNG request should be attempted if there is still time available to do so in the draw.

# Issue M-21: Users can setup hooks to control the expannsion of tiers 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/162 

The protocol has acknowledged this issue.

## Found by 
hash
## Summary
Reentrancy allows to not claim rewards and update tiers

## Vulnerability Detail
The fix for the [issue](https://code4rena.com/reports/2024-03-pooltogether#m-01-the-winner-can-steal-claimer-fees-and-force-him-to-pay-for-the-gas) still allows user's claim the rewards. But as mitigation, the contract will revert. But this still allows the user's to specifically revert on just the last canary tiers in order to always prevent the expansion of the tiers

## Impact
User's have control over the expansion of tiers

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L87

## Tool used
Manual Review

## Recommendation



## Discussion

**sherlock-admin4**

1 comment(s) were left on this issue during the judging contest.

**infect3d** commented:
> expected design



**nevillehuang**

request poc

**sherlock-admin4**

PoC requested from @10xhash

Requests remaining: **8**

**10xhash**

The issue that I wanted to point out is that since a user is not going to be benefited if a canary tier prize is claimed (the claiming bot would keep the fee as the entire prize amount. The canary prize claiming is kept inorder to determine if the number of tiers should be increased / decreased)  a user can revert in the hook if the tier is a canary tier. If it is misunderstood as a single user can disallow the entire tier expansion, it is not so

```diff
diff --git a/pt-v5-vault/test/unit/Claimable.t.sol b/pt-v5-vault/test/unit/Claimable.t.sol
index dfbb351..44b6568 100644
--- a/pt-v5-vault/test/unit/Claimable.t.sol
+++ b/pt-v5-vault/test/unit/Claimable.t.sol
@@ -131,6 +131,22 @@ contract ClaimableTest is Test, IPrizeHooks {
         assertEq(logs.length, 1);
     }
 
+
+
+    function testHash_RevertIfCanaryTiers() public {
+        PrizeHooks memory beforeHookOnly = PrizeHooks(true, false, hooks);
+
+        vm.startPrank(alice);
+        claimable.setHooks(beforeHookOnly);
+        vm.stopPrank();
+
+        mockClaimPrize(alice, 1, 2, prizeRedirectionAddress, 1e17, bob);
+
+        vm.expectRevert();
+        
+        claimable.claimPrize(alice, 1, 2, 1e17, bob);
+    }
+
     function testClaimPrize_afterClaimPrizeHook() public {
         PrizeHooks memory afterHookOnly = PrizeHooks(false, true, hooks);
 
@@ -259,6 +275,13 @@ contract ClaimableTest is Test, IPrizeHooks {
         uint96 reward,
         address rewardRecipient
     ) external returns (address, bytes memory) {
+        // prizePool.isCanarayTier(tier)
+        bool isCanaryTier = true;
+
+        if(isCanaryTier){
+            revert();
+        }
+
         if (useTooMuchGasBefore) {
             for (uint i = 0; i < 1000; i++) {
                 someBytes = keccak256(abi.encode(someBytes));
```

**trmid**

Coincidentally, about 2 weeks ago I started writing and sharing an article about how depositors can use this as a way to "vote" on prize sizes. Recently published [the article](https://pooltogether.mirror.xyz/-96mdzllmO9_9sUbd21eD4IEDJjtGZKLYOk7pgrenEI) showing how this can be used to democratize the prize algorithm.

> User's have control over the expansion of tiers

This is permissionless and powerful!

---

*One dev's bug is another dev's feature.*

# Issue M-22: Gas Manipulation by Malicious Winners in claimPrizes Function 

Source: https://github.com/sherlock-audit/2024-05-pooltogether-judging/issues/163 

## Found by 
0x73696d616f, 0xSpearmint1, MiloTruck, infect3d, jo13
## Summary

A malicious winner can exploit the `claimPrizes` function in the `Claimer` contract by reverting the transaction through returning a huge data chunk. This manipulation can cause the transaction to run out of gas, preventing legitimate claims and allowing the malicious user to claim prizes without computing winners.

## Vulnerability Detail

- The `Claimer` contract allows users to claim prizes on behalf of others by calling the `claimPrizes` function.
- A malicious winner can exploit this function by returning a huge data chunk when called, causing the transaction's gas to be too high and revert.
- Although the function catches the revert, the remaining gas (63/64 of the original gas) is likely insufficient for the rest of the claims.
- The malicious winner can then replay the transaction to claim the fees from the first claimer's computation without needing to compute the winners themselves.

## Impact

- Legitimate claimers may lose gas fees due to transaction reverts caused by malicious winners.
- Malicious winners can exploit this to claim prizes without computing winners, undermining the fairness of the prize distribution.

## Recommendation

- Implement a gas limit check to ensure that sufficient gas remains for the rest of the claims after catching a revert.
- Consider adding a mechanism to penalize or blacklist addresses that attempt to exploit this vulnerability.

