Calm Infrared Seahorse

medium

# The number of tiers is incorrectly used when claiming rewards.

## Summary
The number of tiers is incorrectly used when claiming rewards.
## Vulnerability Detail
When a draw is closed it can be awarded. It is done so by calling the `awardDraw` function. When the function is called it will update the number of tiers for the next draw.

The number of tiers can change each draw such that the number of tiers is increased or decreased by 1, depending on how many tiers have been claimed. 

After the draw is awarded the prizes can be claimed for the awarded draw (keep in mind that the number of tiers has already been changed when awarding a draw). 
When claiming prizes the winner is checked by invoking the `isWinner` function.

The `isWinner` function will check if the tier that a user wants to claim a prize for is valid

The issue is that it compares the tier to the new already updated number of tiers meant for the next draw. Instead, it should use the number of tiers that were present in the awarded draw. This can cause the last tier not to be claimed if the number of tiers decreases or will let users claim a tier that should not be claimed in the given draw if the number of tiers increases.

#### Imagine the following scenario:
- Draw 10 is closed and has 8 tiers.
- After it is closed it is awarded and a number of tiers change to 7.
- The user cannot claim a prize for tier 8 (index 7) as the number of tiers has decreased, even though 8 tiers were awarded during draw 10.

## Impact
Incorrect use of a number of tiers will prevent all tiers from being claimed.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L476-L480

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L472

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/abstract/TieredLiquidityDistributor.sol#L248

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L547

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L1009-L1011
## Tool used
Manual Review
## Recommendation
Use the number of tiers that were awarded when claiming prizes.
