Bubbly Chocolate Wombat

medium

# `PUSH0` opcode Is Not Supported on Linea yet

## Summary

## Vulnerability Detail
 - The current codebase is compiled with Solidity `version 0.8.24`, which includes the `PUSH0` opcode in the compiled bytecode. According to the [README](https://github.com/sherlock-audit/2024-05-pooltogether?tab=readme-ov-file#q-on-what-chains-are-the-smart-contracts-going-to-be-deployed), the protocol will be deployed on the Linea network.
 
 -  This presents an issue because Linea does not yet support the `PUSH0` opcode, which can lead to unexpected behavior or outright failures when deploying and running the smart contracts.[see here](https://docs.linea.build/developers/quickstart/ethereum-differences#evm-opcodes)
 
## Impact
- Deploying the protocol on `Linea` with the current Solidity version (0.8.24) may result in unexpected behavior or failure due to the unsupported `PUSH0` opcode.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L2C1-L3C1
## Tool used
 Manual Review
## Recommendation
- for Linea you may consider to use version 0.8.19 to compile .
