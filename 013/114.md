Bubbly Chocolate Wombat

medium

# Unfair Manipulation of Winning Chances Due to Stolen Yield on `Blast`

## Summary
- The `PoolTogether` protocol on [Blast](https://docs.blast.io/about-blast) will use `WETH` as it's prize token as mentioned by sponsor, which generates yield over time. This yield can be stolen and used to manipulate the vaults' winning chances unfairly. By contributing the stolen yield to their own vault, users can inflate their vault's portion, increasing their chances of winning prizes while decreasing the chances for other legitimate users. This undermines the fairness of the prize distribution.
## Vulnerability Detail

- The `PoolTogether` protocol will be deployed on [Blast](https://docs.blast.io/about-blast), and as the sponsor mentioned, `WETH` will be the prize token.
- `WETH` on [Blast](https://docs.blast.io/about-blast) is a rebasing token that automatically generates yield over time. `WETH` holders have three Yield Modes for rebasing: `Void`, `Automatic`, and `Claimable`. The **Default** mode is `Automatic`, which means the balance will increase over time by the yield generated.
- The [prizePool]() will hold `WETH` as the prize token and will generate yield over time. However, this yield can be stolen by anyone, as they can contribute it to their own vault by calling the `contributePrizeTokens()` function, passing their vault and the amount of yield generated.
```js
function contributePrizeTokens(address _prizeVault, uint256 _amount) public returns (uint256) {
  >>  uint256 _deltaBalance = prizeToken.balanceOf(address(this)) - accountedBalance();
    if (_deltaBalance < _amount) {
      revert ContributionGTDeltaBalance(_amount, _deltaBalance);
    }
    uint24 openDrawId_ = getOpenDrawId();
   >> _vaultAccumulator[_prizeVault].add(_amount, openDrawId_);
   >> _totalAccumulator.add(_amount, openDrawId_);
    emit ContributePrizeTokens(_prizeVault, openDrawId_, _amount);
    return _deltaBalance;
  }

```
- The idea is that this yield is generated from all vault contributions. However, a malicious user can exploit this feature to their advantage. While this might be considered a missing feature (not benefiting from yield) in different protocol contexts, within the `PoolTogether` on [Blast](https://docs.blast.io/about-blast) scenario, it has a detrimental effect. The malicious user's ability to exploit the yield generation artificially inflates their vault's portion compared to other vaults. **This act will decrease other vaults portions consequently increases the malicious user vault portion, thus increase his chances of winning prizes and decrease others chances**.
- For the malicious user's vault, the increase in their contribution portion will lead to a higher `winning zone` thus more chances to win. Conversely, for other vaults, the decrease in their contribution portions will lead to a smaller  `winning zone` reducing their chances of winning. 

- This manipulation undermines the fairness of the prize distribution. Not only will the yield be stolen from all vaults contributions and not be spread to them proportionally, but it will also affect their winning chances by decreasing them. This gives the malicious user an unfair advantage by increasing their likelihood of winning at the expense of others.

### Scenario

**Normal Case**

- Let's say from draw 4 to draw 40, we have:
  - `vault1.contributionBetween = 100`
  - `vault2.contributionBetween = 100`
  - `totalContributionBetween = 200`
- To calculate their portions:
  - `v1.portion = 100 / 200 = 0.5`
  - `v2.portion = 100 / 200 = 0.5`

> Assuming the TWAB balance is 500 for both vaults and they both have 1 user for easy calculation(so users twab also 500) , and ignoring the odds scaling:
  - `winningZone(vault1[user]) = 500 * 0.5 = 250`
  - `winningZone(vault2[user]) = 500 * 0.5 = 250`

**Yield Stolen Case**

- Now let's assume that a malicious user continuously steals yield during this period (from draw 4 to draw 40) 
> `Notice` that the yield will be generated for all the PrizePool balance (probably more than 200 which totalContribution in this period):
  - When we get the contribution of `vault3` (malicious vault) for this period (from draw 4 to 40), we get:
    - `vault3.contribution = 50`

- Now let's recalculate the vaults' portions again:
  - `vault1.contributionBetween = 100`
  - `vault2.contributionBetween = 100`
  - `vault3.contributionBetween = 50`
  - `totalContributionBetween = 250`
- To calculate their portions:
  - `v1.portion = 100 / 250 = 0.4`
  - `v2.portion = 100 / 250 = 0.4`
  - `v3.portion = 50 / 250 = 0.2`

> Assuming the TWAB balance is 500 for all vaults:
  - `winningZone(vault1[user]) = 500 * 0.4 = 200`
  - `winningZone(vault2[user]) = 500 * 0.4 = 200`
  - `winningZone(vault3[maliciousUser) = 500 * 0.2 = 100`

- Notice how the malicious user now has a chance to win without any contribution from it's vault (only stolen yield), and how legitimate users vaults' chances have decreased.

## Impact
- Malicious actors can steal yield generated by the prize pool, inflating their vault's portion and unfairly increasing their win chances. This decreases legitimate players' portion and winning probabilities.
## Code Snippet
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L1037-L1043 
- https://github.com/sherlock-audit/2024-05-pooltogether/blob/1aa1b8c028b659585e4c7a6b9b652fb075f86db3/pt-v5-prize-pool/src/PrizePool.sol#L412-L422
## Tool used
manual review
## Recommendation
- For `Blast`, configure the yield mode to `Claimable` in the constructor so it doesn't affect the `balanceOf()`. Expose a public function that claims this yield and contributes it to the `DONATOR` address. This way, the yield is not wasted, and all contributors benefit from it. Contributors will be incentivized to call this function since it increases the prize size.

```js
 // import those :
 enum YieldMode {
  AUTOMATIC,
  VOID,
  CLAIMABLE
}
interface IERC20Rebasing {
  function configure(YieldMode) external returns (uint256);
  function claim(address recipient, uint256 amount) external returns (uint256);
  function getClaimableAmount(address account) external view returns (uint256);
}
```
```diff

contract prizePool {

 constructor() {

   // you can check by address of by chainid if you are in blast : 
+  if (address(prizeToken) == 0x4300000000000000000000000000000000000004){
+        IERC20Rebasing(address(prizeToken)).configure(YieldMode.CLAIMABLE); //configure claimable yield for WETH
+    }
+  }

+ function claimYield() external returns (uint256){
+    IERC20Rebasing weth = IERC20Rebasing(address(prizeToken));
+    uint amount = weth.getClaimableAmount(address(this));
+     weth.claim(address(this),amount);
+     contributePrizeTokens(DONATOR, amount);
+  }
}
```