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

Let $T_{start}^0$ be the activation time of the network upgrade that activates this ACP. The time axis is divided into _epochs_ of duration $D$ such that an epoch $E_n$ is defined by its start and end times:

$$
E_n \coloneqq [ T_{start}^n,T_{end}^n )
$$

and the difference between $T_{end}^n$ and $T_{start}^n$ is $D$ for any $n$:

$$
T_{end}^n - T_{start}^n = D
$$ 

### Mapping Blocks to Epochs

#### Sealing

An epoch $E_n$ with end time $T_{end}^n$ is *sealed* by the first block that reaches the epoch boundary. For example, a block $b_m$ with timestamp $t_m$ seals $E_n$ if it is the first block for which the following is true:

$$
t_{m-1} < T_{end}^n <= t_m
$$

The sealing block is defined to be a member of the epoch it seals. In this example, $b_m \in E_n$.

#### Epoch Number

Blocks are mapped to epochs using an `EpochNumber`. The `EpochNumber`is constant for all blocks in an epoch, and is incremented by one in the next block after the epoch's sealing block. The first block produced after $T_{start}^0$ will be assigned `EpochNumber = 0`. The block that seals an epoch is given the same `EpochNumber` as the blocks in that epoch.

Note that due the edge case discussed [below](#what-happens-if-there-are-no-blocks-with-a-timestamp-in-an-epochs-range), a block's `EpochNumber` is not expected to be equal to the number of epoch durations that have elapsed since $T_{start}^0$.

#### P-Chain Height

As discussed above, the main [motivation](#motivation) for introducing epochs to the ProposerVM is to provide a fixed view of the P-Chain for the duration of the epoch. To achieve this, ProposerVM blocks are extended to include the current epoch's fixed P-Chain height, `PChainEpochHeight`, in addition to the latest P-Chain height, `PChainHeight`. The validator set at `PChainEpochHeight` should be used by the ProposerVM and the inner VM for the duration of the epoch. The first block of a new epoch will set the epoch's `PChainEpochHeight` to its parent's `PChainHeight`.

This approach ensures that at any given height, the validator set to be used for the next block is known (this is a basic requirement for light clients). Put another way, within an epoch, the next block will use the current block's `PChainEpochHeight`, and at the boundary of the next epoch, the next block will use the current block's `PChainHeight`.

#### Low Traffic Chain Edge Cases

The epoch sealing [definition](#epoch-definition) produces a couple of interesting edge cases worth considering. In each of the below diagrams, **bold** block numbers indicate sealing blocks, and the rectangles denote epoch membership.

##### What happens if there is only a single block with a timestamp in an epoch's range?

Let $b_m$ be the only block with a timestamp in $E_{n+1}$'s range, meaning that $b_m$ seals $E_n$. $b_{m+1}$ must therefore seal $E_{n+1}$, meaning that it would be the only block in $E_{n+1}$, even though its timestamp does not fall with $E_{n+1}$'s range.

<p align="center">
  <img src=./edge_case_1.png />
</p>

##### What happens if there are no blocks with a timestamp in an epoch's range?

The next block that is produced will seal the epoch that its parent belonged to, even if there are entire epoch(s) that have since elapsed. This can result in a scenario in which the the sealing block's `PChainEpochHeight` is behind its `PChainHeight` by an arbitrary amount.

<p align="center">
  <img src=./edge_case_2.png />
</p>

## Backwards Compatibility

This change requires a network upgrade and is therefore not backwards compatible.

## Reference Implementation

The following pseudocode illustrates how the specified epoch definition may be used to select a block's `PChainEpochHeight` and `EpochNumber`:

```go
type Block interface {
    Timestamp() time.Time
    PChainHeight() uint64
    PChainEpochHeight() uint64
    EpochNumber() uint64
}

// Returns true if timestamp1 and timestamp2 are in different epochs
func CrossedEpochBoundary(timestamp1, timestamp2 time.Time) bool

// [grandParent] is [parent]'s parent
func GetPChainEpoch(parent, grandParent Block) (height, number uint64) {
    if CrossedEpochBoundary(parent.Timestamp(), grandParent.Timestamp()) {
		// If the parent crossed the epoch boundary, then it sealed the previous epoch. The child
		// is the first block of the new epoch, so should use the parent's P-Chain height.
		return parent.PChainHeight(), parent.EpochNumber() + 1
	}
	// Otherwise, the parent did not seal the previous epoch, so the child should use the same
	// epoch height. This is true even if the child crosses the epoch boundary, since sealing
	// blocks are considered to be part of the epoch they seal.
	return parent.PChainEpochHeight(), parent.EpochNumber()
}
```

- `func CrossedEpochBoundary(timestamp1, timestamp2 time.Time) bool` divides the time axis into intervals of length $D$, starting from $T_{start}^0$ and returns `true` if the two timestamps are in different intervals. Its usage here checks if the block's parent [sealed](#epoch-definition) its epoch.

- If the parent sealed its epoch, the current block advances the epoch, [refreshing the epoch height](#p-chain-height), and [incrementing the epoch number](#epoch-number).
- Otherwise, the current block uses the current epoch height and number, regardless of whether it seals the epoch.

A full reference implementation is available in [AvalancheGo](https://github.com/ava-labs/avalanchego/pull/3746), and must be merged before this ACP may be considered `Implementable`.

### Setting the Epoch Duration

The epoch duration $D$ is hardcoded into the ProposerVM source code, and may only be changed by a required network upgrade.

#### Changing the Epoch Duration

Future network upgrades may change the value of $D$ to some new duration $D'$. $D'$ should not take effect until the end of the current epoch, rather than the activation time of the network upgrade that defines $D'$. This ensures an in progress epoch at the upgrade activation time cannot have a realized duration less than both $D$ and $D'$.

## Security Considerations

### Excessive Validator Churn

The introduction of epochs concentrates validator set changes over the epoch's duration into a single block at the epoch's boundary. Excessive validator churn can cause consensus failures and other dangerous behavior, so it is imperative that the amount of validator weight change at the epoch boundary is limited. One strategy to accomplish this is to queue validator set changes and spread them out over multiple epochs. Another strategy is to batch updates to the same validator together such that increases and decreases to that validator's weight cancel each other out. Mechanisms to mitigate against this are outside the scope of this ACP and left to validator managers to implement.

## Open Questions

- What should the epoch duration $D$ be set to?

- Should validator churn limits be implemented for the primary network to mitigate against [excessive validator churn](#excessive-validator-churn)?

- Is it safe for `PChainEpochHeight` and `PChainHeight` to differ significantly within a block, as described [above](#low-traffic-chain-edge-cases)?

- Is there a better mechanism than a hardcoded network upgrade parameter that can be used to set $D$? The current approach imposes uniform $D$ across all ProposerVM instances, even though that is not a functional requirement of this proposal.

## Acknowledgements

Thanks to [@iansuvak](https://github.com/iansuvak),  [@geoff-vball](https://github.com/geoff-vball), [@yacovm](https://github.com/yacovm), [@michaelkaplan13](https://github.com/michaelkaplan13), [@StephenButtolph](https://github.com/StephenButtolph), and [@aaronbuchwald](https://github.com/aaronbuchwald) for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
