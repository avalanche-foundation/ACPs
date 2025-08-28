| ACP | 226 |
| :- | :- |
| **Title** | Dynamic Minimum Block Times |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/228)) |
| **Track** | Standards |

## Abstract

Proposes replacing the current block production rate limiting mechanism on Avalanche EVM chains with a new mechanism where validators collectively and dynamically determine the minimum time between blocks.

## Motivation

Currently, Avalanche EVM chains employ a mechanism to limit the rate of block production by increasing the "block gas cost" that must be burned if blocks are produced more frequently than the target block rate specified for the chain. The block gas cost is paid by summing the "priority fee" amounts that all transactions included in the block collectively burn. This mechanism has a few notable suboptimal aspects:

1. There is no explicit minimum block delay time. Validators are capable of producing blocks as frequently as they would like by paying the additional fee, and too rapid block production could cause network stability issues.
1. The target block rate can only be changed in a required network upgrade, which makes updates difficult to coordinate and operationalize.
1. The target block rate can only be specified with 1-second granularity, which does not allow for configuring sub-second block times as performance improvements are made to make them feasible.

With the prospect of ACP-194 removing block execution from consensus and allowing for increases to the gas target through the dynamic ACP-176 mechanism, Avalanche EVM chains would be better suited by having a dynamic minimum block delay time denominated in milliseconds. This allows networks to ensure that blocks are never produced more frequently than the minimum block delay, and allows validators to dynamically influence the minimum block delay value by setting their preference.

## Specification

### Block Header Changes

Upon activation of this ACP, the `blockGasCost` field in block headers will be required to be set to 0. This means that no validation of the cumulative priority fee amounts of transactions within the block exceeding the block gas cost is required. Additionally, two new fields will be added to EVM block headers: `timestampMilliseconds` and `minimumBlockDelay`.

#### `timestampMilliseconds`

The canonical serialization and interpretation of EVM blocks already contains a block timestamp specified in seconds. Altering this would require deep changes to the EVM codebase, as well as cause breaking changes to tooling such as indexers and block explorers. Instead, a new field is added representing the unix timestamp in milliseconds. Header verification should verify the `block.timestamp` (in seconds) is aligned with the `block.timeMilliseconds`, more precisely: `block.timestampMilliseconds / 1000 == block.timestamp`.

Existing tools that do not need millisecond granularity do not need to parse the new field, which limits the amount of breaking changes.

The `timestampMilliseconds` field will be represented in block headers as a `uint64`.

#### `minimumBlockDelay`

The new `minimumBlockDelay` field in the block header encodes the minimum number of milliseconds that must pass before the next block is allowed to be accepted. Specifically, if block $B$ has a `minimumBlockDelay` of $d$, then the effective timestamp of block $B+1$ in milliseconds must be at least $d$ greater than the effective timestamp of block $B$ in milliseconds.

The `minimumBlockDelay` field will be represented in block headers as a `uint64`.

The value of `minimumBlockDelay` can be updated in each block according to a mechanism similar to the one used by ACP-176 to support dynamic gas targets. The mechanism is specified below.

### Dynamic `minimumBlockDelay` mechanism

The `minimumBlockDelay` can be defined as:

$$ m = M * e^{\frac{q}{D}} $$

Where:
- $M$ is the global minimum `minimumBlockDelay` value in milliseconds
- $q$ is a non-negative integer that is initialized upon the activation of this mechanism
- $D$ is a constant that helps control the rate of change of `minimumBlockDelay`

After the execution of transactions in block $b$, the value of $q$ can be increased or decreased by up to $Q$. It must be the case that $\left|\Delta q\right| \leq Q$, or block $b$ is considered invalid. The amount by which $q$ changes after executing block $b$ is specified by the block builder.

Block builders (i.e., validators) may set their desired value for $M$ (i.e., their desired `minimumBlockDelay`) in their configuration, and their desired value for $q$ can then be calculated as:

$$q_{desired} = D \cdot ln\left(\frac{M_{desired}}{M}\right)$$

Note that since $q_{desired}$ is only used locally and can be different for each node, it is safe for implementations to approximate the value of $ln\left(\frac{M_{desired}}{M}\right)$ and round the resulting value to the nearest integer. Alternatively, client implementations can choose to use binary search to find the closest integer solution, as `coreth` [does to calculate a node's desired target excess](https://github.com/ava-labs/coreth/blob/ebaa8e028a3a8747d11e6822088b4af7863451d8/plugin/evm/upgrade/acp176/acp176.go#L170).

When building a block, builders can calculate their next preferred value for $q$ based on the network's current value (`q_current`) according to:

  ```python
  # Calculates a node's new desired value for q for a given block
  def calc_next_q(q_current: int, q_desired: int, max_change: int) -> int:
    if q_desired > q_current:
        return q_current + min(q_desired - q_current, max_change)
    else:
        return q_current - min(q_current - q_desired, max_change)
  ```

As $q$ is updated after the execution of transactions within the block, $m$ is also updated such that $m = M \cdot e^{\frac{q}{D}}$ at all times. As noted above, the change to $m$ only takes effect for subsequent block production, and cannot change the time at which block $b$ can be produced itself.

### Activation Parameters for the C-Chain

Parameters at activation on the C-Chain are:

<div align="center">

| Parameter | Description | C-Chain Configuration|
| - | - | - |
| $M$ | minimum `minimumBlockDelay` value | 100 milliseconds |
| $q$ | initial target `minimumBlockDelay` excess | 3,141,253 |
| $D$ | `minimumBlockDelay` update constant | $2^{20}$ |
| $Q$ | `minimumBlockDelay` update factor change limit | 200 |

</div>

$M$ was chosen to balance providing the option of significantly faster block times without the need for future network upgrades while still providing an upper bound on the maximum possible rate of block production and changes to the `minimumBlockDelay` value.

Based on the 100-millisecond value for $M$, $q$ was chosen such that the effective `minimumBlockDelay` value at time of activation is as close as possible to the current target block rate of the C-Chain, which is 2 seconds.

$D$ and $Q$ were chosen such that it takes approximately 3,600 consecutive blocks of the maximum allowed change in $q$ for the effective `minimumBlockDelay` value to either halve or double.

### ProposerVM `MinBlkDelay`

The ProposerVM currently offers a static, configurable `MinBlkDelay` seconds for consecutive blocks. With this ACP enforcing a dynamic minimum block delay time, any EVM instance adopting this ACP that also leverages the ProposerVM should ensure that the ProposerVM `MinBlkDelay` is set to 0.

### Note on Block Building

While there is no longer a requirement for blocks to burn a minimum block gas cost after the activation of this ACP, block builders should still take priority fees into account when building blocks to allow for transaction prioritization and to maximize the amount of native token (AVAX) burned in the block. 

From a user (transaction issuer) perspective, this means that a non-zero priority fee would only ever need to be set to ensure inclusion during periods of maximum gas utilization.

## Backwards Compatibility

While this proposal requires a network upgrade and updates the EVM block header format, it does so in a way that tries to maintain as much backwards compatibility as possible. Specifically, applications that currently parse and use the existing timestamp field that is denominated in seconds can continue to do so. The `timestampMilliseconds` header value only needs to be used in cases where more granular timestamps are required.

## Reference Implementation

A reference implementation is not yet provided, and must be made available for this ACP to be considered `implementable`.

## Security Considerations

Too rapid block production may cause availability issues if validators of the given blockchain are not able to keep up with blocks being proposed to consensus. This new mechanism allows validators to help influence the maximum frequency at which blocks are allowed to be produced, but potential misconfiguration or overly aggressive settings may cause problems for some validators.

The mechanism for the minimum block delay time to adapt based on validator preference has already been used previously to allow for dynamic gas targets based on validator preference on the C-Chain, providing more confidence that it is suitable for controlling this network parameter as well. However, because each block is capable of changing the value of the minimum block delay by a certain amount, the lower the minimum block delay is, the more blocks that can be produced in a given time, and the faster the minimum block delay value will be able to change. This creates a dynamic where the mechanism for controlling `minimumBlockDelay` is more reactive at lower values, and less reactive at higher values. The global minimum `minimumBlockDelay` ($M$) provides a lower bound of how quickly blocks can ever be produced, but it is left to validators to ensure that the effective value does not exceed their collective preference.

## Acknowledgments

Thanks to [Luigi D'Onorio DeMeo](https://x.com/luigidemeo) for continually bringing up the idea of reducing block times to provide better UX for users of Avalanche blockchains.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
