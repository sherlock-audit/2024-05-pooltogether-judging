Real Ceramic Koala

medium

# Vault beneficiary contribution may be shared to all in case of shutdown

## Summary
If the shutdown timestamp occurs in between a draw, then the draw manager's reserve donation will be shared across all user's instead of favoring vault beneficiary

## Vulnerability Detail
When the lastObservationAt timestamp happens to not coincide with the draw end time stamp, the shutdownDrawId will coincide with the draw (lastAwardedDrawId + 1), in case the most recent draw is being awarded.And hence the finishReward call will also be made in the shutdownDrawId

Eg:
draw length = 2 days
period length = 1 day
lastObservation occurs at day 5 which would be draw 3 and the finish draw function will be called for draw 2 in this draw (before lastObservation occurs ie. in the time range 4-5 day)
 
In such a scenario, the donation of the draw manager intended to the vault beneficiary will be shared to all user's instead

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L314-L369)
```solidity
  function finishDraw(address _rewardRecipient) external returns (uint24) {
    
    ....

    uint256 remainingReserve = prizePool.reserve();


    emit DrawFinished(
      msg.sender,
      _rewardRecipient,
      drawId,
      _computeElapsedTime(startDrawAuction.closedAt, block.timestamp),
      _finishDrawReward,
      remainingReserve
    );
    
    if (remainingReserve != 0 && vaultBeneficiary != address(0)) {
      _reward(address(this), remainingReserve);
      prizePool.withdrawRewards(address(prizePool), remainingReserve);
=>    prizePool.contributePrizeTokens(vaultBeneficiary, remainingReserve);
    }
```

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L873-L888)
```solidity
  function computeShutdownPortion(address _vault, address _account) public view returns (ShutdownPortion memory) {
    // @audit balances / contributions are only considered till shutdownDrawId - 1
=>  uint24 drawIdPriorToShutdown = getShutdownDrawId() - 1;
    uint24 startDrawIdInclusive = computeRangeStartDrawIdInclusive(drawIdPriorToShutdown, grandPrizePeriodDraws);


    (uint256 vaultContrib, uint256 totalContrib) = _getVaultShares(
      _vault,
      startDrawIdInclusive,
      drawIdPriorToShutdown
    );


    (uint256 _userTwab, uint256 _vaultTwabTotalSupply) = getVaultUserBalanceAndTotalSupplyTwab(
      _vault,
      _account,
      startDrawIdInclusive,
      drawIdPriorToShutdown
    );
```

## Impact
Rewards intended for vault beneficiary will be shared to all user's

## Code Snippet

## Tool used
Manual Review

## Recommendation
Sync the drawPeriod such that the lastObservationAt will always occur at a draw end