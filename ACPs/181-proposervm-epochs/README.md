| ACP | 181 |
| :--- | :--- |
| **Title** | ProposerVM Epochs |
| **Author(s)** | Cam Schultz [@cam-schultz](https://github.com/cam-schultz) |
| **Status** | Proposed, Implementable, Activated, Stale ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

Proposes the introduction of epochs to the ProposerVM, such that a consistent view of the P-Chain may be provided to inner VMs for a known duration of time. This would enable VMs to optimize validator set retrievals, which currently must be done as often as every P-Chain block.

## Motivation

The P-Chain maintains a registry of L1 and Subnet validators (including Primary Network validators). Validators are added, removed, or their weights changed by issuing P-Chain transactions that are included in P-Chain blocks. When describing an L1 or Subnet's validator set, what is really being described are the weights, BLS keys, and Node IDs of the active validators at a particular P-Chain height. Use cases that require on-demand views of L1 or Subnet validator sets need to fetch validator sets at arbitrary P-Chain heights, while use cases that require up-to-date views need to fetch them as often as every P-Chain block.

ProposerVM epochs during which the P-Chain height is fixed would widen this window to a predictable epoch duration, allowing these use cases to implement optimizations such as pre-fetching validator sets once per epoch, or allowing more efficient backwards traversal of the P-Chain to fetch historical validator sets.

## Specification

### Epoch Definition

An epoch $E_n$ is defined by its start time $T_{start}^n$ and its end time $T_{end}^n$. A block $b_m$ with timestamp $t_m$ *seals* $E_n$ if both of the following are true:
- $t_{m-1} < T_{end}^n <= t_m$
- $b_{m-1} \in E_n$ 

In other words, $b$ seals $E_n$ if it's the first block to cross the epoch boundary, and its parent is in $E_n$.

A sealing block is a member of the epoch it seals: $b_m \in E_n$.

### Epoch Duration

Epochs have a constant duration $D$, such that for any given epoch $E_n$, the difference between its start and end times equals this duration: 

$$
T_{end}^n - T_{start}^n = D
$$ 

$D$ is hardcoded into the ProposerVM source code, and may only be changed by a required network upgrade.

#### Changing the Epoch Duration

Future network upgrades may change the value of $D$ to some new duration $D'$. $D'$ should not take effect until the end of the current epoch, rather than the activation time of the network upgrade that defines $D'$. This ensures an in progress epoch at the upgrade activation time cannot have a realized duration less than both $D$ and $D'$.

### Epoch Number

Each epoch is associated with a monotonically increasing number, with $E_0$ being the epoch beginning at the activation time of the network upgrade that activates this ACP, and subsequent epochs incrementing the epoch number. Note that validators do not need to agree on epoch numbers, only the transition points between epochs, so no numbering scheme is provided in this ACP.

### P-Chain Height

As discussed above, the main [motivation](#motivation) for introducing epochs to the ProposerVM is to provide a fixed view of the P-Chain for the duration of the epoch. To achieve this, ProposerVM blocks are extended to include the current epoch's fixed P-Chain height, `PChainEpochHeight`, in addition to the latest P-Chain height, `PChainHeight`. The validator set at `PChainEpochHeight` should be used by the ProposerVM and the inner VM for the duration of the epoch. The first block of a new epoch will set the epoch's `PChainEpochHeight` to its parent's `PChainHeight`.

This approach ensures that at any given height, the validator set to be used for the next block is known (this is a basic requirement for light clients). Put another way, within an epoch, the next block will use the current block's `PChainEpochHeight`, and at the boundary of the next epoch, the next block will use the current block's `PChainHeight`.

### Low Traffic Chain Edge Cases

The epoch sealing [definition](#epoch-definition) produces a couple of interesting edge cases worth considering. In each of the below diagrams, **bold** block numbers indicate sealing blocks, and the rectangles denote epoch membership.

1. What happens if there is only a single block with a timestamp in an epoch's range?

    Let $b_m$ be the only block with a timestamp in $E_{n+1}$'s range, meaning that $b_m$ seals $E_n$. $b_{m+1}$ must therefore seal $E_{n+1}$, meaning that it would be the only block in $E_{n+1}$, even though its timestamp does not fall with $E_{n+1}$'s range.

    <p align="center">
      <img src=./edge_case_1.png />
    </p>

2. What happens if there are no blocks with a timestamp in an epoch's range?

    The next block that is produced will seal the epoch that its parent belonged to, even if there are entire epoch(s) that have since elapsed. This can result in a scenario in which the the sealing block's `PChainEpochHeight` is behind its `PChainHeight` by an arbitrary amount.

    <p align="center">
      <img src=./edge_case_2.png />
    </p>

## Backwards Compatibility

This change requires a network upgrade and is therefore not backwards compatible.

## Reference Implementation

A draft reference implementation is available in [AvalancheGo](https://github.com/ava-labs/avalanchego/pull/3746), and must be merged before this ACP may be considered `Implementable`.

## Security Considerations

### Excessive Validator Churn

The introduction of epochs concentrates validator set changes over the epoch's duration into a single block at the epoch's boundary. Excessive validator churn can cause consensus failures and other dangerous behavior, so it is imperative that the amount of validator weight change at the epoch boundary is limited. One strategy to accomplish this is to queue validator set changes and spread them out over multiple epochs. Another strategy is to batch updates to the same validator together such that increases and decreases to that validator's weight cancel each other out. Mechanisms to mitigate against this are outside the scope of this ACP and left to validator managers to implement.

## Open Questions

- What should the epoch duration $D$ be set to?

- Should the epoch numbering scheme be defined in this ACP?

- Should validator churn limits be implemented for the primary network to mitigate against [excessive validator churn](#excessive-validator-churn)?

- Is it safe for `PChainEpochHeight` and `PChainHeight` to differ significantly within a block, as described [above](#low-traffic-chain-edge-cases)?

## Acknowledgements

Thanks to [@iansuvak](https://github.com/iansuvak),  [@geoff-vball](https://github.com/geoff-vball), [@yacovm](https://github.com/yacovm), [@michaelkaplan13](https://github.com/michaelkaplan13), and [@StephenButtolph](https://github.com/StephenButtolph) for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
