| ACP | 176 |
| :- | :- |
| **Title** | Dynamic EVM Gas Limits and Price Discovery Updates |
| **Author(s)** | Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)), Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

Proposes that the C-Chain and Subnet-EVM chains adopt a dynamic fee mechanism similar to the one [introduced on the P-Chain as part of ACP-103](../103-dynamic-fees/README.md), with modifications to allow for block proposers (i.e. validators) to dynamically adjust the target gas consumption per unit time.

## Motivation

Currently, the C-Chain has a static gas target of [15,000,000 gas](https://github.com/ava-labs/coreth/blob/39ec874505b42a44e452b8809a2cc6d09098e84e/params/avalanche_params.go#L32) per [10 second rolling window](https://github.com/ava-labs/coreth/blob/39ec874505b42a44e452b8809a2cc6d09098e84e/params/avalanche_params.go#L36), and uses a modified version of the [EIP-1559](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1559.md) dynamic fee mechanism to adjust the base fee of blocks based on the gas consumed in the previously 10 second window. This has two notable drawbacks:

1. The windower mechanism used to determine the base fee of blocks can lead to outsized spikes in the gas price when there is a large block. This is because after a large block that uses all of its gas limit, blocks that follow in the same window continue to result in increased gas prices even if they are relatively small blocks that are under the target gas consumption.
2. The static gas target necessitates a required network upgrade in order to modify. This is cumbersome and makes it difficult for the network to adjust its capacity in response to performance optimization or hardware requirement increases.

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

Prior to the activation of this mechanism, $T$ is initialized to a prior estimate of the network's desired target gas consumption per second, which must be a positive integer.

After the execution of block $b$, $T$ can be adjusted up or down:

$$\frac{-T}{D} \le targetChange \le \frac{T}{D}$$
$$T = max(P, T + targetChange)$$

Where:
 - $D$ is the target gas consumption rate bound divisor
 - $P$ is the global minimum allowed target gas consumption rate for the network, which must be at least $D$.
 - $targetChange$ is set by the builder of block $b$ within the valid range. 
 
 If $|targetChange| > \frac{T}{D}$, block $b$ is considered invalid. Block builders (i.e. validators), may set their desired value for $T$ (i.e. `desired_rate`) in their configuration, and $targetChange$ can then be calculated for blocks that they build according to:

  ```python
  # Calculates the amount by which to adjust the target gas consumed per second
  def calc_gas_consumption_rate_change(current_rate: int, desired_rate: int) -> int:
    max_change = current_rate / target_gas_consumption_rate_bound_divisor
    if desired_rate > current_rate:
        return min(desired_rate - current_rate, max_change)
    else:
        return -1 * min(current_rate - desired_rate, max_change)
  ```

As the value of $T$ dynamically adjusts after the execution of each block, the value of $R$ (capacity added per second) is also updated such that:

$$R = T*2$$

This ensures that the gas price remains equally reactive in either direction depending on how far the actual gas consumption rate is from the target, whether it is above or below.

The value of $C$ must also adjust proportionately, so we set:

$$C = R*10$$

This means that the maximum stored gas capacity would be reached after 10 seconds where no blocks have been accepted.

Note that the values of $T$, $R$, and $C$ are updated **after** the execution of block $b$, which means they only take effect in determining the gas price of block $b+1$. The change to each of these values in block $b$ does not effect the gas price for transaction included in block $b$ itself.

Allowing block builders to adjust the target gas consumption rate in blocks that they produce makes it such that the effect target gas consumption rate should loosely converge over time around the stake-weighted average value set by validators of the network. This is because the number of blocks each validator produces is proportional to their stake weight. This means that an individual validator's effect on the resulting target gas consumption for the network is proportional to their stake weight.

Currently, it is not valid for a block to contain zero transactions since it would not have any effect and would be a waste of resources to accept into the blockchain. However, with this change, it is possible for blocks with zero transactions to still effect value of $T$. To ensure that validators are able to help influence the value of $T$ even if there are no transactions to be included at the time they are proposing a block, blocks with no transactions that alter the value of $T$ will now be considered valid. This makes it such that validators would not need to create and include a no-op transaction just be able to produce a block to alter the current value of $T$ if there are no pending transactions when it is their turn to propose a block.

### Configuration Parameters
As noted above, the gas price determination mechanism depends on the values of $T$, $M$, $K$, $C$, and $R$ to be set as parameters. $T$ is adjusted dynamically from its initial value based on $D$ and $P$, and the values of $R$ and $C$ are derived from $T$. 

Parameters at activation on the C-Chain are:

<div align="center">

| Parameter | Description | P-Chain Configuration|
| - | - | - |
| $T$ | initial target gas consumed per second | $1,500,000$ |
| $M$ | minimum gas price | $1*10^{-18}$ AVAX  |
| $K$ | gas price update constant | $2,164,043$ |
| $D$ | target change bound divisor | $1024$ |
| $P$ | minimum target gas consumption per second | $1,000,000$

</div>

$T$ was chosen as the current target gas consumption rate on the C-Chain. Avalanche L1s may change this value to match their current gas consumption rate if they would like.

$M$ was chosen as the minimum possible denomination of the native EVM asset, such that the gas price will be more likely to consistently be in a range of price discovery. The price discovery mechanism has already been battle tested on the P-Chain (and prior to that on Ethereum for blob gas prices as defined by EIP-4844), giving confidence that it will correct react to any increase in network usage in order to prevent a DOS attack.

$K$ was chosen such that at sustained maximum capacity ($T*2$ gas/second), the fee rate will double every ~30 seconds.

$D$ was chosen to match the [gas limit bound divisor that Ethereum currently uses](https://github.com/ethereum/go-ethereum/blob/52766bedb9316cd6cddacbb282809e3bdfba143e/params/protocol_params.go#L26) to limit the amount that validators can change their execution layer gas limit.

$P$ was chosen as a safe bound on the minimum target gas usage that is lower than the current target gas consumption rate on the C-Chain today. The target gas consumption rate will not move towards $P$ unless the majority of stake weight of the network specifies $P$ as their desired gas consumption rate target.

## Backwards Compatibility

The changes proposed in this ACP require a required network upgrade in order to take effect. Prior to its activation, the current gas limit and price discovery mechanisms will continue to be used. Its activation should have relatively minor compatibility effects on any developer tooling. Notably, transaction formats, and thus wallets, are not impacted. After its activation, given that the value of $C$ is dynamically adjusted, the maximum possible gas consumed by an individual block, and thus maximum possible consumed by an individual transaction, will also dynamically adjust. This can affect that maximum size of smart contract that can be deployed at any given time, but should not have significant practical impact.

## Reference Implementation

A full reference implementation has not been provided yet. It will be provided once this ACP is considered Implementable.

## Security Considerations

This ACP changes the mechanism for determining the gas price on Avalanche EVM chains. The gas price is meant to adopt dynamically to respond to changes in demand for using the chain. If it does not react as expected, the chain could be at risk for a DOS attack (if the usage price is too low), or over charge users during period of low activity. This price discovery mechanism has already been employed on the P-Chain, but should again be thouroughly tested for use on the C-Chain prior to activation on the Avalanche Mainnet.

Further, this ACP also introduces a mechanism for validators to change the gas limit of the C-Chain. If this limit is set too high, it is possible that validator nodes will not be able to keep up in the processing of blocks. An upper bound on the maximum possible gas limit could be considered to try to mitigate this risk, though it would then take further required network upgrades to scale the network past that limit. 

## Acknowledgments

Thanks to the following non-exhaustive list of individuals for input, discussion, and feedback on this ACP.
- [Emin Gün Sirer](https://x.com/el33th4xor)
- [Luigi D'Onorio DeMeo](https://x.com/luigidemeo)
- [Darioush Jalali](https://github.com/darioush)
- [Aaron Buchwald](https://github.com/aaronbuchwald)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
