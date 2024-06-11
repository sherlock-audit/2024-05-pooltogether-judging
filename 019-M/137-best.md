Real Ceramic Koala

high

# `shutdownBalanceOf` calculation can overflow

## Summary
The shutdownBalanceOf calculation can revert due to multiplication overflow

## Vulnerability Detail
In shutdownBalanceOf, the multiplication `return (shutdownPortion.numerator * balance) / shutdownPortion.denominator;` can over flow uint256 

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L916-L931)
```solidity
      shutdownPortion = computeShutdownPortion(_vault, _account);
      _shutdownPortions[_vault][_account] = shutdownPortion;
    } else {
      shutdownPortion = _shutdownPortions[_vault][_account];
    }


    if (shutdownPortion.denominator == 0) {
      return 0;
    }

=>  return (shutdownPortion.numerator * balance) / shutdownPortion.denominator;
```

The shutdownPortion.numerator component expanded is equal to `vaultContrib * _userTwab` for the `grandPrizePeriodSize`. 

[link](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L873-L894)
```solidity
  function computeShutdownPortion(address _vault, address _account) public view returns (ShutdownPortion memory) {

    ...

=>  return ShutdownPortion(vaultContrib * _userTwab, totalContrib * _vaultTwabTotalSupply);
```

Hence in case of lower valued tokens, it can overflow uint256 due to 3 consecutive multiplication

Eg:
(10^18 * 10^18 * 10^18 * 10^10 (user twab) * 10^10 (vaultcontrib) * 10^4 (current balance)) == 10^78 > 2^256

For token such a MILADY, 10^10 tokens cost ~2500$

## Impact
User's won't be able to withdraw their assets after vault shutdown for some lower valued tokens

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L916-L931

## Tool used
Manual Review

## Recommendation
Refactor the calculation