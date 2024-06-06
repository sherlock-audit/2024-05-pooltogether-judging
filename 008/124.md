Crazy Linen Walrus

medium

# `Claimers` can receive less `feePerClaim` than they should if some prizes are already claimed or if reverts because of a reverting hook

## Summary

If a claimer propose an array of prizes to claim, but some of these prizes have already been claimed, or some claims revert, then the actual `feePerClaim` received will be less **compared to what it should really be, as expected by the VRGDA algorithm**

This happens because `_computeFeePerClaim` can undervaluate the value of `feePerClaim`, as it compute failed claims as successful ones.

This poses an issue, as `feePerClaim` is then used as an input by `_vault.claimPrize(_winners[w], _tier, _prizeIndices[w][p], _feePerClaim, _feeRecipient)`, which will transfer the undervaluated fee to the claimer.


## Vulnerability Detail
Auctions for claiming prizes are based on the [VRGDA algorithm](https://www.paradigm.xyz/2022/08/vrgda)
In simple terms, this algorithm update price depending on either the numbers of claim is behind or ahead of time schedule.
In order to have a schedule, a target of claim per time unit is defined.
Just to give an idea, let's simplify that to the extreme (we will see complete formula afterward) and say : `price(t) = claim/expected * targetPrice(t)` 
E.g: if 10 claims per 2 hours are exepected, then at t=1h, 5 claims should be concluded.
If only 4 were claimed, then we can calculate that (4 claims)/(5 expected) < 1, price will be lower that target.
if 6 were claimed, then we will have (6 claims)/(5 expected) > 1, price will be greater than target.

The formula that has been implemented into `LinearVRGDALib` is the following:

```haskell
price = p0 * e^ (k * (t - n+1/r))   ; with k = ln(maxFee/minFee) * t_target
```

With:
- `n` the number of claim already completed
- `r` the expected rate per hour
- `k` the decay constant (speed at which price will change)
- `p0` the target price (or fee in our case)

The more `k *(t - n+1/r) > 0`, the more `price > p0`
When `t = n+1/r` <=> `(k * (t - n+1/r)) = 0`, then `price = p0`
The more `k * (t - n+1/r) < 0`, the more `price < p0`

We understand that the more whe are behind schedule in term of expected claim, the higher the fees earned by claimer will be.
And the more we are ahead of schedule, the lower the fee for claimers will be (as their is no urgency)

### Scenario
Now let's see what happens in this scenario:
1. For now, 0 prizes have already been claimed from the prize pool, `n = 0`
2. Alice has 5 prizes to claim, she build her claim array
3. It seems that 2 of the prizes Alice was going to claim are claimed right before, now `n = 2`
4. Alice tx is executed with the 5 prizes

Now let's see what happen from a code perspective.
The code above is the entry point for claiming prizes. As we can see L113, the `feePerClaim` is computed based on the [number of claims to count](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L126-L136) (`_countClaims`) and the number of [already claimed prizes](https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-prize-pool/src/PrizePool.sol#L302-L302) (`prizePool::claimCount()`)
Then, the computed `feePerClaim` value is given to `_claim` which actually claim the prizes into the Prize Pool.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
```solidity
File: pt-v5-claimer/src/Claimer.sol

090:   function claimPrizes(
091:     IClaimable _vault,
092:     uint8 _tier,
093:     address[] calldata _winners,
094:     uint32[][] calldata _prizeIndices,
095:     address _feeRecipient,
096:     uint256 _minFeePerClaim
097:   ) external returns (uint256 totalFees) {
...:
...:          /* some code */
...:
112:     if (!feeRecipientZeroAddress) {
113:       feePerClaim = SafeCast.toUint96(_computeFeePerClaim(_tier, _countClaims(_winners, _prizeIndices), prizePool.claimCount()));
114:       if (feePerClaim < _minFeePerClaim) {
115:         revert VrgdaClaimFeeBelowMin(_minFeePerClaim, feePerClaim);
116:       }
117:     }
118:
119:     return feePerClaim * _claim(_vault, _tier, _winners, _prizeIndices, _feeRecipient, feePerClaim); 
120:   }
```

Now, let's see how `_computeFeePerClaim` actually compute `feePerClaim`.
We see above L230-241 that a fee is calculated for each of the claims of the array, starting at `_claimedCount` (The number of prizes already claimed) based on the VRGDA formula L309. The returned value (which is stored into `feePerClaim`) is the averaged fee as shown L241.
And as we explained earlier, the higher the number of claim we make, the lower the earned fee are. So, a higher value of `_claimedCount + i` will give lower fees.

https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236
```solidity
File: pt-v5-claimer/src/Claimer.sol
205:   /// @param _claimCount The number of claims to check
206:   /// @param _claimedCount The number of prizes already claimed
207:   /// @return The total fees for the claims
208:   function _computeFeePerClaim(
209:     uint8 _tier,
210:     uint256 _claimCount,
211:     uint256 _claimedCount
212:   ) internal view returns (uint256) {
...:
...:        /* some code */
...:
227:     uint256 elapsed = block.timestamp - (prizePool.lastAwardedDrawAwardedAt());
228:     uint256 fee;
229: 
230:     for (uint256 i = 0; i < _claimCount; i++) {
231:       fee += _computeFeeForNextClaim(
232:         targetFee,
233:         decayConstant,
234:         perTimeUnit,
235:         elapsed,
236:         _claimedCount + i,         
237:         _maxFee
238:       );
239:     }
240: 
241:     return fee / _claimCount;
242:   }
...:
...:        /* some code */
...:
301:   function _computeFeeForNextClaim(
302:     uint256 _targetFee,
303:     SD59x18 _decayConstant,
304:     SD59x18 _perTimeUnit,
305:     uint256 _elapsed,
306:     uint256 _sold,
307:     uint256 _maxFee
308:   ) internal pure returns (uint256) {
309:     uint256 fee = LinearVRGDALib.getVRGDAPrice(
310:       _targetFee,
311:       _elapsed,
312:       _sold,
313:       _perTimeUnit,
314:       _decayConstant
315:     );
316:     return fee > _maxFee ? _maxFee : fee;
317:   }
318: 
```

What we can see from this, is that the computation will be executed for `_claimCount = 2` and up to `i = 5`, so as if there has been 7 claimed prizes, while in reality only 5 prizes are claimed, leading in an undervaluation of the fees to award.
As you probably have infered, the computation should have been made for `i = 3` to be correct.

## Impact
The `feePerClaim` computation is incorrect as the VRGDA is calculated for more claims that will really happen, leading to less fee earned by claimers at the time of the call.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L113
https://github.com/sherlock-audit/2024-05-pooltogether//blob/main/pt-v5-claimer/src/Claimer.sol#L236

## Tool used
Manual Review

## Recommendation
The `PrizePool` contract expose a function to check if a prize has already been claimed: `wasClaimed`
This can be used to countClaims based on the actual true number of claimable prizes from the array.

This isn't a "perfect" solution though, as there are still issues when not already claimed prizes revert because of reverting prize hooks. In that case, VRGDA will still count the claim as happening, but we can consider this less likely to happen.

```diff
  function _countClaims(
    address[] calldata _winners,
    uint32[][] calldata _prizeIndices
  ) internal pure returns (uint256) {
    uint256 claimCount;
    uint256 length = _winners.length;
    for (uint256 i = 0; i < length; i++) {
-     claimCount += _prizeIndices[i].length;
+	  numPrize = _prizeIndices[i].length;
+	  for(uint256 j = 0; j < numPrize; j++) {
+     	bool wasClaimed = wasClaimed(_vault, _winner, _drawId,_tier, _prizeIndex);
+     	if(!wasClaimed) {
+		 claimCount += 1;
+		}
+     }
    }
    return claimCount;
  }
```