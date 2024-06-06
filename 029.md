Shambolic Red Gerbil

medium

# Undistributed prize liquidity due to `tierLiquidityUtilizationRate` remains stuck in the prize pool

## Summary

`PrizePool.awardDraw()` doesn't allocate unused prize liquidity to the reserve, causing it to remain stuck in the prize pool until it shuts down.

## Vulnerability Detail

In `TieredLiquidityDistributor._computePrizeSize()`, the size of each prize for all tiers is calculated as such:

[TieredLiquidityDistributor.sol#L424-L426](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L424-L426)

```solidity
    uint256 prizeSize = convert(
      convert(remainingTierLiquidity).mul(tierLiquidityUtilizationRate).div(convert(prizeCount))
    );
```

Where:
- `remainingTierLiquidity` is the amount of liquidity in a tier.
- `tierLiquidityUtilizationRate` is the [percentage of tier liquidity to distribute](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L112-L113), which is a fraction up to 100%.
- `prizeCount` is the number of prizes for a tier.

As seen from above, the total amount of liquidity used in a tier is `remainingTierLiquidity * tierLiquidityUtilizationRate`. This means that when `awardDraw()` is called, the amount of liquidity that remains unused is equal to the sum of `remainingTierLiquidity * (1 - tierLiquidityUtilizationRate)` for all tiers.

The issue is that in `awardDraw()`, this unused liquidity is not distributed anywhere. Funds can only leave the prize pool in two ways:

1. Funds are allocated to prizes and won by depositors.
2. Funds are added to the reserve and rewarded to draw bots.

Since the unused liquidity in all tiers is not allocated to (1) or (2), it will remain stuck in the prize pool. 

## Impact

For every draw, the unused liquidity in all tiers, equal to a percentage of `1 - tierLiquidityUtilizationRate`, remain stuck and never leave the prize pool, causing a loss of funds.

All unused liquidity can only be withdrawn after the prize pool shuts down. Assuming normal operation (ie. the number of unawarded draws in a row never exceeds `drawTimeout`), this will only occur after [the last observation in `TwabController`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-twab-controller/src/libraries/TwabLib.sol#L285-L290), which is ~136 years later.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L424-L426

## Tool used

Manual Review

## Recommendation

In `awardDraw()`, consider adding the unused liquidity to the reserve. 