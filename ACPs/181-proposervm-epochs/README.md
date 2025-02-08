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

An epoch $E_n$ is defined by its start time $T_{start}^n$ and its end time $T_{end}^n$. Any block with timestamp $t$ is included in epoch $E_n$ if $T_{start}^n <= t < T_{end}^n$. The first block whose timestamp is greater than or equal to $T_{end}^n$ and whose *parent's* timestamp is less than $t_{start}^n$ will be the final block in $E_n$. Formally, for a block with timestamp $t$ and parent timestamp $t_{parent}, the block *seals* $E_n$ if 

$$
t >= T_{end}^n \: \text{ and } \: t_{parent} < T_{start}^n
$$

The next block after the block that seals $E_n$ is the first block of epoch $E_{n+1}$.

### Epoch Duration

Epochs have a constant duration $D$, such that for any given epoch $E_n$, the difference between its start and end times equals this duration: 

$$
T_{end}^n - T_{start}^n = D
$$ 

$D$ is hardcoded into the ProposerVM source code, and may only be changed by a required network upgrade.

#### Changing the Epoch Duration

Future network upgrades may change the value of $D$ to some new duration $D'$. $D'$ should not take effect until the end of the current epoch, rather than the activation time of the network upgrade that defines $D'$. This ensures an in progress epoch at the upgrade activation time cannot be less than both $D$ and $D'$.

### Epoch Number

Each epoch is associated with a monotonically increasing number, with $E_0$ being the epoch beginning at the activation time of the network upgrade that activates this ACP, and subsequent epochs incrementing the epoch number. Note that validators do not need to agree on epoch numbers, only the transition points between epochs, so no numbering scheme is provided in this ACP.

### P-Chain Height

As discussed above, the main [motivation](#motivation) for introducing epochs to the ProposerVM is to provide a fixed view of the P-Chain for the duration of the epoch. To achieve this, ProposerVM blocks are extended to include the current epoch's fixed P-Chain height, `PChainEpochHeight`, in addition to the latest P-Chain height, `PChainHeight`. The validator set at `PChainEpochHeight` should be used by the ProposerVM and the inner VM for the duration of the epoch. The first block of a new epoch will set the epoch's `PChainEpochHeight` to its parent's `PChainHeight`.

This approach ensures that at any given height, the validator set to be used for the next block is known (this is a basic requirement for light clients). Put another way, within an epoch, the next block will use the current block's `PChainEpochHeight`, and at the boundary of the next epoch, the next block will use the current block's `PChainHeight`.

## Backwards Compatibility

This change requires a network upgrade and is therefore not backwards compatible.

## Reference Implementation

A reference implementation will be provided in [AvalancheGo](https://github.com/ava-labs/avalanchego), which must be merged before this ACP may be considered `Implementable`.

## Security Considerations

### Excessive Validator Churn

The introduction of epochs concentrates validator set changes over the epoch's duration into a single block at the epoch's boundary. Excessive validator churn can cause consensus failures and other dangerous behavior, so it is imperative that the amount of validator weight change at the epoch boundary is limited. One strategy to accomplish this is to queue validator set changes and spread them out over multiple epochs. Another strategy is to batch updates to the same validator together such that increases and decreases to that validator's weight cancel each other out. Mechanisms to mitigate against this are outside the scope of this ACP and left to validator managers to implement. 

## Open Questions

What should the epoch duration $D$ be set to?

Should the epoch numbering scheme be defined in this ACP?

Should validator churn limits be implemented for the primary network to mitigate against [excessive validator churn](#excessive-validator-churn)?

## Acknowledgements

Thanks to [@iansuvak](https://github.com/iansuvak),  [@geoff-vball](https://github.com/geoff-vball), [@yacovm](https://github.com/yacovm), [@michaelkaplan13](https://github.com/michaelkaplan13), and [@StephenButtolph](https://github.com/StephenButtolph) for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
