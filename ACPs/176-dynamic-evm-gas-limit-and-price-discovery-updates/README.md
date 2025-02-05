| ACP | 176 |
| :- | :- |
| **Title** | Dynamic EVM Gas Limits and Price Discovery Updates |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/178)) |
| **Track** | Standards |

## Abstract

Proposes that the C-Chain and Subnet-EVM chains adopt a dynamic fee mechanism similar to the one [introduced on the P-Chain as part of ACP-103](../103-dynamic-fees/README.md), with modifications to allow for block proposers (i.e. validators) to dynamically adjust the target gas consumption per unit time.

## Motivation

Currently, the C-Chain has a static gas target of [15,000,000 gas](https://github.com/ava-labs/coreth/blob/39ec874505b42a44e452b8809a2cc6d09098e84e/params/avalanche_params.go#L32) per [10 second rolling window](https://github.com/ava-labs/coreth/blob/39ec874505b42a44e452b8809a2cc6d09098e84e/params/avalanche_params.go#L36), and uses a modified version of the [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) dynamic fee mechanism to adjust the base fee of blocks based on the gas consumed in the previous 10 second window. This has two notable drawbacks:

1. The windower mechanism used to determine the base fee of blocks can lead to outsized spikes in the gas price when there is a large block. This is because after a large block that uses all of its gas limit, blocks that follow in the same window continue to result in increased gas prices even if they are relatively small blocks that are under the target gas consumption.
2. The static gas target necessitates a required network upgrade in order to modify. This is cumbersome and makes it difficult for the network to adjust its capacity in response to performance optimizations or hardware requirement increases.

To better position Avalanche EVM chains, including the C-Chain, to be able to handle future increases in load, we propose replacing the above mechanism with one that better handles blocks that consume a large amount of gas, and that allows for validators to dynamically adjust the target rate of consumption.

## Specification

### Gas Price Determination

The mechanism to determine the base fee of a block is the same as the one used in ACP-103 to determine the gas price of a block on the P-Chain. This mechanism calculates the gas price for a given block $b$ based on the following parameters:

<div align="center">

| | |
|---|---|
| $T$ | the target gas consumed per second |
| $M$ | minimum gas price |
| $K$ | gas price update constant |
| $C$ | maximum gas capacity |
| $R$ | gas capacity added per second |

</div>

The full specification of this mechanism can be found [here](../103-dynamic-fees/README.md).

### Making $T$ Dynamic

As noted above, the gas price determination mechanism relies on a target gas consumption per second, $T$, in order to calculate the gas price for a given block. $T$ will be adjusted dynamically according to the following specification.

Let $q$ be a non-negative integer that is initialized to 0 upon activation of this mechanism. Let the target gas consumption per second be expressed as:

$$T = P \cdot e^{\frac{q}{D}}$$

where $P$ is the global minimum allowed target gas consumption rate for the network, and $D$ is a constant that helps control the rate of change of the target gas consumption.

After the execution of transactions in block $b$, the value of $q$ can be increased or decreased up to $Q$. It must be the case that $\left|\Delta q\right| \leq Q$, or block $b$ is considered invalid. The amount by which $q$ changes after executing block $b$ is specified by the block builder.

Block builders (i.e. validators), may set their desired value for $T$ (i.e. their desired gas consumption rate) in their configuration, and their desired value for $q$ can then be calculated as:

$$q_{desired} = D \cdot ln\left(\frac{T_{desired}}{P}\right)$$

Note that since $q_{desired}$ is only used locally and can be different for each node, it is safe for implementations to approximate the value of $ln\left(\frac{T_{desired}}{P}\right)$, and round the resulting value to the nearest integer.

When building a block, builders can calculate their next preferred value for $q$ based on the network's current value (`q_current`) according to:

  ```python
  # Calculates a node's new desired value for q given for a given block
  def calc_next_q(q_current: int, q_desired: int, max_change: int) -> int:
    if q_desired > q_current:
        return q_current + min(q_desired - q_current, max_change)
    else:
        return q_current - min(q_current - q_desired, max_change)
  ```

As $q$ is updated after the execution of transactions within the block, $T$ is also updated such that $T = P \cdot e^{\frac{q}{D}}$ at all times. As the value of $T$ adjusts, the value of $R$ (capacity added per second) is also updated such that:

$$R = T \cdot 2$$

This ensures that the gas price can increase and decrease at the same rate.

The value of $C$ must also adjust proportionately, so we set:

$$C = R \cdot 10$$

This means that the maximum stored gas capacity would be reached after 10 seconds where no blocks have been accepted.

In order to keep roughly constant the time it takes for the gas price to double at sustained maximum network capacity usage, the value of $K$ used in the gas price determination mechanism must be updated proportionally to $T$ such that:

$$K = T \cdot 43$$

In order to have the gas price not be directly impacted by the change in $K$, we also update $x$ (excess gas consumption) proportionally. When updating $x$ after executing a block, instead of setting $x = x + G$ as specified in ACP-103, we set:

$$x_{n+1} = (x + G) \cdot \frac{K_{n+1}}{K_{n}}$$

Note that the value of $q$ (and thus also $T$, $R$, $C$, $K$, and $x$) are updated **after** the execution of block $b$, which means they only take effect in determining the gas price of block $b+1$. The change to each of these values in block $b$ does not effect the gas price for transactions included in block $b$ itself.

Allowing block builders to adjust the target gas consumption rate in blocks that they produce makes it such that the effective target gas consumption rate should converge over time to the point where 50% of the voting stake weight wants it increased and 50% of the voting stake weight wants it decreased. This is because the number of blocks each validator produces is proportional to their stake weight.

As noted in ACP-103, the maximum gas consumed in a given period of time $\Delta{t}$, is $r + R \cdot \Delta{t}$, where $r$ is the remaining gas capacity at the end of previous block execution. The upper bound across all $\Delta{t}$ is $C + R \cdot \Delta{t}$. Phrased differently, the maximum amount of gas that can be consumed by any given block $b$ is:

$$gasLimit_{b} = min(r + R \cdot \Delta{t}, C)$$ 

Currently, it is not valid for a block to contain zero transactions since it would not have any effect and would be a waste of resources to accept into the blockchain. However, with this change, it is possible for blocks with zero transactions to still effect value of $T$. To ensure that validators are able to help influence the value of $T$ even if there are no transactions to be included at the time they are proposing a block, blocks with no transactions that alter the value of $T$ will now be considered valid. This makes it such that validators would not need to create and include a no-op transaction just be able to produce a block to alter the current value of $T$ if there are no pending transactions when it is their turn to propose a block.

### Configuration Parameters

As noted above, the gas price determination mechanism depends on the values of $T$, $M$, $K$, $C$, and $R$ to be set as parameters. $T$ is adjusted dynamically from its initial value based on $D$ and $P$, and the values of $R$ and $C$ are derived from $T$. 

Parameters at activation on the C-Chain are:

<div align="center">

| Parameter | Description | C-Chain Configuration|
| - | - | - |
| $P$ | minimum target gas consumption per second | $1,000,000$ |
| $D$ | target gas consumption rate update constant | $2^{25}$ |
| $Q$ | target gas consumption rate update factor change limit | $2^{15}$ |
| $M$ | minimum gas price | $1*10^{-18}$ AVAX  |
| $K$ | initial gas price update factor | $43,000,000$ |

</div>

$P$ was chosen as a safe bound on the minimum target gas usage on the C-Chain. The current gas target of the C-Chain is $1,500,000$ per second. The target gas consumption rate will only stay at $P$ if the majority of stake weight of the network specifies $P$ as their desired gas consumption rate target. 

$D$ and $Q$ were chosen to give each block builder the ability to adjust the value of $T$ by roughly $\frac{1}{1024}$ of its current value, which matches the [gas limit bound divisor that Ethereum currently uses](https://github.com/ethereum/go-ethereum/blob/52766bedb9316cd6cddacbb282809e3bdfba143e/params/protocol_params.go#L26) to limit the amount that validators can change the execution layer gas limit in a single block. $D$ and $Q$ were scaled up by a factor of $2^{15}$ to provide block builders more granularity in the adjustments to $T$ that they can make.

$M$ was chosen as the minimum possible denomination of the native EVM asset, such that the gas price will be more likely to consistently be in a range of price discovery. The price discovery mechanism has already been battle tested on the P-Chain (and prior to that on Ethereum for blob gas prices as defined by EIP-4844), giving confidence that it will correctly react to any increase in network usage in order to prevent a DOS attack.

$K$ was chosen such that at sustained maximum capacity ($T*2$ gas/second), the fee rate will double every ~29.8 seconds.

### Choosing $T_{desired}$

As mentioned above, this new mechanism allows for validators to specify their desired target gas consumption rate ($T_{desired}$) in their configuration, and the value that they set impacts the effective target gas consumption rate of the network over time. The higher the value of $T$, the more resources (storage, compute, etc) that are able to be used by the network. When choosing what value makes sense for them, validators should consider the resources that are required to properly support that level of gas consumption, the utility the network provides by having higher transaction per second throughput, and the stability of network should it reach that level of utilization.

While Avalanche Network Clients can set default configuration values for the desired target gas consumption rate, each validator can choose to set this value independently based on their own considerations.

## Backwards Compatibility

The changes proposed in this ACP require a required network upgrade in order to take effect. Prior to its activation, the current gas limit and price discovery mechanisms will continue to be used. Its activation should have relatively minor compatibility effects on any developer tooling. Notably, transaction formats, and thus wallets, are not impacted. After its activation, given that the value of $C$ is dynamically adjusted, the maximum possible gas consumed by an individual block, and thus maximum possible consumed by an individual transaction, will also dynamically adjust. The upper bound on the amount of gas consumed by a single transaction fluctuating means that transactions that are considered invalid at one time may be considered valid at a different point in time, and vice versa. While potentially unintuitive, as long as the minimum gas consumption rate is set sufficiently high this should not have significant practical impact, and is also currently the case on the Ethereum mainnet.

## Reference Implementation

A full reference implementation has not been provided yet. It will be provided once this ACP is considered Implementable.

## Security Considerations

This ACP changes the mechanism for determining the gas price on Avalanche EVM chains. The gas price is meant to adapt dynamically to respond to changes in demand for using the chain. If it does not react as expected, the chain could be at risk for a DOS attack (if the usage price is too low), or over charge users during period of low activity. This price discovery mechanism has already been employed on the P-Chain, but should again be thoroughly tested for use on the C-Chain prior to activation on the Avalanche Mainnet.

Further, this ACP also introduces a mechanism for validators to change the gas limit of the C-Chain. If this limit is set too high, it is possible that validator nodes will not be able to keep up in the processing of blocks. An upper bound on the maximum possible gas limit could be considered to try to mitigate this risk, though it would then take further required network upgrades to scale the network past that limit. 

## Acknowledgments

Thanks to the following non-exhaustive list of individuals for input, discussion, and feedback on this ACP.
- [Emin Gün Sirer](https://x.com/el33th4xor)
- [Luigi D'Onorio DeMeo](https://x.com/luigidemeo)
- [Darioush Jalali](https://github.com/darioush)
- [Aaron Buchwald](https://github.com/aaronbuchwald)
- [Geoff Stuart](https://github.com/geoff-vball)
- [Meag FitzGerald](https://github.com/meaghanfitzgerald)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
