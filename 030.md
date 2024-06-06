Shambolic Red Gerbil

medium

# `uint96` is too small for `_directlyContributedReserve`, which is cumulative

## Summary

`_directlyContributedReserve` is declared as `uint96`, which is too small as it is a cumulative value.

## Vulnerability Detail

In `PrizePool.sol`, `_directlyContributedReserve` is the cumulative amount of tokens that that is directly contributed to the reserve:

[PrizePool.sol#L298-L299](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L298-L299)

```solidity
  /// @notice Tracks reserve that was contributed directly to the reserve. Always increases.
  uint96 internal _directlyContributedReserve;
```

When `contributeReserve()` is called, the amount of tokens added to the reserve is also added to `_directlyContributedReserve`:

[PrizePool.sol#L624-L629](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L624-L629)

```solidity
  function contributeReserve(uint96 _amount) external notShutdown {
    _reserve += _amount;
    _directlyContributedReserve += _amount;
    prizeToken.safeTransferFrom(msg.sender, address(this), _amount);
    emit ContributedReserve(msg.sender, _amount);
  }
```

However, unlike `_reserve`, `_directlyContributedReserve` is a cumulative amount (ie. the total amount of tokens contributed to the reserve over the prize pool's lifetime) and never decreases. 

Therefore, `uint96` is too small when `prizeToken` is a token with low value. For example, PEPE has 18 decimals and a current price of $0.00001468. `uint96` is only able to store up to $1,163,070 worth of PEPE, which is easily reachable since `_directlyContributedReserve` is a cumulative amount.

Once `_directlyContributedReserve` reaches `type(uint96).max`, `contributeReward()` will always revert when called and will be permanently DOSed.

## Impact

When `prizeToken` has low value, `contributeReward()` will be permanently DOSed after the prize pool has been live for some time. Note that `contributeReward()` needs to be called to add to the pool's reserve when:

1. The current reserve amount is too little compared to gas prices or the RNG fee, so draw bots have little to no incentive to call `DrawManager.startDraw()` or `DrawManager.finishDraw()` to award draws.
2. The number of prizes won is more than expected, causing the reserves to be used to award prizes.

If `contributeReward()` cannot be called to increase the pool's reserves, (1) breaks core functionality of the protocol as draws will not be automatically awarded and will be missed, while (2) causes a loss of funds as there will be no liquidity for prizes.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L298-L299

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L624-L629

## Tool used

Manual Review

## Recommendation

Use a larger type for `_directlyContributedReserve`, such as `uint128`.
