Shambolic Red Gerbil

medium

# `TierCalculationLib.calculateWinningZone()` rounding down could severely reduce a user's chances of winning

## Summary

`TierCalculationLib.calculateWinningZone()` rounds down to the nearest whole number, which significantly reduces the odds of winning when vaults use tokens with low decimals.

## Vulnerability Detail

`TierCalculationLib.calculateWinningZone()` calculates a user's winning zone as such:

[TierCalculationLib.sol#L100-L107](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L100-L107)

```solidity
  function calculateWinningZone(
    uint256 _userTwab,
    SD59x18 _vaultContributionFraction,
    SD59x18 _tierOdds
  ) internal pure returns (uint256) {
    return
      uint256(convert(convert(int256(_userTwab)).mul(_tierOdds).mul(_vaultContributionFraction)));
  }
```

Where:
- `_userTwab` is the time-weighted average balance of a user in a vault.
- `_tierOdds` is the probability of a tier being awarded.
- `_vaultContributionFraction` is a time-weighted percentage of a vault's contribution relative to the total contribution of all vaults.

As seen from above, the calculation is performed in `SD59x18` and converted back to a `uint256` number afterwards. This effectively rounds down the winning zone to the nearest whole number as the fraction part of `SD59x18` is removed.

A user wins if a random number between 0 and `_vaultTwabTotalSupply` is less than the calculated winning zone:

[TierCalculationLib.sol#L68-L70](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L68-L70)

```solidity
    return
      UniformRandomNumber.uniform(_userSpecificRandomNumber, _vaultTwabTotalSupply) <
      calculateWinningZone(_userTwab, _vaultContributionFraction, _tierOdds);
```

However, rounding down becomes an issue when `_vaultTwabTotalSupply` is small, as the reduction in the winning zone due to rounding down becomes relatively larger.

Note that `_vaultTwabTotalSupply` and `_userTwab` are time-weighted average balances. When a [vault's asset token](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L137-L138) has low decimals, it is possible for both time-weighted balances to become very small. For example:

- Assume that:
  - The prize pool has been deployed for 365 days.
  - `drawPeriodSeconds = 1 days` and `grandPrizePeriodDraws = 365`, so a grand prize is awarded every 365 days.
- A new vault is deployed with GUSD as the asset token, which has 2 decimals.
- A user deposits 100 GUSD, and one day passes.
- When calculating `_vaultTwabTotalSupply` and `_userTwab` for a grand prize, it weighs the vault's balance over 365 days. 
- Therefore, `_vaultTwabTotalSupply = _userTwab = 100e2 * 1 days / 365 days = 27`.

To illustrate the effect of rounding down in `calculateWinningZone()`:

- `_vaultContributionFraction = 1` as the new vault is the only vault in the prize pool.
- `_tierOdds = 1 / 365`
- `_userTwab * _tierOdds * _vaultContributionFraction = ~0.07506`, which rounds down to 0.

Since the user is the only one in the entire prize pool, he should have a `1 / 365` chance in winning the grand prize in each draw. However, due to rounding down, his odds of winning becomes 0.

Note that his odds will be 0 for the first 13 days, since:

- `_userTwab = 100e2 * 13 days / 365 days = 356`
- `_userTwab * _tierOdds * _vaultContributionFraction = ~0.97579`, which still rounds down to 0.

Even after the first 13 days, his odds will still be significantly reduced as `_vaultTwabTotalSupply` will remain very small. 

Additionally, this example only considers one user and one vault. The impact of rounding down could be much more severe when multiple users and vaults are considered, as `_userTwab` and `_vaultContributionFraction` would be smaller than presented above.

## Impact

When a vault's asset token has low decimals, `calculateWinningZone()` rounding down significantly reduces a user's chances of winning lower tiers (ie. tier 0, tier 1). In extreme cases, his odds can be reduced to 0 for an extended period of time.

This causes a loss of funds for users as they have no chance of winning prizes.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L100-L107

https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L68-L70

## Tool used

Manual Review

## Recommendation

Consider using a more precise method of comparing the random number to the winning zone. For example, `calculateWinningZone()` could return the calculated winning zone as a `SD59x18` unwrapped to `uint256` (ie. a decimal number scaled by `1e18`):

```diff
  function calculateWinningZone(
    uint256 _userTwab,
    SD59x18 _vaultContributionFraction,
    SD59x18 _tierOdds
  ) internal pure returns (uint256) {
    return
-     uint256(convert(convert(int256(_userTwab)).mul(_tierOdds).mul(_vaultContributionFraction)));
+     SD59x18.unwrap(convert(int256(_userTwab)).mul(_tierOdds).mul(_vaultContributionFraction));
  }
```

The random number's upper bound would then be `_vaultTwabTotalSupply * 1e18`, which ensures that the chance of winning is the same without precision loss:

```diff
    return
-     UniformRandomNumber.uniform(_userSpecificRandomNumber, _vaultTwabTotalSupply) <
+     UniformRandomNumber.uniform(_userSpecificRandomNumber, _vaultTwabTotalSupply * 1e18) <
      calculateWinningZone(_userTwab, _vaultContributionFraction, _tierOdds);
```