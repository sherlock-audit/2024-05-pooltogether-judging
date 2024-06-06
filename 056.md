Creamy Ginger Salamander

high

# If a user calls `contributePrizeTokens` they are vulnerable to a frontrunning attack

## Summary
If a user calls `contributePrizeTokens` they are vulnerable to a frontrunning attack 

## Vulnerability Detail
Inside PrizePool.sol the purpose of `contributePrizeTokens()` is clearly stated in the natspec
>Contributes prize tokens on behalf of the given vault.

Another important point to note is that the natspec states
>The tokens should have already been transferred to the prize pool.

Hence when a user wants to call this function, they will first transfer the amount of PrizeTokens into the PrizePool contract and then call `contributePrizeTokens` 

This opens up an attack vector by which an attacker can steal the funds a user intended to contribute on behalf of their vault, instead for the attacker's vault.

It involves the following steps

1. An honest user wants to contribute on behalf of their vault, so they follow the natspec and transfer an `_amount` of tokens into the PrizePool contract

2. The honest user then calls `contributePrizeTokens` passing in their `_prizeVault` and the `_amount` they just transferred in

3. An attacker sees the Tx to call `contributePrizeTokens` in the mempool and frontruns it by calling `contributePrizeTokens` but passing in the attacker's `_prizeVault` for the same `_amount`

**In the final state,**  the honest user's tokens will contribute to the attacker's prizeVault AND the honest user's call to `contributePrizeTokens`  will revert



## Impact
the honest user's tokens will contribute to the attacker's prizeVault AND the honest user's call to `contributePrizeTokens`  will revert

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L406-L422

## Tool used

Manual Review

## Recommendation
Access control `contributePrizeTokens()` to the contracts that should call it.

OR

Educate users that this function is vulnerable to the described attack and hence should not be called by them.