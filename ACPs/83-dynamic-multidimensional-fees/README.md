```text
ACP: 83
Title: Dynamic multidimensional fees for P-chain and X-chain
Author(s): Alberto Benegiamo <https://github.com/abi87>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/89
Status: Proposed
Track: Standards
```

## Abstract

Introduce a dynamic and multidimensional fees scheme for the P-chain and X-chain.

Dynamic fees helps to preserve the stability of the chain as it provides a feedback mechanism that increases the cost of resources when the network operates above its target utilization.

Multidimensional fees ensures that high demand for orthogonal resources does not drive up the price of underutilized resources. For example, networks provide and consume orthogonal resources including, but not limited to, bandwidth, chain state, read/write throughput, and CPU. By independently metering each resource, they can be granularly priced and stay closer to optimal resource utilization.


## Motivation

The P-Chain and X-Chain currently have fixed fees and in some cases those fees are fixed to zero.

This makes transaction issuance predictable, but does not provide feedback mechanism to preserve chain stability under high load. In contrast, the C-Chain, which has the highest and most regular load among the chains on the Primary Network, already supports dynamic fees. This ACP proposes to introduce a similar dynamic fee mechanism for the P-Chain and X-Chain to further improve the Primary Network's stability and resilience under load.

However, unlike the C-Chain, we propose a multidimensional fee scheme with an exponential update rule for each fee dimension. The [HyperSDK](https://github.com/ava-labs/hypersdk) already utilizes a multidimensional fee scheme with optional priority fees and its efficiency is backed by [academic research](https://arxiv.org/abs/2208.07919).

Finally we will split the fees into two parts: there will be a `base fee`, calculated by the network and updated block by block, which must be paid by transactions to be included in a block. Moreover there will be an optional `priority fee` to be paid on top of the `base fee` which may be included by the transaction issuer to speed up inclusion.

## Specification

We introduce the multidimensional scheme first and then how to apply the dynamic fee update rule for each fee dimension. Finally we list the new block verification rules, valid once the new fee scheme activates.

### Multidimensional scheme components

We define four fee dimensions, `Bandwidth`, `DBReads`, `DBWrites`, `Compute`, to describe transactions complexity. In more details:

- `Bandwidth` measures the transaction size in bytes, as encoded by the Avalanchego codec. Byte length is a proxy for the network resources needed to disseminate the transaction.
- `DBReads` measures the number of DB reads needed to verify the transactions. DB reads include UTXOs reads and any other state quantity relevant for the specific transaction.
- `DBWrites` measures the number of DB writes following the transaction verification. DB writes include UTXOs generated as outputs of the transactions and any other state quantity relevant for the specific transaction.
- `Compute` measures the number of signatures to be verified, including UTXOs ones and those related to authorization of specific operations.

For each fee dimension $i$, we define:

- *fee rate* $r_i$ as the price, denominated in AVAX, to be paid for a transaction with complexity $u_i$ along the fee dimension $i$.
- *base fee* as the minimal fee needed to accept a transaction. Base fee is given be the formula
$$base \  fee = \sum_{i=0}^3 r_i \times u_i$$
- *priority fee* as an optional fee paid on top of the base fee to speed up the transaction inclusion in a block.

### Dynamic scheme components

Fee rates are updated in time, to allow a fee increase when network is getting congested. Each new block is a potential source of congestion, as its transactions carry complexity that each validator must process to verify and eventually accept the block. The more complexity carries a block, and the more rapidly blocks are produced, the higher the congestion.  

We seek a scheme that rapidly increases the fees when blocks complexity goes above a defined threshold and that equally rapidly decreases the fees once complexity goes down (because blocks carry less/simpler transactions, or because they are produced more slowly).

Suppose that a block $B$ is the current chain tip. $B$ has the following features:

- $t$ is its timestamp.
- $B_i$ is the cumulated complexity of all its transactions along the fee dimension $i$.
- $r_{i,t}$ is the fee rate applied to all of its transactions along the fee dimension $i$.

A new block is incoming, whose timestamp is $t + \Delta T$.  
We wish to maintain a *target complexity rate* $T_i$, i.e. we would want to process every second a block whose cumulated complexity is $T_i$. Any complexity more than that causes some congestion that we want to penalize via fees; any congestion below $T_i$ causes the network to be under utilized.

The fee rates $r_{i,t+\Delta T}$ applied to the incoming blocks are defined with the following *update formula*:

$$r_{i,t+\Delta T} = max \bigl( r^{min}_i, e^{k_i \frac{B_i-T_i \times \Delta T}{T_i \times \Delta T}} \bigr)$$

where $r^{min}_i$ and $k_i$ are the minimal fee rate allowed and the update constants along dimension $i$ respectivelly.

The update formula guarantees that fee rates increase if parent block is complex (large $B_i$) and if blocks are emitted rapidly (small $\Delta T$). Simmetrically fee rates decrease if parent block is less complex and if blocks are produced less frequently. We introduce a minimal, non-zero fee rate, to avoid fee rates reaching zero, which would cause any subsequent fee to be zero as well.  
The update formula has a few paramenters to be tuned, independently, for each fee dimension. We defer discussion about tuning to the [implementation section](#tuning-the-update-formula).

## Block verification rules

Upon activation of the dynamic multidimensional fees scheme we modify block processing as follows:

- **Bound block complexity**. For each fee dimension $i$, we define a *maximal block complexity* $C_i$. A block is only valid if the block complexity $B_i$ is less than the maximum block complexity: $B_i \leq C_i$.
- **Verify transaction fee**. When verifying each transaction in a block, we confirm that it can cover its own base fee. Note that both base fee and optional priority fees are burned.

## User Experience

### How will the wallets estimate the fees?

AvalancheGo nodes will provide new APIs exposing the current and expected fee rates, as they are likely to change block by block. Wallets can then use the fees rates to select UTXOs to pay the transaction fees. Moreover Avalanchego implementation proposed above offers a `fees.Calculator` struct that can be reused by wallets and downstreams to evaluate calculate fees.

### How will wallets be able to re-issue Txs at a higher fee?

Wallets should be able to simply re-issue the transaction since current AvalancheGo implementation drops mempool transactions whose fee rate is lower than current one. More specifically, a transaction may be valid the moment it enters the mempool and it won’t be re-verified as long as it stays in there. However, as soon as the transaction is selected to be included in the next block, it is re-verified against the latest preferred tip. If fees are not enough by this time, the transaction is dropped and the wallet can simply re-issue it at a higher fee, or wait for the fee rate to go down. Note that priority fees offer some buffer space against an increase in the fee rate. A transaction paying just the base fee will be evicted from the mempool in the face of a fee rate increase, while a transaction paying some extra priority fee may have enough buffer room to stay valid after some amount of fee increase.

### How does priority fees guarantee a faster block inclusion?

AvalancheGo mempool will be restructured to order transactions by priority fees. Transactions paying priority fees will be selected for block inclusion first, without violating any spend dependency.

## Backwards Compatibility

Modifying the fee scheme for P-Chain and X-Chain requires a mandatory upgrade for activation. Moreover, wallets must be modified to properly handle the new fee scheme once activated.

## Reference Implementation

The implementation is split across multiple PRs:

- P-Chain work is tracked in this issue: <https://github.com/ava-labs/avalanchego/issues/2707>
- X-Chain work is tracked in this issue: <https://github.com/ava-labs/avalanchego/issues/2708>

A very important implementation step is tuning the update formula parameters for each chain and each fee dimension. We show here the principles we followed for tuning and a simulation based on historical data.

### Tuning the update formula

The basic idea is to measure the complexity of blocks already accepted and derive the parameters from it. You can find the historical data in [this repo](https://github.com/abi87/complexities).  
To simplify the exposition I am purposefully ignoring chain specifics (like P-chain proposal blocks). We can account for chain specifics while processing the historical data. Here are the principles:

- **Target block complexity rate**: calculate the distribution of block complexity and pick a high enough quantile.
- **Max block complexity**: this is probably the trickiest parameter to set.
Historically we had [pretty big transactions](https://subnets.avax.network/p-chain/tx/27pjHPRCvd3zaoQUYMesqtkVfZ188uP93zetNSqk3kSH1WjED1) (more than $1.000$ referenced utxos). Setting a max block complexity so high that these big transactions are allowed is akin to setting no complexity cap.
On the other side, we still want to allow, even encourage, UTXO consolidation, so we may want to allow transactions [like this](https://subnets.avax.network/p-chain/tx/2LxyHzbi2AGJ4GAcHXth6pj5DwVLWeVmog2SAfh4WrqSBdENhV).
A principled way to set max block complexity may be the following:
  - calculate the target block complexity rate (see previous point)
  - calculate the median time elapsed among consecutive blocks
  - The product of these two quantities should gives us something like a target block complexity.
  - Set the max block complexity to say $\times 50$ the target value.
- **Update coefficient**: this is the $k_i$ parameter in the exponential fee update. I suggest we size it as follows:
  - Find the largest historical peak, i.e. the sequence of consecutive blocks which contained the most complexity in the shortest period of time
  - Tune $k_i$ so that it would cause a $\times 10000$ increase in the fee rate for such a peak. This increase would push fees from the milliAVAX we normally pay under stable network condition up to tens of AVAX.
- **Initial fee rates** : this is the initial base fee set at activation.
 We could size them so that transactions fees do not change very much with respect to the currently fixes values.
 Note that we do currently support zero fees transactions, which are not allowed in the new fee scheme.  Zero fee transactions would immediately see the largest increase when dynamic fees kick in. Still we could make the change as small as possible
- **Minimal fee rates** :  these are required since we cannot have zero fee rates in our dynamic fees proposal.
  We update fee rates via a multiplicative process so as soon as a fee rate is zero all sequent fee rates would be zero.
  We could set minimal fee rates to equal to initial fee rates or to a fraction of them.

We simulate below how the update formula would behave on an peak period from Avalanche mainnet.

<p align="center">
  <img src=./complexities.png />
  <img src=./fee.png />
</p>

Figure 1 shows a peak period, starting with block [wqKJcvEv86TBpmJY2pAY7X65hzqJr3VnHriGh4oiAktWx5qT1](https://subnets.avax.network/p-chain/block/wqKJcvEv86TBpmJY2pAY7X65hzqJr3VnHriGh4oiAktWx5qT1) and going for roughly 30 blocks. We only show `Bandwidth` for clarity, but other fees dimensions have similar behaviour.
The network load is much larger than target and sustained.  
Figure 2 show the fee dynamic in response to the peak: fees scale up from a few milliAVAX up to around 25 AVAX. Moreover as soon as the peak is over, and complexity goes back to the target value, fees are reduced very rapidly.

## Security Considerations

The new fee scheme is expected to help network stability as it offers economic incentives to users to hold transactions issuance in times of high load. While fees are expected to remain generally low when the system is not loaded, a sudden load increase, with fuller blocks, would push the dynamic fees algo to increase fee rates. The increase is expected to continue until the load is reduced. Load reduction happens by both dropping unconfirmed transactions whose fee-rate is not sufficient anymore and by pushing users that optimize their transactions costs to delay transaction issuance until the fee rate goes down to an acceptable level.  
Note finally that the exponential fee update mechanism detailed above is [proven](https://ethresear.ch/t/multidimensional-eip-1559/11651) to be robust against strategic behaviors of users delaying transactions issuance and then suddenly push a bulk of transactions once the fee rate is low enough.

## Acknowledgements

Thanks to @StephenButtolph @patrick-ogrady and @dhrubabasu for their feedback on these ideas.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
