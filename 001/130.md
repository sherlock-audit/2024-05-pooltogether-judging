Small Khaki Sealion

medium

# A new draw auction for the same draw cannot be started if the RNG request succeeded but the `finishDraw` auction expired

## Summary

If the `finishDraw` auction expires while the RNG request succeeded, a new draw auction for the same draw cannot be started, resulting in the current draw not being rewarded.

## Vulnerability Detail

Multiple RNG request (upper limit is `maxRetries`) attempts are possible for the same draw if such a request fails. The two-step process consists of starting a draw action and requesting a random number, and once the RNG request succeeded, `finishDraw` is called to retrieve the random number and provide it to the prize pool. Both the `startDraw` and `finishDraw` auctions are time-restricted and have to be retried if the elapsed auction time exceeds the configured duration.

However, there is a specific scenario where it is not possible to start a new draw auction:

`startDraw` has been called for a new draw, and the RNG request has been sent to the randomness provider contract. But the `finishDraw` call for this specific (completed) RNG request is too late, i.e., [the auction expired](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L329-L331). As a result, it is not possible to call the `startDraw` function for the same draw again, because the previous RNG request succeeded and thus it [reverts with `AlreadyStartedDraw`](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L238-L240). Consequently, the current draw can not be rewarded.

## Impact

The current draw can not be rewarded, and it must be waited for the next draw to be rewarded.

## Code Snippet

[DrawManager.startDraw(..)](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-draw-manager/src/DrawManager.sol#L238-L240)

```solidity
219: function startDraw(address _rewardRecipient, uint32 _rngRequestId) external returns (uint24) {
220:
221:   if (_rewardRecipient == address(0)) revert RewardRecipientIsZero();
222:   uint24 drawId = prizePool.getDrawIdToAward();
223:   uint48 closesAt = prizePool.drawClosesAt(drawId);
224:   if (closesAt > block.timestamp) revert DrawHasNotClosed();
225:   if (rng.requestedAtBlock(_rngRequestId) != block.number) revert RngRequestNotInSameBlock();
226:
227:   StartDrawAuction memory lastRequest = getLastStartDrawAuction();
228:   uint256 auctionOpenedAt;
229:
230:   if (lastRequest.drawId != drawId) { // if this request is for a new draw
231:     // auctioned opened at the close of the draw
232:     auctionOpenedAt = closesAt;
233:     // clear out the old ones
234:     while (_startDrawAuctions.length > 0) {
235:       _startDrawAuctions.pop();
236:     }
237:   } else { // the old request is for the same draw
238: ❌  if (!rng.isRequestFailed(lastRequest.rngRequestId)) {
239: ❌    revert AlreadyStartedDraw();
240:     } else if (_startDrawAuctions.length > maxRetries) { // if request has failed and we have retried too many times
241:       revert RetryLimitReached();
242:     } else if (block.number == rng.requestedAtBlock(lastRequest.rngRequestId)) { // requests cannot be reused
243:       revert StaleRngRequest();
244:     } else {
245:       // auctioned opened at the close of the last auction
246:       auctionOpenedAt = lastRequest.closedAt;
247:     }
248:   }
```

## Tool used

Manual Review

## Recommendation

Consider allowing the `startDraw` function to be called for the same draw again if the RNG request has succeeded but the `finishDraw` auction has expired.
