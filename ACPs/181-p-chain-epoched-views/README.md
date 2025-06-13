| ACP | 181 |
| :--- | :--- |
| **Title** | P-Chain Epoched Views |
| **Author(s)** | Cam Schultz [@cam-schultz](https://github.com/cam-schultz) |
| **Status** | Proposed, Implementable, Activated, Stale ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

Proposes a standard P-Chain epoching scheme such that any VM that implements it uses a P-Chain block height known prior to the generation of its next block. This would enable VMs to optimize validator set retrievals, which currently must be done during block execution. This standard does *not* introduce epochs to the P-Chain's VM directly. Instead, it provides a standard that may be implemented by layers that inject P-Chain state into VMs, such as the ProposerVM.

## Motivation

The P-Chain maintains a registry of L1 and Subnet validators (including Primary Network validators). Validators are added, removed, or their weights changed by issuing P-Chain transactions that are included in P-Chain blocks. When describing an L1 or Subnet's validator set, what is really being described are the weights, BLS keys, and Node IDs of the active validators at a particular P-Chain height. Use cases that require on-demand views of L1 or Subnet validator sets need to fetch validator sets at arbitrary P-Chain heights, while use cases that require up-to-date views need to fetch them as often as every P-Chain block.

Epochs during which the P-Chain height is fixed would widen this window to a predictable epoch duration, allowing these use cases to implement optimizations such as pre-fetching validator sets once per epoch, or allowing more efficient backwards traversal of the P-Chain to fetch historical validator sets.

## Specification

### Assumptions

In the following specification, we assume that a block $b_m$ has timestamp $t_m$ and P-Chain height $p_m$.

### Epoch Definition

An epoch is defined as a contiguous range of blocks that share the same three values:
- An Epoch Number
- An Epoch P-Chain Height
- An Epoch Start Time

Let $E_N$ denote an epoch with epoch number $N$. $E_N$'s start time is denoted as $T_{start}^N$, and its P-Chain height as $P_N$. 

$E_0$ is defined as the epoch whose start time $T_{start}^0$ is the block timestamp of the block that activates this ACP (i.e. the first block at or following this ACP's activation timestamp).

### Epoch Sealing

An epoch $E_N$ is *sealed* by the first block with a timestamp greater than or equal to $T_{start}^N + D$, where $D$ is a constant defined in the network upgrade that activates this ACP. Let $B_{S_N}$ denote the block that sealed $E_N$.

The sealing block is defined to be a member of the epoch it seals. This guarantees that every epoch will contain at least one block. 

### Advancing an Epoch

We advance from the current epoch $E_N$ to the next epoch $E_{N+1}$ when the next block after $B_{S_N}$ is produced. This block will be a member of $E_{N+1}$, and will have the values:
- $P_{N+1}$ equal to the P-Chain height of $B_{S_N}$
- $T_{start}^{N+1}$ equal to $B_{S_N}$'s timestamp
- The epoch number, $N+1$ increments the previous epoch's epoch number by exactly $1$

## Properties and Use Cases

### Epoch Duration Bounds

Since an epoch's start time is set to the [timestamp of the sealing block of the previous epoch](#advancing-an-epoch), all epochs are guaranteed to have a duration of at least $D$, as measured from the epoch's starting time to the timestamp of the epoch's sealing block. However, since a sealing block is [defined](#epoch-sealing) to be a member of the epoch it seals, there is no upper bound on an epoch's duration, since that sealing block may be produced at any point in the future beyond $T_{start}^N + D$.

### Fixing the P-Chain Height

When building a block, Avalanche blockchains use the P-Chain height [embedded in the block](#assumptions) to determine the validator set. If instead the epoch P-Chain height is used, then we can ensure that when a block is built, the validator set to be used for the next block is known. To see this, suppose block $b_m$ seals epoch $E_N$. Then the next block, $b_{m+1}$ will begin a new epoch, $E_{N+1}$ with $P_{N+1}$ equal to $b_m$'s P-Chain height, $p_m$. If instead $b_m$ does not seal $E_N$, then $b_{m+1}$ will continue to use $P_{N}$. Both candidates for $b_{m+1}$'s P-Chain height ($p_m$ and $P_N$) are known at $b_m$ build time.

## Backwards Compatibility

This change requires a network upgrade and is therefore not backwards compatible.

Any downstream entities that depend on a VM's view of the P-Chain will also need to account for epoched P-Chain views. For instance, ICM messages are signed by an L1's validator set at a specific P-Chain height. Currently, the constructor of the signed message can in practice use the validator set at the P-Chain tip, since all deployed Avalanche VMs are at most behind the P-Chain by a fixed number of blocks. With epoching, however, the ICM message constructor must take into account the epoch P-Chain height of the verifying chain, which may be arbitrarily far behind the P-Chain tip.

## Reference Implementation

The following pseudocode illustrates how an epoch may be calculated for a block:

```go
// Epoch Duration
const D time.Duration

type Epoch struct {
    PChainHeight uint64
    Number uint64
    StartTime time.Time
}

type Block interface {
    Timestamp() time.Time
    PChainHeight() uint64
    Epoch() Epoch
}

func GetPChainEpoch(parent Block) Epoch {
    if parent.Timestamp().After(time.Add(parent.Epoch().StartTime, D)) {
        // If the parent crossed its epoch boundary, then it sealed its epoch.
        // The child is the first block of the new epoch, so it should use the parent's
        // P-Chain height as the new epoch's height, and its parent's timestamp as the new
        // epoch's starting time
        return Epoch{
            PChainHeight: parent.PChainHeight()
            Number: parent.Epoch().Number + 1
            StartTime: parent.Timestamp()
        }
    }

    // Otherwise, the parent did not seal its epoch, so the child should use the same
    // epoch height. This is true even if the child seals its epoch, since sealing
    // blocks are considered to be part of the epoch they seal.
    return Epoch{
        PChainHeight: parent.Epoch().PChainHeight
        Number: parent.Epoch().Number
        StartTime: parent.Epoch().StartTime
    }
}
```

- If the parent sealed its epoch, the current block [advances the epoch](#advancing-an-epoch), refreshing the epoch height, incrementing the epoch number, and setting the epoch starting time.
- Otherwise, the current block uses the current epoch height, number, and starting time, regardless of whether it seals the epoch.

A full reference implementation that implements this ACP in the ProposerVM is available in [AvalancheGo](https://github.com/ava-labs/avalanchego/pull/3746), and must be merged before this ACP may be considered `Implementable`.

### Setting the Epoch Duration

The epoch duration $D$ is hardcoded in the upgrade configuration, and may only be changed by a required network upgrade.

#### Changing the Epoch Duration

Future network upgrades may change the value of $D$ to some new duration $D'$. $D'$ should not take effect until the end of the current epoch, rather than the activation time of the network upgrade that defines $D'$. This ensures an in progress epoch at the upgrade activation time cannot have a realized duration less than both $D$ and $D'$.

## Security Considerations

### Excessive Validator Churn

The introduction of epochs concentrates validator set changes over the epoch's duration into a single block at the epoch's boundary. Excessive validator churn can cause consensus failures and other dangerous behavior, so it is imperative that the amount of validator weight change at the epoch boundary is limited. One strategy to accomplish this is to queue validator set changes and spread them out over multiple epochs. Another strategy is to batch updates to the same validator together such that increases and decreases to that validator's weight cancel each other out. Mechanisms to mitigate against this are outside the scope of this ACP and left to validator managers to implement.

## Open Questions

- What should the epoch duration $D$ be set to?

- Should validator churn limits be implemented for the primary network to mitigate against [excessive validator churn](#excessive-validator-churn)?

- Is it safe for `PChainEpochHeight` and `PChainHeight` to differ significantly within a block, due to [unbounded epoch duration](#epoch-duration-bounds)?

- Is there a better mechanism than a hardcoded network upgrade parameter that can be used to set $D$? The current approach imposes uniform $D$ across all VM instances, even though that is not a functional requirement of this proposal.

## Acknowledgements

Thanks to [@iansuvak](https://github.com/iansuvak),  [@geoff-vball](https://github.com/geoff-vball), [@yacovm](https://github.com/yacovm), [@michaelkaplan13](https://github.com/michaelkaplan13), [@StephenButtolph](https://github.com/StephenButtolph), and [@aaronbuchwald](https://github.com/aaronbuchwald) for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
