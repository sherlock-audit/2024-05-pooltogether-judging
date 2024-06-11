Shambolic Red Gerbil

high

# `drawTimeoutAt()` causes the prize pool to shutdown one draw earlier

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