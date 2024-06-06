Shambolic Red Gerbil

medium

# `DrawManager.startDraw()` can be called after prize pool shutdown, even though `finishDraw()` always reverts

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