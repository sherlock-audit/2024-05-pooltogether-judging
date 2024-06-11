Small Khaki Sealion

medium

# The RNG finish draw auction rewards are overpaid due to missing to account for the time it takes to fulfill the Witnet randomness request

## Summary

The rewards for finishing the draw and submitting the previously requested randomness result are slightly overpaid due to the incorrect calculation of the elapsed auction time, wrongly including the time it takes to fulfill the Witnet randomness request.

## Vulnerability Detail

The `DrawManager.finishDraw` function calculates the rewards for finishing the draw, i.e., providing the previously requested randomness result, via the `_computeFinishDrawReward` function in lines [`338-342`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L338-L342). The calculation is based on the difference between the timestamp when the start draw auction was closed (`startDrawAuction.closedAt`) and the current block timestamp. Specifically, a parabolic fractional dutch auction (PFDA) is used to incentivize shorter auction durations, resulting in faster randomness submissions.

However, the requested randomness result at block X is not immediately available to be used in the `finishDraw` function. According to the Witnet documentation, [the randomness result is available after 5-10 minutes](https://docs.witnet.io/smart-contracts/witnet-randomness-oracle/code-examples). This time delay is currently included when determining the elapsed auction time because `startDrawAuction.closedAt` marks the time when the randomness request was made, not when the result was available to be submitted.

Consequently, the protocol **always** overpays rewards for the finish draw. Over the course of many draws, this can lead to a significant overpayment of rewards.

It should be noted that the PoolTogether documentation about the draw auction timing [clearly outlines the timeline for the randomness auction](https://dev.pooltogether.com/protocol/design/draw-auction/#auction-timing) and states that finish draw auction price will rise **once the random number is available**:

> _The RNG auction for any given draw starts at the beginning of the following draw period and must be completed within the auction duration. Following the start RNG auction, there is an “unknown” waiting period while the RNG service fulfills the request._
>
> _**Once the random number is available**, the finished RNG auction will rise in price until it is called to award the draw with the available random number. Once the draw is awarded for a prize pool, it will distribute the fractional reserve portions based on the auction results that contributed to the closing of that draw._

## Impact

The protocol overpays rewards for the finish draw.

## Code Snippet

[pt-v5-draw-manager/src/DrawManager.sol#L339](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L339)

```solidity
311: /// @notice Called to award the prize pool and pay out rewards.
312: /// @param _rewardRecipient The recipient of the finish draw reward.
313: /// @return The awarded draw ID
314: function finishDraw(address _rewardRecipient) external returns (uint24) {
315:   if (_rewardRecipient == address(0)) {
316:     revert RewardRecipientIsZero();
317:   }
318:
319:   StartDrawAuction memory startDrawAuction = getLastStartDrawAuction();
...    // [...]
338:   (uint256 _finishDrawReward, UD2x18 finishFraction) = _computeFinishDrawReward(
339: ❌  startDrawAuction.closedAt,
340:     block.timestamp,
341:     availableRewards
342:   );
343:   uint256 randomNumber = rng.randomNumber(startDrawAuction.rngRequestId);
```

## Tool used

Manual Review

## Recommendation

Consider determining the auction start for the finish draw reward calculation based on the timestamp when the randomness result was made available. This timestamp is accessible by using the [`WitnetRandomnessV2.fetchRandomnessAfterProof`](https://github.com/witnet/witnet-solidity-bridge/blob/4d1e464824bd7080072eb330e56faca8f89efb8f/contracts/apps/WitnetRandomnessV2.sol#L196) function ([instead of `fetchRandomnessAfter`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-rng-witnet/src/RngWitnet.sol#L124)). The `_witnetResultTimestamp` return value can be used to better approximate the auction start, hence, paying out rewards more accurately.
