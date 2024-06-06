Shambolic Red Gerbil

medium

# `TpdaLiquidationPair.swapExactAmountOut()` can be DOSed by a vault's mint limit

## Summary

By repeatedly DOSing `TpdaLiquidationPair.swapExactAmountOut()` for a period of time, an attacker can swap the liquidatable balancce in a vault for profit.

## Vulnerability Detail

When liquidation bots call `TpdaLiquidationPair.swapExactAmountOut()`, they specify the amount of tokens they wish to receive in `_amountOut`. `_amountOut` is then checked against the available balance to swap from the vault:

[TpdaLiquidationPair.sol#L141-L144](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L141-L144)

```solidity
        uint256 availableOut = _availableBalance();
        if (_amountOut > availableOut) {
            revert InsufficientBalance(_amountOut, availableOut);
        }
```

The available balance to swap is determined by the liquidatable balance of the vault:
 
[TpdaLiquidationPair.sol#L184-L186](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L184-L186)

```solidity
    function _availableBalance() internal returns (uint256) {
        return ((1e18 - smoothingFactor) * source.liquidatableBalanceOf(address(_tokenOut))) / 1e18;
    }
```

However, when the output token from the swap (ie. `tokenOut`) is vault shares, `PrizeVault.liquidatableBalanceOf()` is restricted by the mint limit:

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

This means that if the amount of shares minted is close to `type(uint96).max`, the available balance in the vault (ie. `_liquidYield`) will be restricted by the remaining number of shares that can be minted.

However, an attacker can take advantage of this to force all calls to `TpdaLiquidationPair.swapExactAmountOut()` to revert:

- Assume a vault has the following state:
  - The amount of `_availableYield` in the vault is `1000e18`.
  - The amount of shares currently minted is `type(uint96).max - 2000e18`, so `_liquidYield` is not restricted by the mint limit.
  - `yieldFeePercentage = 0` and `smoothingFactor = 0`.
- A liquidation bot calls `swapExactAmountOut()` with `_amountOut = 1000e18`.
- An attacker front-runs the liquidation bot's transaction and deposits `1000e18 + 1` tokens, which mints the same amount of shares:
  - The amount of shares minted is now `type(uint96).max - 1000e18 + 1`, which means the mint limit is `1000e18 - 1`.
  - As such, the available balance in the vault is reduced to `1000e18 - 1`.
- The liquidation bot's transaction is now executed:
  - In `swapExactAmountOut()`, `_amountOut > availableOut` so the call reverts.

Note that the `type(uint96).max` mint limit is reachable for tokens with low value. For example, PEPE has 18 decimals and a current price of $0.00001468, so `type(uint96).max` is equal to $1,163,070 worth of PEPE. For tokens with a higher value, the attacker can borrow funds in the front-run transaction, and back-run the victim's transaction to return the funds.

This is an issue as the price paid by liquidation bots for the liquidatable balance decreases linearly over time:

[TpdaLiquidationPair.sol#L191-L195](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L191-L195)

```solidity
        uint256 elapsedTime = block.timestamp - lastAuctionAt;
        if (elapsedTime == 0) {
            return type(uint192).max;
        }
        uint192 price = uint192((targetAuctionPeriod * lastAuctionPrice) / elapsedTime);
```

As such, an attacker can repeatedly perform this attack (or deposit sufficient funds until the mint limit is 0) to prevent any liquidation bot from swapping the liquidatable balance. After the price has decreased sufficiently, the attacker can then swap the liquidatable balance for himself at a profit.

## Impact

By depositing funds into a vault to reach the mint limit, an attacker can DOS all calls to `TpdaLiquidationPair.swapExactAmountOut()` and prevent liquidation bots from swapping the vault's liquidatable balance. This allows the attacker to purchase the liquidatable balance at a discount, which causes a loss of funds for users in the vault.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L141-L144

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L184-L186

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L687-L709

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L191-L195

## Tool used

Manual Review

## Recommendation

Consider implementing `liquidatableBalanceOf()` and/or `_availableBalance()` such that it is not restricted by the vault's mint limit. 

For example, consider adding a `_tokenOut` parameter to `TpdaLiquidationPair.swapExactAmountOut()` for callers to specify the output token. This would allow liquidation bots to swap for the vault's asset token, which is not restricted by the mint limit, when the mint limit is reached.