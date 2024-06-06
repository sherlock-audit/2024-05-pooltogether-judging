Bubbly Chocolate Wombat

medium

# Hardcoded Gas for Hooks May Prevent Winners from Receiving Prizes Due to Insufficiency on Different Chains

## Summary
- The hardcoded gas limit of `150,000` for winner hooks may be insufficient on certain chains like `Arbitrum` and `zksync` , causing prize claim failures. This issue arises due to varying gas consumption and fee models across different networks.
## Vulnerability Detail
The [Claimable](https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-vault/src/abstract/Claimable.sol)contract has a hardcoded gas limit of `HOOK_GAS = 150,000` for executing `beforeClaimPrize` and `afterClaimPrize` hooks:
```js
/// @notice The gas to give to each of the before and after prize claim hooks.
/// @dev This should be enough gas to mint an NFT if needed.
uint24 public constant HOOK_GAS = 150_000;
```
- As you see, the comment assumes that this should be enough gas to mint an `NFT`
- However, different blockchain networks have varying gas consumption and fee models. For instance, while `150,000` gas might be sufficient on Ethereum, it may not be enough on other chains like `Arbitrum` or `zkSync` due to their gas requirements and evolving fee modules.which is mentioned in their docs [here](https://docs.zksync.io/build/quick-start/best-practices.html#do-not-rely-on-evm-gas-logic)
- This discrepancy can cause the hooks to fail on certain chains where the protocol will be deployed, leading to transaction reverts when calling the *winner hook* when the gas required for these hooks exceeds `150,000` on certain chains for the same action (Minting an Nft), preventing the winner from recieving their prize.
### POC : 
- add a mainnet  url to fork arbitrum and see how the tx revert due `outOfGas` in [this test](https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/test/unit/Claimable.t.sol#L153-L172) : 
```diff
    function testClaimPrize_bothHooks() public {
++      vm.createSelectFork(arb_url);
            PrizeHooks memory bothHooks = PrizeHooks(true, true, hooks);

            vm.startPrank(alice);
            claimable.setHooks(bothHooks);
            vm.stopPrank();

            mockClaimPrize(alice, 1, 2, prizeRedirectionAddress, 1e17, bob);

            vm.expectEmit();
            emit BeforeClaimPrizeCalled(alice, 1, 2, 1e17, bob);
            vm.expectEmit();
            emit AfterClaimPrizeCalled(alice, 1, 2, 9e17, prizeRedirectionAddress, "hook data");

            vm.recordLogs();
            claimable.claimPrize(alice, 1, 2, 1e17, bob);
            Vm.Log[] memory logs = vm.getRecordedLogs();

            assertEq(logs.length, 2);
        }
```
- console : 
```sh
    ├─ [150000] ClaimableTest::afterClaimPrize(alice: [0x328809Bc894f92807417D2dAD6b7C998c1aFdac6], 1, 2, 900000000000000000 [9e17], prizeRedirectionAddress: [0x1dD8E5dC40Be9fB68833C78dd78D3F71EA16f762], 0x686f6f6b2064617461)
    >>  │   │   └─ ← [OutOfGas] EvmError: OutOfGas
        │   └─ ← [Revert] EvmError: Revert
 ```
## Impact
- Hooks will fail on certain chains, preventing winners from receiving their prizes.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L21
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L86-L93
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-vault/src/abstract/Claimable.sol#L109-L117
## Tool used
Manual Review , Foundry Testing
## Recommendation
 - Provide chain-specific configurations to adjust the gas limit for winner hooks  appropriately.
