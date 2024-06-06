Slow Licorice Opossum

high

# Attackers can create invalid observations by contributing 0 PrizeToken, causing getDisbursedBetween to return incorrect values.

## Summary

Attackers can create invalid observations by contributing 0 PrizeToken, causing getDisbursedBetween to return incorrect values.

## Vulnerability Detail

Function contributePrizeTokens allows user to contribute 0 PrizeToken. When it’s the first contribution, an observation is created with both available and disbursed being 0. However, the cardinality will be increased by 1.

As for function getDisbursedBetween, when looking for `atOrAfterStart`, the observation corresponding to _startDrawId is obtained by default first. However, when the available and disbursed are all 0, It will use binary search and `atOrAfterStart` will be `_accumulator.observations[afterOrAtDrawId]`.

```solidity
    Observation memory atOrAfterStart;
    if (_startDrawId <= oldestDrawId || ringBufferInfo.cardinality == 1) {
      atOrAfterStart = _accumulator.observations[oldestDrawId];
    } else {
      // check if the start draw has an observation, otherwise search for the earliest observation after
      atOrAfterStart = _accumulator.observations[_startDrawId];
      if (atOrAfterStart.available == 0 && atOrAfterStart.disbursed == 0) {
        (, , , uint24 afterOrAtDrawId) = binarySearch(
          _accumulator.drawRingBuffer,
          oldestIndex,
          newestIndex,
          ringBufferInfo.cardinality,
          _startDrawId
        );
        atOrAfterStart = _accumulator.observations[afterOrAtDrawId];
      }
    }
```

There is a problem here. For example, when _startDrawId is equal to 1, the observation directly obtained is the result of the attacker making 0 contribution.

```solidity
atOrAfterStart = _accumulator.observations[_startDrawId];
```

Therefore, a binary search is performed, and the two DrawId obtained are 1 and 2. But here the latter (DrawId=2) will be directly selected as the result.

```solidity
(, , , uint24 afterOrAtDrawId) = binarySearch(
...
atOrAfterStart = _accumulator.observations[afterOrAtDrawId];
```

Finally, the function getDisbursedBetween calculates a smaller result than expected.

## Impact

Function getDisbursedBetween calculates a smaller result than expected, causing some prizeTokens are locked.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L412-L422

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L176-L192

## Tool used

Manual Review

## Recommendation

Disable contributing 0 prizetokens.