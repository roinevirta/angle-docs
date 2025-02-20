---
description: >-
  Growing surplus, incentivizing veANGLE holders, and offering yield to the Core
  module's liquidity providers
---

# 📈 Lending Strategies - Yield on Reserves

## 🔎 TL;DR

- Angle Core module earns interest on the reserves it holds by lending it to other platforms.
- To do so, it relies on strategies which decide how much and in which protocols reserves should be placed.
- This mechanism is modular: there can be multiple strategies for a single collateral, each interacting with multiple platforms.
- Strategies are what enable the Core module to offer higher yield to SLPs than what they would get by lending directly to other protocol.

## 💡 Rationale

Lending a fraction of the reserves to other lending platforms is part of what makes the Core module attractive to Standard Liquidity Providers. By lending reserves, it can at the same time offer interest to Standard Liquidity Providers, accumulate some reserves, and incentivize veANGLE holders.

The distribution of interest between SLPs, veANGLE holders, and reserves, is dictated by two parameters that can be found in [Angle Analytics](https://analytics.angle.money). More information in the [SLPs FAQ page](standard-liquidity-providers/faq-slps.md#do-slps-get-all-transaction-fees-and-lending-returns-from-the-protocol).

![Angle Strategies Flow](../.gitbook/assets/angle-strategies-flow.jpg)

## 🎨 Design

The design of that is inspired by what [Yearn](https://yearn.finance) does. Angle Core module relies on strategies, that themselves use Lender's contracts interacting with lending and other yield farming protocols.

Just like on Yearn, new strategies to get some yield on the Core module's collateral can be added along the way by governance votes. Each strategy can also support multiple lending platforms or protocols.

Each collateral for each stablecoin has its set of strategies to get some yield on it. For instance, for a agEUR stablecoin backed by USDC and DAI, we may have for the USDC collateral a single strategy trying to always optimize to get the best APY between Compound and Aave, and for the DAI stablecoin two strategies, one that just consists in lending to Compound and one that consists in optimizing between Aave and Euler.

Since gas cost is quite high on Ethereum, users minting and burning, SLPs depositing and withdrawing, as well as HAs opening and closing positions never interact directly with strategy contracts. When they send or withdraw collateral to Angle Core module, their collateral goes or is taken from its reserves, and it is not directly lent or withdrawn from strategies.

The way collateral is lent or withdrawn from strategies and their corresponding lending platforms is through keepers calling the `harvest`function to withdraw or lend collateral to strategies.

{% hint style="info" %}
It was preferred to get inspiration from Yearn rather than using Yearn directly in order to keep full control on the reserves and to remove a third party integration which would have taken up some fees.
{% endhint %}

## 💹 Debt Ratio

For each strategy associated to a collateral, it is possible to compute a debt ratio that corresponds to the ratio between what has been lent and the total amount of collateral in the pool (what's kept in reserves + what has been lent across all strategies). Each strategy has its target debt ratio. Below this debt ratio, keepers can call the `harvest` function to give more collateral to the strategy and above this debt ratio, collateral can be withdrawn from the strategy and hence from the corresponding lending platforms.

{% hint style="info" %}
Keepers cannot choose the amount they lend or withdraw from lending platforms: this is automatically computed using the strategy's target debt ratio at each call to `harvest.`
{% endhint %}

## ✖️ Multiplier Effect For SLPs

Thanks to the lending strategies, SLPs get rewards not only from their capital, but also from that of Users and HAs. This allows them to get potentially higher yield than if they were farming only with their capital. More info in the SLP page below.

{% content-ref url="standard-liquidity-providers/" %}
[standard-liquidity-providers](standard-liquidity-providers/)
{% endcontent-ref %}

## 🤓 Strategies details

Here we describe strategies currently used for the USDC, DAI, FRAX and wETH backing the agEUR issued through the Core module on Ethereum.

### Base strategy

The first strategy implemented simply consisted in optimizing lending between Compound, Aave and Euler, and putting everything in the platform with the highest APY possible.

Following [this governance vote](https://snapshot.org/#/anglegovernance.eth/proposal/0xb1b4d98c080ec587b2563a6aaa6f854e0a42ce6881f61bced62cf9fa8ae42898), this strategy was updated for USDC and DAI to an improved model that consists in splitting funds between the different lending platforms supported following an optimal ratio computed off-chain. This was designed to maximize the global APR earned by the strategy while keeping strategy updates permissionless: anyone can suggest an allocation of funds between lending platforms, the strategy just checks whether suggested allocations improve the current APR, and adjusts in case of.

![Improved Optimizer APR Strategy](../.gitbook/assets/Optimizer-APR-Strategy-V2.jpg)

### Folding strategies

Angle Core module also relies on more advanced **folding** strategies. The idea behind folding is to lend capital, borrow from it and then re-supply the borrowed assets in order to profit from governance tokens rewards which can offset the borrowing cost.

Usually, folding strategies in DeFi simply target a specific leverage for which teams are confident enough, and know they can make an additional profit. However this does not guarantee a profit in all cases, can be sub-optimal, and most of all requires human intervention and monitoring.

**Angle folding strategy automatically compute the optimal quantity of assets to borrow and re-supply on-chain to maximize its yield.**

This makes it both more efficient that other folding strategies, and a clear improvement compared to the base strategy.

### stETH strategy

The strategy used for ETH is the StETHAcc strategy forked from Yearn [here](https://github.com/Grandthrax/yearn-steth-acc/blob/master/contracts/Strategy.sol). It buys stETH from Lido or Curve stETH/ETH pool depending on where it's cheaper, and then earns the stETH yield. stETH is exchanged to ETH when needed through the Curve pool.

The contract can be found [here](https://github.com/AngleProtocol/angle-core/blob/main/contracts/strategies/StrategyStETHAcc.sol).

{% hint style="info" %}
The state of the strategies used for different collateral types can be tracked on the [Angle Analytics](https://analytics.angle.money), or on the [developers documentation](https://developers.angle.money/overview/smart-contracts/mainnet-contracts).
{% endhint %}
