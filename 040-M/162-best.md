Real Ceramic Koala

medium

# Users can setup hooks to control the expannsion of tiers

## Summary
Reentrancy allows to not claim rewards and update tiers

## Vulnerability Detail
The fix for the [issue](https://code4rena.com/reports/2024-03-pooltogether#m-01-the-winner-can-steal-claimer-fees-and-force-him-to-pay-for-the-gas) still allows user's claim the rewards. But as mitigation, the contract will revert. But this still allows the user's to specifically revert on just the last canary tiers in order to always prevent the expansion of the tiers

## Impact
User's have control over the expansion of tiers

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L87

## Tool used
Manual Review

## Recommendation