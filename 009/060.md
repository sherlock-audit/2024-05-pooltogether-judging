Gigantic Iron Orca

high

# Critical Access Control Flaw in PoolTogether V5 Prize Pool Contract - Potential for Prize Manipulation

## Summary

The `setDrawManager` function in the PrizePool contract lacks proper access control and input validation, potentially allowing a malicious actor to disrupt the prize distribution process.

## Vulnerability Detail

The `setDrawManager` function is designed to be called only once by the contract creator to set the address of the Draw Manager contract. However, it does not verify if the provided address is a valid contract address. An attacker could exploit this by setting the `drawManager` to an arbitrary address they control, including an externally owned account (EOA) or a contract with malicious behavior. This would give the attacker control over the draw awarding process, potentially allowing them to manipulate the outcome of draws or disrupt the prize distribution.

## Impact

The impact of this vulnerability is significant. A compromised draw manager could lead to:

*   **Unfair Draws:** The attacker could manipulate the winning random number or other parameters to favor themselves or their associates.
*   **Prize Theft:** The attacker could divert prize funds to their own address or prevent legitimate winners from claiming their prizes.
*   **Denial of Service:** The attacker could disrupt the draw process, preventing any draws from being awarded and causing the prize pool to become unusable.
https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/PrizePool.sol#L392
## Code Snippet

```solidity
function setDrawManager(address _drawManager) external {
    if (msg.sender != creator) {
        revert OnlyCreator();
    }
    if (drawManager != address(0)) {
        revert DrawManagerAlreadySet();
    }
    drawManager = _drawManager;

    emit SetDrawManager(_drawManager);
}
```

## Tool used

Manual Review

## Recommendation

To mitigate this vulnerability, add a check to verify that the `_drawManager` address is a valid contract address before setting it. This can be done using the `extcodesize` opcode, which returns the size of the code at a given address. If the size is zero, it indicates that the address is not a contract.

```solidity
function setDrawManager(address _drawManager) external {
    if (msg.sender != creator) {
        revert OnlyCreator();
    }
    if (drawManager != address(0)) {
        revert DrawManagerAlreadySet();
    }
    // Add check for contract code size
    if (extcodesize(_drawManager) == 0) {
        revert InvalidDrawManagerAddress();
    }
    drawManager = _drawManager;

    emit SetDrawManager(_drawManager);
}
```

By adding this check, the contract ensures that only a valid contract address can be set as the draw manager, preventing attackers from taking control of the prize distribution process.