Slow Licorice Opossum

medium

# `getDisbursedBetween` may return incorrect values when `ringBufferInfo.cardinality = 1`

## Summary

`getDisbursedBetween` may return incorrect values when `ringBufferInfo.cardinality = 1`

## Vulnerability Detail

`getDisbursedBetween` is used to get the balance that was disbursed between the given start and end draw ids, inclusive. However, the following code snippet may result in incorrect calculation results.

```solidity
if (_endDrawId >= _newestDrawId || ringBufferInfo.cardinality == 1) {
  atOrBeforeEnd = _accumulator.observations[_newestDrawId];
} else {
```

This code means that when `_endDrawId < _newestDrawId` and `ringBufferInfo.cardinality = 1`, the newest observation will be obtained, which is logically wrong.

For example, firstDrawOpensAt = 100, drawPeriodSeconds = 10, _lastAwardedDrawId = 0.

When block.timestamp = 111, the `contributePrizeTokens` is called for the first time. That is, the first observation id will be 2 (openDrawId = (111 - 100)/10 + 1). 

Then, when block.timestamp = 115, function `awardDraw` is called, the awardingDrawId will be 1. Therefore, it will calculate 

```solidity
getTotalContributedBetween(1, 1)
```

Since no contribution was made in the first period, it should return 0. However, in the function getDisbursedBetween, it takes the newest observation (id = 2) as end observation, causing incorrect calculation.

Note that this issue also applies to the following code snippet:

```solidity
if (_startDrawId <= oldestDrawId || ringBufferInfo.cardinality == 1) {
  atOrAfterStart = _accumulator.observations[oldestDrawId];
} else {
```

## Impact

Function getDisbursedBetween will return wrong values.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L177-L179

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/DrawAccumulatorLib.sol#L195-L197

## Tool used

Manual Review

## Recommendation

```solidity
if (_endDrawId >= _newestDrawId) {
  atOrBeforeEnd = _accumulator.observations[_newestDrawId];
} else if (_endDrawId < _newestDrawId && ringBufferInfo.cardinality == 1) {
	atOrBeforeEnd = _accumulator.observations[oldestDrawId];
}
```