Shambolic Red Gerbil

high

# Inconsistent result from `DrawAccumulator.binarySearch()` causes prize pool calculations to be incorrect

## Summary

The result returned from `DrawAccumulator.binarySearch()` is not consistent, causing `DrawAccumulator.getDisbursedBetween()` to wrongly exclude the start and/or end draw ID under certain conditions. This causes all prize calculation in the prize pool to be incorrect.

## Vulnerability Detail

In `DrawAccumulator.binarySearch()`, the terminating condition (ie. the condition at which the binary search stops) is as shown:

[DrawAccumulatorLib.sol#L257-L262](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L257-L262)

```solidity
      bool targetAtOrAfter = beforeOrAtDrawId <= _targetDrawId;

      // Check if we've found the corresponding Observation.
      if (targetAtOrAfter && _targetDrawId <= afterOrAtDrawId) {
        break;
      }
```

As seen from above, the terminating condition is `beforeOrAtDrawId <= targetId <= afterOrAtDrawId`. However, when `targetId` is in the draw ring buffer, the result returned by `binarySearch()` may be inconsistent. For example:

- Assume that:
  - The current draw ring buffer contains `[1, 2, 3, ...]`.
  - `targetId = 2`.
- `binarySearch()` could return either:
  - `beforeOrAtDrawId = 1` and `afterOrAtDrawId = 2` as `beforeOrAtDrawId < targetId = afterOrAtDrawId`.
  - `beforeOrAtDrawId = 2` and `afterOrAtDrawId = 3` as `beforeOrAtDrawId = targetId < afterOrAtDrawId`.

The result returned depends on the length of the draw ring buffer, more specifically, the difference between `_oldestIndex` and `_newestIndex`. This is demonstrated in the POC below.

`binarySearch()` is used in `DrawAccumulator.getDisbursedBetween()` to find the total balance between a start and end draw, inclusive of both draws:

[DrawAccumulatorLib.sol#L183-L190](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L183-L190)

```solidity
        (, , , uint24 afterOrAtDrawId) = binarySearch(
          _accumulator.drawRingBuffer,
          oldestIndex,
          newestIndex,
          ringBufferInfo.cardinality,
          _startDrawId
        );
        atOrAfterStart = _accumulator.observations[afterOrAtDrawId];
```

Since `afterOrAtDrawId` is taken as the start draw without checking if `beforeOrAtDrawId` is equal to `_startDrawId`, it is possible for `afterOrAtDrawId` to be greater than `_startDrawId` when `_startDrawId` is present in `drawRingBuffer`. For example:

- `_startDrawId = 2`.
- `binarySearch()` returns `beforeOrAtDrawId = 2` and `afterOrAtDrawId = 3` .
- The logic above takes `afterOrAtDrawId = 3` as the starting draw.
- However, since the result from `getDisbursedBetween()` is inclusive of `_startDrawId`, `beforeOrAtDrawId` should be taken as the starting draw instead.

Similarly, the ending draw returned could wrongly exclude `_endDrawId` even when it is present in `drawRingBuffer`, since `beforeOrAtDrawId` is always used:

[DrawAccumulatorLib.sol#L201-L208](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L201-L208)

```solidity
        (, uint24 beforeOrAtDrawId, , ) = binarySearch(
          _accumulator.drawRingBuffer,
          oldestIndex,
          newestIndex,
          ringBufferInfo.cardinality,
          _endDrawId
        );
        atOrBeforeEnd = _accumulator.observations[beforeOrAtDrawId];
```

As a result, `getDisbursedBetween()` could return a balance far smaller than the actual disbursed balance between `_startDrawId` and `_endDrawId`.

The following POC demonstrates how the results returned by `binarySearch()` are inconsistent:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.19;

import "forge-std/Test.sol";
import "src/libraries/DrawAccumulatorLib.sol";

contract BinarySearchTest is Test { 
    uint24[MAX_OBSERVATION_CARDINALITY] drawRingBuffer;

    function testBinarySearchInconsistent() public {
        // Buffer of length 3
        drawRingBuffer[0] = 1;
        drawRingBuffer[1] = 2;
        drawRingBuffer[2] = 3;
        
        // BInary search returns (2, 3)
        (, uint24 beforeOrAtDrawId,, uint24 afterOrAtDrawId) = DrawAccumulatorLib.binarySearch(
            drawRingBuffer, 
            0, // _oldestIndex
            2, // _newestIndex
            MAX_OBSERVATION_CARDINALITY, 
            2  // _targetDrawId
        );
        console2.log(beforeOrAtDrawId, afterOrAtDrawId);
        
        // Buffer of length 5
        drawRingBuffer[3] = 4;
        drawRingBuffer[4] = 5;

        // Binary search returns (1, 2)
        (, beforeOrAtDrawId,, afterOrAtDrawId) = DrawAccumulatorLib.binarySearch(
            drawRingBuffer, 
            0, // _oldestIndex
            4, // _newestIndex
            MAX_OBSERVATION_CARDINALITY, 
            2  // _targetDrawId
        );
        console2.log(beforeOrAtDrawId, afterOrAtDrawId);
    }
}
```

## Impact

`DrawAccumulatorLib.getDisbursedBetween()` could return a balance much smaller than the actual balance between two draws. It is used in many crucial calculations of the prize pool:
- Used in [`PrizePool.awardDraw()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L479) to find the total amount of new prize liquidity to distribute.
- Used in [`PrizePool._getVaultShares()`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L1119-L1137) to find the amount of liquidity contributed by a vault and the total amount of liquidity between two draws.

Due to the bug in `getDisbursedBetween()`, all calculations in the prize pool involving the allocation/distribution of prize liquidity will be incorrect, causing a loss of funds as users will receive incorrect prize amounts.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L257-L262

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L183-L190

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L201-L208

## Tool used

Manual Review

## Recommendation

When finding the start draw in `binarySearch()`, consider checking if `beforeOrAtDrawId` equals to `_startDrawId` before using `afterOrAtDrawId`:

```diff
- (, , , uint24 afterOrAtDrawId) = binarySearch(
+ (, uint24 beforeOrAtDrawId, , uint24 afterOrAtDrawId) = binarySearch(
    _accumulator.drawRingBuffer,
    oldestIndex,
    newestIndex,
    ringBufferInfo.cardinality,
    _startDrawId
  );
- atOrAfterStart = _accumulator.observations[afterOrAtDrawId];
+ atOrAfterStart = _accumulator.observations[
+   beforeOrAtDrawId == _startDrawId ? beforeOrAtDrawId : afterOrAtDrawId
+ ];
```

Similarly, when finding the end draw, check if `afterOrAtDrawId` equals to `_endDrawId` before using `beforeOrAtDrawId`:

```diff
- (, uint24 beforeOrAtDrawId, , ) = binarySearch(
+ (, uint24 beforeOrAtDrawId, , uint24 afterOrAtDrawId) = binarySearch(
    _accumulator.drawRingBuffer,
    oldestIndex,
    newestIndex,
    ringBufferInfo.cardinality,
    _endDrawId
  );
- atOrBeforeEnd = _accumulator.observations[beforeOrAtDrawId];
+ atOrBeforeEnd = _accumulator.observations[
+   afterOrAtDrawId == _endDrawId ? afterOrAtDrawId : beforeOrAtDrawId
+ ];
```