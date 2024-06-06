Small Khaki Sealion

medium

# Entire prize pool reserve is used up after a draw is awarded, preventing building up a larger reserve over time

## Summary

The `finishDraw` function caps the incentives with the `maxRewards` upper bound but donates the entire remaining reserve to the prize pool, which prevents building up a larger reserve over time.

## Vulnerability Detail

A portion of the vault contributions are captured as reserve, used to fund the incentives to award the draw (request a random number via [`DrawManager.startDraw`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L219) and submit the random number and award the draw via [`DrawManager.finishDraw`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L314)). The collected prize pool reserve is kept track of via the `TieredLiquidityDistributor._reserve` variable.

The eligible rewards (`availableRewards`) for starting and finishing the draw are calculated via the [`_computeAvailableRewards`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L505-L508) function in line [`333`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L333) of the `finishDraw` function.

```solidity
503: /// @notice Computes the available rewards for the auction (limited by max).
504: /// @return The amount of rewards available for the auction
505: function _computeAvailableRewards() internal view returns (uint256) {
506:   uint256 totalReserve = prizePool.reserve() + prizePool.pendingReserveContributions();
507:   return totalReserve > maxRewards ? maxRewards : totalReserve;
508: }
```

Those rewards are capped by the `maxRewards` upper bound to ensure that not the entire reserve is used.

[After the draw got rewarded](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L344), the remaining reserve (`remainingReserve`) is [donated to the prize pool](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L354-L369).

However, recalling that the rewards are capped by `maxRewards`, donating the remaining reserve causes the reserve to be used up **entirely**, which prevents building up a larger reserve over time to [act as a cushion when there is insufficient tier liquidity](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L374-L386).

## Impact

Whenever a draw is awarded, the entire prize pool reserve is used up (as incentives and donations), which prevents building up a larger reserve over time.

## Code Snippet

[DrawManager.finishDraw#L354-L369](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L354-L369)

```solidity
314: function finishDraw(address _rewardRecipient) external returns (uint24) {
...    // [...]
353:
354:   uint256 remainingReserve = prizePool.reserve();
355:
356:   emit DrawFinished(
357:     msg.sender,
358:     _rewardRecipient,
359:     drawId,
360:     _computeElapsedTime(startDrawAuction.closedAt, block.timestamp),
361:     _finishDrawReward,
362:     remainingReserve
363:   );
364:
365:   if (remainingReserve != 0 && vaultBeneficiary != address(0)) {
366:     _reward(address(this), remainingReserve);
367:     prizePool.withdrawRewards(address(prizePool), remainingReserve);
368:     prizePool.contributePrizeTokens(vaultBeneficiary, remainingReserve);
369:   }
```

## Tool used

Manual Review

## Recommendation

Consider calculating the remaining reserve as the difference between the `availableRewards` (which incorporates the `maxRewards` cap) and the rewards actually used as incentives.
