Swift Ruby Porcupine

medium

# The swap mechanism does not have a deadline parameter

## Summary
The swap mechanism does not have a deadline parameter

## Vulnerability Detail
`swapExactAmountOut()` in `TpdaLiquidationPair.sol` is missing the `deadline` parameter

```js
    function swapExactAmountOut(
        address _receiver,
        uint256 _amountOut,
        uint256 _amountInMax,
        bytes calldata _flashSwapData
    ) external returns (uint256) {
```
The deadline check ensure that the transaction can be executed on time and the expired transaction revert.

## Impact
The transaction can be pending in mempool for a long time and the trading activity is very time sensitive. Without deadline check, the trade transaction can be executed in a long time after the user submit the transaction, at that time, the trade can be done in a sub-optimal price, which harms user's position.

The deadline check ensure that the transaction can be executed on time and the expired transaction revert.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L122-L167

## Tool used
Manual Review

## Recommendation
Add a deadline timestamp parameter to the `swapExactAmountOut()`  method and revert the transaction if the expiry has passed.