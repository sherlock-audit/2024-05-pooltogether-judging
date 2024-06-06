Creamy Ginger Salamander

medium

# gas yield can never be claimed and all yield will be lost

## Summary
Gas yield can never be claimed and all yield will be lost

## Vulnerability Detail
The protocol has clearly stated they plan on deploying to Blast

They also state the following
>We're interested to know if there will be any issues deploying the code as-is to any of these chains

During deployment of the contracts on Blast, the contract's gas yield mode should be configured to be able to claim accrued gas fees spent on the contract. 

This is significant because for this protocol specifically a lot of gas will be spent often by all the bots doing their functions described by the protocol in the README as the following:

>Draw Bots that take advantage of the two phase Dutch auction that is used to trigger the RNG request.
Liquidation Bots that swap yield for WETH on the prize vaults
Claimer Bots that earn fees by claiming prizes for users

## Impact
The protocol is meant to be deployed on blast, meaning that the gas accrue yield.

Gas yields will never accrue to the protocol contracts and the yield will forever be lost.

The protocol should be refunding these bots their gas expenses on blast, not doing so reduces their profit margins, dis-incentivizing them to call functions in a timely manner.

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-rng-witnet/src/RngWitnet.sol#L1

## Tool used

Manual Review

## Recommendation
 set the gas yield of the contracts to claimable 
