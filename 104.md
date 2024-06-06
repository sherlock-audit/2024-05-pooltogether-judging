Refined Paisley Otter

high

# The result of the function `calculatePseudoNumber` can be previewed before calling `claimPrize`

## Summary

Weak PNRG is a very know vulnerability that happens a random number can be manipulated through one of it's variables, in this case, the address. To try to mitigate this issue, the probability is tied to the amount in the twab, which in theory makes the amount invested the only considered factor on luck, but one thing makes it still vulnerable.

## Vulnerability Detail

When `awardDraw` is called, it saves the `_lastAwardedDrawId`  with the return value of the `getOpenDrawId` function:
```javascript
function getOpenDrawId() public view returns (uint24) {
    uint24 shutdownDrawId = getShutdownDrawId();
    uint24 openDrawId = getDrawId(block.timestamp);
    return openDrawId > shutdownDrawId ? shutdownDrawId : openDrawId;
  }

function getDrawId(uint256 _timestamp) public view returns (uint24) {
    uint48 _firstDrawOpensAt = firstDrawOpensAt;
    return
      (_timestamp < _firstDrawOpensAt)
        ? 1
        : (uint24((_timestamp - _firstDrawOpensAt) / drawPeriodSeconds) + 1);
```
`lastAwardedDraw` is used as a parameter for the ending drawId in the function `getVaultUserBalanceAndTotalSupplyTwab`, which will get the twab that is between the opening and the closing of the draw using the timestamp based on those, that means, you can still alter the amount in twab before calling `claimPrize`.
The function `calculatePseudoNumber`, calculates a pseudo random number using the user address, and if that number is less than another calculated value, the user is a winner. Homever, right after `awardDraw` is called, the user can get all the variables and simulate with a lot of different address if one wins, and after that deposit to match the twab amount.

## Impact

Malicious users can have an advantage on winning over the other users

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-prize-pool/src/libraries/TierCalculationLib.sol#L81C3-L93C4

## Tool used

Manual Review

## Recommendation

Get the user twab from a earlier period