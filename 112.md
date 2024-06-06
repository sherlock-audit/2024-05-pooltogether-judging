Icy Coal Mammoth

high

# When `PrizePool` reserves are low and more than expected number of prizes are won, race conditions are created to claim prizes. Some users' prize funds will be locked due to `PrizePool::claimPrize()` call reverting.

### When `PrizePool` reserves are low and more than expected number of prizes are won, race conditions are created to claim prizes. Some users' prize funds will be locked due to `PrizePool::claimPrize()` call reverting

## Summary

The determination of winners for a given draw and tier are statistically independant events. That is, if Alice winning a prize for a given draw and tier does not make it more or less likely that Bob will win a prize in the same draw and tier. Consequently, there may be more (or less) than expected winners in any given draw. 
This can be an issue because the `tierLiquidity.prizeSize` for a given tier and draw is based on the expected number of winners, not the actual number of winners. In the case where there is more than the expected number of winners there may not be enough liquidity in the reserve to cover the extra prize(s). 
Under these circumstances, there is a race for users' to call `PrizePool::claimPrize()` before the liqudiity in the tier runs out. Once tier liquidity runs out, there will be user(s) who are geneuine winners for a given draw and tier (`PrizePool::isWinner()` returns true), but calls to `PrizePool::claimPrize()` for their address will revert, resulting in their prize-funds being locked.

As an analogy, the protocol is currently like a lottery where there can be more than 1 grand prize winner (tier 0). In the event there is 2 grand prize winners, it allows both users to make a claim on the full grand prize instead of allocating an equal portion to each winner. The issue is present in each tier level with varying frequencies and impacts.

## Vulnerability Detail

The expected number of prizes awarded for any given draw and tier is given by 4^t where t is the tier number. For example, in tier 1, there is 4 expected prizes. In reality, this means over a large number of draws the average number of tier 1 prizes awarded per draw will approach 4.

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L39-L41

The way this is implemented, in tier t, is by allowing users to generate 4^t different `userSpecificRandomNumber` in the `PrizePool::isWinner()` function. This means, users get 4^t chances to generate a random number in the winning zone (ie. in tier t, users are 4^t more likely to be able to claim a prize than in tier 0).

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1022-L1029

```javascript
    uint256 userSpecificRandomNumber = TierCalculationLib.calculatePseudoRandomNumber(
      lastAwardedDrawId_,
      _vault,
      _user,
      _tier,
@>    _prizeIndex,
      _winningRandomNumber
    );
```

The user wins a prize for a given vault, draw, tier, and prize index if their user-specific uniformly distributed psuedo-random number falls within the `winning zone`:

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L68-L70 

Due to the statistical independence of each user's random numbers generated, there is theoretically no limit to the number of prizes that can be won in a given draw and tier. This results in the possibility of the required liquidity in a tier exceeding the available liquidity in that tier. 

The issue is made possible because the `tierLiquidity.prizeSize` is updated when a user calls `PrizePool::claimPrize()` via `TieredLiquidityDistributor::_getTier()` and `TieredLiquidityDistributor::_computePrizeSize()`. As shown below, the prize size returned is based on dividing the `remainingTierLiquidity` by `prizeCount` which is the <u>expected prize count (4^t)</u>, not the <u>actual prize count</u> for that draw and tier.

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L411-L429

```javascript
  function _computePrizeSize(
    uint8 _tier,
    uint8 _numberOfTiers,
    uint128 _tierPrizeTokenPerShare,
    uint128 _prizeTokenPerShare
  ) internal view returns (uint104) {
    uint256 prizeCount = TierCalculationLib.prizeCount(_tier);
    uint256 remainingTierLiquidity = _getTierRemainingLiquidity(
      _tierPrizeTokenPerShare,
      _prizeTokenPerShare,
      _numShares(_tier, _numberOfTiers)
    );

    uint256 prizeSize = convert(
@>    convert(remainingTierLiquidity).mul(tierLiquidityUtilizationRate).div(convert(prizeCount))
    );

    return prizeSize > type(uint104).max ? type(uint104).max : uint104(prizeSize);
  }
```

Crucially, note that in the event there are less than the expected number of winners in a draw and tier, the protocol does not 'save' these extra prizes in reserves to protect against this issue ocurring in the future. It simply inflates that tier's liquidity for future draws. As shown above, in future draws `tierLiquidity.prizeSize` will be recalculated based on `remainingTierLiquidity` and `prizeCount`.

Also note there is no protection against this issue by ensuring the reserves exceed a sufficient size. At any point, the  `drawManager` can allocate any amount of the reserves to any address as a reward:

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L434-L449

The risk of at least one prize being unclaimable is present in any draw where a tier's prize size `tierLiquidity.prizeSize` is greater than the reserve amount. Note that unless the value of the grand prize is kept in the reserve, the issue will be present in every single draw.


## Impact

When `tierLiquidity.prizeSize` exceeds `TieredLiquidityDistributor._reserve`, and there are more winners than the expected number of winners, a race to call `PrizePool::claimPrize()` begins. In tier t, the first 4^t prize claims will be successful, with each subsequent call reverting.

Addresses that were entitled to a prize, but were not the first 4^t winning addresses to claim will <b><u>forfeit their prize</b></u>.

## Code Snippet

The POC below was adapted from a test in `PrizePool.t.sol` and shows a scenario where five users can make a prize claim in tier 1 for a given draw. There is not enough liquidity for all users to claim, so the fifth claim reverts.

Paste the following import and test in PrizePool.t.sol.

```javascript

import { InsufficientLiquidity } from "../src/abstract/TieredLiquidityDistributor.sol";

  function test_ClaimPrizesRevertsScenarioOne() public {
    uint users = 50;
    uint balance = 1e18;
    uint totalSupply = users * balance;
    uint8 t = 1; // Show revert in tier 1
    uint256 yieldAmt = 100e18;

    // Vault contributes yield to prize pool
    contribute(yieldAmt);

    // Draw is awarded with this random number
    uint256 random = uint256(keccak256(abi.encode(1234)));
    awardDraw(random);

    // Based on this seed, addresses, and TWAB values, Users 10, 15, 16, 18, and 41 win a prize in tier 1
    address user10 = makeAddr(string(abi.encodePacked(uint(10))));
    address user15 = makeAddr(string(abi.encodePacked(uint(15))));
    address user16 = makeAddr(string(abi.encodePacked(uint(16))));
    address user18 = makeAddr(string(abi.encodePacked(uint(18))));
    address user41 = makeAddr(string(abi.encodePacked(uint(41))));
    address user42 = makeAddr(string(abi.encodePacked(uint(42))));

    // Mock user TWAB
    mockTwabForUser(address(this), user10, t, balance);
    mockTwabForUser(address(this), user15, t, balance);
    mockTwabForUser(address(this), user16, t, balance);
    mockTwabForUser(address(this), user18, t, balance);
    mockTwabForUser(address(this), user41, t, balance);
    mockTwabForUser(address(this), user42, t, balance);

    // Mock vault TWAB
    mockTwabTotalSupply(address(this), t, totalSupply);

    // 5 Different users are able to claim prizes for this draw in tier 1
    assertEq(prizePool.isWinner(address(this), user10, 1, 0), true);
    assertEq(prizePool.isWinner(address(this), user15, 1, 2), true);
    assertEq(prizePool.isWinner(address(this), user16, 1, 3), true);
    assertEq(prizePool.isWinner(address(this), user18, 1, 0), true);
    assertEq(prizePool.isWinner(address(this), user41, 1, 1), true);

    // There is not enough liquidity for all 5 winners
    assertGt(5 * prizePool.getTierPrizeSize(1), prizePool.getTierRemainingLiquidity(1) + prizePool.reserve());

    // Race condition is has been created, first 4 users can claim
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user10, 1, 0, user10, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user15, 1, 2, user15, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user16, 1, 3, user16, 0, address(this)));
    assertEq(prizePool.getTierPrizeSize(1), prizePool.claimPrize(user18, 1, 0, user18, 0, address(this)));

    // Fifth user's claim reverts due to insufficient liquditiy
    vm.expectRevert(abi.encodeWithSelector(InsufficientLiquidity.selector, prizePool.getTierPrizeSize(1)));
    prizePool.claimPrize(user41, 1, 1, user41, 0, address(this));
  }
```

## Tool used

Manual Review

## Recommendation

Without changing the core logic of winner selection, my recommended mitigation falls into two categories:

  1. Waiting until the end of the claim period before calculating `tierLiquidity.prizeSize` and distributing pool tokens. This means the payout values for a given draw and tier will be based on the actual number of winners, not the expected number of winners. As an example, if two addresses were to win the grand prize on the same draw, they would each recieve half of the total grand prize.
  This implementation would be more like a lottery where the total prize pool in each tier is guaranteed, but not the value of each winner's payout, this is only determined after the number of successful winners has been determined.

  2. Managing the size of the reserve pool (`TieredLiquidityDistributor._reserve`) by taking steps to keep the reserve pool above some 'minimum reserve buffer':

- Requiring the reserve stays above the 'minimum reserve buffer' after each call to `prizePool::allocateRewardFromReserve()`
- Bootstrapping reserve pool to the 'minimum reserve buffer' by temporarily increasing `TieredLiquidityDistributor.reserveShares` upon PrizePool deployment, or if reserve dips below 'minimum buffer'
- For draws where the actual number of prize claims is less than the expected number, transfer the extra prizes to the reserve until the reserve pool exceed some 'minimum reserve buffer'.
  
    Note that the size of 'minimum reserve buffer' depends on the protocol's risk appetite for this issue arrising. To reduce the risk significantly, ideally the size of the reserve would exceed the liquidity available in tier 0. However the significant tradeoff is user's would recieve less yield because the reserve would need to grow in accordance with the grand prize pool. 


