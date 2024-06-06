Suave Porcelain Owl

medium

# Unrestricted Access to mint Function

## Summary
The function is expected to be called by the vault (as stated in the comments) but there seems to be a missing access control. Which mean that any address can call it and it could be exploited to create an arbitrary amount of tokens, leading to inflation and loss of value.
## Vulnerability Detail
The mint function does not restrict which address can call it, allowing any address to mint new tokens. This can lead to unauthorized token minting, resulting in inflation of the token supply and potential devaluation of the token.
## Impact
Unlimited tokens can be minted. Token value could drop.
## Code Snippet
  function mint(address _to, uint96 _amount) external {
    if (_to == address(0)) {
      revert TransferToZeroAddress();
    }
    _transferBalance(msg.sender, address(0), _to, _amount);
  }

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-twab-controller/src/TwabController.sol#L484-L489
## Tool used

Manual Review

## Recommendation
Use a modifier to control access.