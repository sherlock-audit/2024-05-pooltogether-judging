Festive Bone Cobra

medium

# Current Tpda liquidations can not deal with upwards yield fluctuations, leading to yield loss in some scenarios

## Summary

Tpda liquidations are vulnerable to yield fluctuations, despite the existence of the smooth factor, as they can not deal with upwards yield movement after the price point has passed and allow arbitrage. Thus, most of this yield may be arbitraged, given a big enough upward price movement of the underlying prize vault.

## Vulnerability Detail

Tpda liquidations set a target period and use as reference price the last auction price. This works really well when the yield vault is well behaved, as the yield from the `PrizeVault` will smoothly match the price from the Tpda liquidation.

However, when the yield is less consistent, the Tpda has issues dealing with upwards yield price movement after it has already decreased the price. The smooth factor [exists](https://dev.pooltogether.com/protocol/design/#target-period-dutch-auction-tpda) as a mitigation for such scenarios, but it is not significant enough as a lot of yield may still arbitraged for very little assets contributed to the prize pool. 
> This simple mechanism allows the auctions to handle yield that fluctuates significantly by setting the smoothing percentage to a higher value.

Additionally, the effectiveness of the smooth factor depends largely on the time period that yield keeps accruing. Given a long enough period where yield is not signficantly accrued, the liquidation yield will be depleted over time, as it mostly consists of previous liquidations not having been fully completed due to the smoothing factor.

One example of such an event happening is a yield vault where the yield is relatively stale for the first hours / days. The price of the tpda will keep decreasing until it becomes profitable to liquidate, but a sudden yield increase is impossible to deal with in this system, as the tpda can not regain the price back up. For simplicity, consider the extreme case where the minimum price of `100` is reached due to a stale yield vault. If the yield vault has a sudden yield increase of `10%` and the vault holds `1e6` USD, this is `100k` USD that may be freely arbitraged by liquidators (may be less due to the smoothing factor, for example, `80%` would arbitrage 20k). This is an extreme scenario, but this risk exists and is a limitation of the tpda.

Additionally, often yield reports in yield vaults can be triggered by users / protocol admin, worsening this situation.

## Impact

Given an upwards price movement, liquidators may arbitrage a significant portion of the gained yield, while the prize pool loses.

## Code Snippet

https://github.com/sherlock-audit/2024-05-pooltogether/blob/main/pt-v5-tpda-liquidator/src/TpdaLiquidationPair.sol#L195-L199

## Tool used

Manual Review

Vscode

## Recommendation

Due to not using oracles a flawless solution is not possible, but this may be further mitigated by resetting the Tpda if there is a significant instantaneous upward yield movement. Past a certain yield fluctuation, the price discovery process should reset, because the price point may be completely off and liquidators may arbitrage all this profit and there is nothing that can be done. To achieve this, the twap price of uniswap, for example, could be used to detect big discrepancies and reset the curve in this case.