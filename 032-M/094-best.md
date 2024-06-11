Festive Bone Cobra

medium

# Some tiers will not be awarded in the PrizePool when the last observation has been erased in the TwabController

## Summary

Tiers are awarded based on the past twab supplies registered in the `TwabController`, but it reverts if an observation that has been erased due to the circular buffer is tried to fetch. Thus, some tiers will not be claimed.

## Vulnerability Detail

In `PrizePool::isWinner()`, the `twab` balances are fetched from the `TwabController` since [startDrawIdInclusive](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1014). This variable is calculated using [PrizePool::computeRangeStartDrawIdInclusive()](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1058), where `_endDrawIdInclusive` is the `lastAwardedDrawId_` and `_rangeSize` is an estimate of the prize frequency in draws, basically `1 / odds` rounded up.

To illustrate the issue, consider the grand prize, where odds are `1 / g = 0.00273972602e18` with `g == 365 days`. The estimated prize frequency is `1e18 / 0.00273972602e18 == 365.000000985` rounded up to the nearest integer, `366`. If `lastAwardedDrawId_` is `366`, then `startDrawIdInclusive` is `366 - 366 + 1 == 1`. This means that it will fetch the `twab` balance from the `TwabController` since the opening of `drawId == 1`.

The last awarded draw in the example is `lastAwardedDrawId_ == 366`, which means that at least `366` draw periods have passed since `drawId == 1`. If `drawPeriodSeconds` is `2 days`, this is 732 days in the past. The `TwabController` allows a minimum `PERIOD_LENGTH` of `1 hour`, and can store up to `17520` observations, which is `17520 / 24 == 730` days. It stores the observations in a circular buffer, which means that past observations are overwritten after the buffer has been filled.

The `PrizeVault` tries to fetch an observation that no longer exists, `startDrawIdInclusive == 1`, as `lastAwardedDrawId_ == 366`, so the current `drawId` is at least `367`, which means that it has already overriden index `0` of the ring buffer. In this case, on `DrawAccumulatorLib::getDisbursedBetween()`, it fetches the [oldest](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L178) possible index, which is index `1`. Thus, the `PrizeVault` is not problematic in this scenario.

However, the `TwabController` [reverts](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-twab-controller/src/libraries/TwabLib.sol#L648-L651) if an observation older than the oldest in the ring buffer is tried to fetch, as is the case here due to the fact that it can only store `730` days, which means that it would be impossible to claim the prize.

This will also happen when users are late to claim their shutdown portions, as `twab` values in the `TwabController` can keep being registered after the `PrizePool` has shutdown, so the timestamp corresponding to `startDrawIdInclusive` in the `TwabController` may have been erased (after shutdown values can not be added to the accumulators in the `PrizePool`).

## Impact

Impossibility to claim prizes when the `startDrawIdInclusive` is older than the oldest observation in the `TwabController`. Also, impossible to claim the shutdown portion if the `TwabController` has been overwritten enough times after the shutdown.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-twab-controller/src/libraries/TwabLib.sol#L648-L651
```solidity
function _getPreviousOrAtObservation(
  ObservationLib.Observation[MAX_CARDINALITY] storage _observations,
  AccountDetails memory _accountDetails,
  PeriodOffsetRelativeTimestamp _offsetTargetTime
) private view returns (ObservationLib.Observation memory prevOrAtObservation) {
  ...
  if (PeriodOffsetRelativeTimestamp.unwrap(_offsetTargetTime) < prevOrAtObservation.timestamp) {
    // if the user didn't have any activity prior to the oldest observation, then we know they had a zero balance
    if (_accountDetails.cardinality < MAX_CARDINALITY) {
      ...
    } else {
      // if we are missing their history, we must revert
      revert InsufficientHistory(
        _offsetTargetTime,
        PeriodOffsetRelativeTimestamp.wrap(prevOrAtObservation.timestamp)
      );
    }
  }
  ...
}
```

## Tool used

Manual Review

Vscode

## Recommendation

Instead of reverting in the `TwabController`, use the oldest possible observation, which is the best estimate in this case that still allows claiming prizes.