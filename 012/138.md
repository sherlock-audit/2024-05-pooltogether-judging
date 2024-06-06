Real Ceramic Koala

medium

# `permit` is not compatible with DAI

## Summary
`permit` is not compatible with DAI

## Vulnerability Detail
The EIP-2612 Permit standard is not compatible with dai and other similar tokens which has a different [function signature](https://eips.ethereum.org/EIPS/eip-2612#backwards-compatibility). Hence the `depositWithPermit` function will revert incase of such tokens 

## Impact
User's won't be able to use the permit functionality for depositing dai and similar tokens

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/PrizeVault.sol#L546

## Tool used
Manual Review

## Recommendation
Acknowledge