Shambolic Red Gerbil

medium

# Distributing liquidity based on the last `grandPrizePeriodDraws` days post-shutdown is problematic

## Summary

After a prize pool shuts down, its funds are distributed based on draws before it shut down. This could cause a loss of funds for prize pools that contribute new liquidity to the pool post-shutdown.

## Vulnerability Detail

After a vault is shutdown, all existing and new liquidity is distributed based on a user and vault's time-weighted prize contribution during the last `grandPrizePeriodDraws` before shutdown:

[PrizePool.sol#L873-L895](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L873-L895)

```solidity
  function computeShutdownPortion(address _vault, address _account) public view returns (ShutdownPortion memory) {
    uint24 drawIdPriorToShutdown = getShutdownDrawId() - 1;
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

    if (_vaultTwabTotalSupply == 0) {
      return ShutdownPortion(0, 0);
    }

    return ShutdownPortion(vaultContrib * _userTwab, totalContrib * _vaultTwabTotalSupply);
  }
```

As seen from above, the draw ID used as the ending draw is `getShutdownDrawId() - 1`, which is the draw before the pool shut down.

However, this method of allocating liquidity is problematic when considering that new liquidity can be contributed to the prize pool, even after it is shutdown.

Firstly, if a vault shuts down before any liquidity is contributed through `contributePrizeTokens()`, all future liquidity contributed to the prize pool will be permanently stuck. For example:

- A prize pool and vault are newly deployed.
- Assume that `drawTimeout = 1`, which means the prize pool shuts down after one draw is missed.
- In the first draw period, the prize pool does not generate any yield, and therefore, it does not contribute any liquidity.
- The first draw is missed as there is no prize liquidity, so the pool shuts down.
- In the second draw period, the prize pool generates some yield and calls `contributePrizeTokens()` to add it to the prize pool.
- When users of the vault attempt to withdraw the yield from the prize pool through `withdrawShutdownBalance()`:
  - In `computeShutdownPortion()`, `totalContrib = 0` as no liquidity was contributed before the pool shut down.
  - In `shutdownBalanceOf()`, `shutdownPortion.denominator = 0`, so the withdrawable balance for all users is `0`.
- As such, since all users cannot withdraw any balance from the prize pool, the new prize liquidity is effectively stuck forever.

Secondly, prize pools that did not contribute any liquidity before the pool shut down, but start doing so post-shutdown will always lose their yield. For example:
- Assume two vaults are deployed, called vault A and B.
- Before the pool shuts down:
  - Vault A calls `contributePrizeTokens()` and contributes some prize liquidity.
  - Vault B does not generate any yield, so it does not contribute any prize liquidity.
- After the pool shuts down, vault B starts generating yield and consistently calls `contributePrizeTokens()`.
- However, since vault B did not contribute any liquidity before the pool shut down, all contributions from vault B are only withdrawable by users of vault A.
- This causes a loss of yield for users in vault B.

Note that since vaults call `contributePrizeTokens()` without checking if the pool has shut down and the liquidation of yield is performed automatically by liquidation bots, it is not unlikely for a vault to contribute liquidity to a pool after it is shut down.

## Impact

When a vault contributes to the prize pool post-shutdown but did not do so before shutdown, all users of the vault lose their yield, causing a loss of funds.

In the case where the prize pool did not receive any liquidity before shutdown, all funds contributed post-shutdown will be permanently stuck in the pool, causing a loss of yield for all users.

## Code Snippet

## Tool used

Manual Review

## Recommendation

Consider a different method of allocating funds in the prize pool after shutdown. For example, the ending draw in `computeShutdownPortion()` could include the latest draw.