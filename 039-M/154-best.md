Real Ceramic Koala

medium

# User's might be able to add already errored witnet requests on L2's

## Summary
User's might be able to add already errored witnet requests on L2's 

## Vulnerability Detail
`block.number` of L2's like Arbitrum return Ethereum's block number but [lagged](https://docs.arbitrum.io/build-decentralized-apps/arbitrum-vs-ethereum/block-numbers-and-time#ethereum-block-numbers-within-arbitrum). Currently it is synced approximately every minute but is recommended to be only relied on a scale of hours. 
The current best time possible for witnet to report is 3 minutes (usually reported in 5-10 mins) and the wit/2 upgrade will (most likely) [decrease block time](https://discord.com/channels/492453503390842880/1154045538900115456/1247216363961843733). 
In such a case blocks could occur such that the rngRequest made at `block.number` is reported in the same block with `ERROR` as the status. 
Since no checks are made for the request status inside `startDraw` for the new request id, a user can add such a request and earn additional reward without any useful work

## Impact
User can claim reward without actually contributing to obtaining the random number

## Code Snippet
https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-draw-manager/src/DrawManager.sol#L219-L225

## Tool used
Manual Review

## Recommendation
Check if the status is `ERROR` before adding a request