# ACP-256: Enhance AVAX Economic Sustainability by Gradually Reducing Staking Emissions

| ACP | 256 |
| :--- | :--- |
| **Title** | Enhance AVAX Economic Sustainability by Gradually Reducing Staking Emissions |
| **Author(s)** | [yoshiyoshi](https://github.com/yy00shi)  |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/)) |
| **Track** | Standards |

---

## Abstract

This proposal outlines a strategy to enhance the long-term economic sustainability and security of the Avalanche Network by gradually reducing AVAX staking rewards. We propose a phased reduction of the maximum staking yield over an 12-month period, ultimately decreasing new emissions from staking by approximately 50%. This measure aims to reduce annual AVAX emissions significantly, estimated to be a reduction of over 6.9 million AVAX per year (equivalent to over $120M at recent price levels), thereby decreasing inflationary pressure, reinforcing the long-term value of AVAX, and signaling the network's maturation from a high-incentive growth phase to a self-sustaining economic model.

## Motivation & Rationale

The Avalanche Network has successfully achieved a high degree of decentralization and security, thanks in large part to the robust incentives provided to validators since its inception. The current staking yield has been instrumental in bootstrapping the network to its current state.

However, as the ecosystem matures, it is prudent to evolve its economic policy to ensure long-term sustainability. High emission rates, while effective for growth, perpetually introduce new supply, creating inflationary pressure that can dilute value for all token holders.

The primary motivations for this proposal are:

1.  **Long-Term Economic Sustainability:** To secure the future of the network, we must transition towards a model where security is maintained with minimal necessary issuance. Reducing emissions ensures the scarcity and long-term value proposition of AVAX.
2.  **Decreased Sell Pressure:** A significant portion of staking rewards are often sold on the open market. Halving these emissions will systematically reduce this structural sell pressure, contributing to a healthier market dynamic for AVAX.
3.  **Signal of Network Maturity:** This proposal signals a confident shift from a subsidized growth model to one where the network's utility, transaction fees, and ecosystem value are the primary drivers of security and demand. This is a key step in attracting long-term, fundamental-driven investors.
4.  **Aligning with Real Yield:** As the Avalanche ecosystem grows, transaction fees and subnet demand will constitute a larger portion of validator revenue. Reducing reliance on inflationary yield positions the network to thrive on its own economic activity.

## Specification: The Proposed Change

The current staking reward mechanism in Avalanche is governed by a dynamic function. This proposal seeks to adjust the parameters of this function in a predictable, phased manner.

The goal is to reduce the maximum potential staking yield from its current level of approximately **~7.0%** (at ~50% Staking Ratio) to a new target of **~3.5%**. This will be achieved over four distinct phases, with a four-month interval between each phase to allow the market and stakers to adjust smoothly.

### Phased Reduction Schedule

| Phase | Timeline | Action | Target Max Yield |
| :--- | :--- | :--- | :--- |
| **0** | Current State | No Change | ~7.0% |
| **1** | Activation (e.g., Q4 2025) | Reduce max yield by ~25% from the starting point. | ~6.125% |
| **2** | Phase 1 + 4 Months | Reduce max yield by an additional ~0.875%. | ~5.25% |
| **3** | Phase 2 + 4 Months | Reduce max yield by an additional ~0.875%. | ~4.375% |
| **4** | Phase 3 + 4 Months | Reduce max yield by an additional ~0.875%. | ~3.5% |

*Note: The actual yield received by stakers is dynamic based on staking duration and total amount staked. The figures above represent the yield typically achievable under average network conditions.*

### Technical Implementation

The implementation requires a modification to the `reward` function within the Avalanche protocol's Platform Chain (P-Chain). This change can be enacted via a standard network upgrade activation (hard fork), with the phased adjustments programmed to trigger automatically at the specified future timestamps or block heights.

## Impact Analysis

### Economic Impact

* **Emission Reduction:** Based on a circulating staked supply of ~422 million AVAX, a 50% reduction in yield translates to a decrease of approximately **~7 million AVAX** in new emissions annually.
* **Value Preservation:** At a hypothetical price of $25/AVAX, this represents a reduction of over **$175 million** in annual inflationary pressure.

### Risk Assessment & Mitigations

1.  **Risk: Validator Churn & Reduced Security**
    * **Concern:** A lower yield could disincentivize validators, leading some to unstake and potentially reducing the network's total staked value.
    * **Mitigation:**
        * **Gradual Phasing:** The 12-month timeline is specifically designed to prevent an economic shock. It allows stakers to plan and the market to price in the changes gradually.
        * **Competitive Yield:** A final yield of ~3.5% remains competitive and attractive compared to the "real" (inflation-adjusted) yields on other major PoS networks and traditional finance benchmarks.
        * **Focus on Total Return:** The reduction in inflation benefits *all* holders, including validators, by preserving the purchasing power of their principal stake. The total economic return (yield + principal value appreciation) may be superior under a lower inflation regime.

2.  **Risk: Negative Market Sentiment**
    * **Concern:** The proposal could be misconstrued as a negative development, harming short-term price action.
    * **Mitigation:**
        * **Clear Communication:** The rationale must be communicated clearly, focusing on the long-term benefits of economic sustainability and network maturity. This ACP draft is the first step in that process.
        * **Positive Framing:** This is not a "cut" but a "strategic transition" to a more robust economic model that has been successfully navigated by other major protocols like Bitcoin (via halvings).

## Next Steps

We submit this draft to the Avalanche community for discussion and feedback. We encourage all community members, validators, and developers to review the proposal and contribute to the conversation. Following a period of public discussion, the aim is to formalize this ACP for a governance vote.

---
**Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
