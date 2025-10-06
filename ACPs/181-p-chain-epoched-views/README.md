| ACP | 181 |
| :--- | :--- |
| **Title** | P-Chain Epoched Views |
| **Author(s)** | Cam Schultz [@cam-schultz](https://github.com/cam-schultz) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/211)) |
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

Let block $b_a$ be the block that activates this ACP. The first epoch ($E_0$) has $T_{start}^0 = t_{a-1}$, and $P_0 = p_{a-1}$. In other words, the first epoch start time is the timestamp of the last block prior to the activation of this ACP, and similarly, the first epoch P-Chain height is the P-Chain height of last block prior to the activation of this ACP.

### Epoch Sealing

An epoch $E_N$ is *sealed* by the first block with a timestamp greater than or equal to $T_{start}^N + D$, where $D$ is a constant defined in the network upgrade that activates this ACP. Let $B_{S_N}$ denote the block that sealed $E_N$.

The sealing block is defined to be a member of the epoch it seals. This guarantees that every epoch will contain at least one block. 

### Advancing an Epoch

We advance from the current epoch $E_N$ to the next epoch $E_{N+1}$ when the next block after $B_{S_N}$ is produced. This block will be a member of $E_{N+1}$, and will have the values:
- $P_{N+1}$ equal to the P-Chain height of $B_{S_N}$
- $T_{start}^{N+1}$ equal to $B_{S_N}$'s timestamp
- The epoch number, $N+1$ increments the previous epoch's epoch number by exactly $1$

## Properties

### Epoch Duration Bounds

Since an epoch's start time is set to the [timestamp of the sealing block of the previous epoch](#advancing-an-epoch), all epochs are guaranteed to have a duration of at least $D$, as measured from the epoch's starting time to the timestamp of the epoch's sealing block. However, since a sealing block is [defined](#epoch-sealing) to be a member of the epoch it seals, there is no upper bound on an epoch's duration, since that sealing block may be produced at any point in the future beyond $T_{start}^N + D$.

### Fixing the P-Chain Height

When building a block, Avalanche blockchains use the P-Chain height [embedded in the block](#assumptions) to determine the validator set. If instead the epoch P-Chain height is used, then we can ensure that when a block is built, the validator set to be used for the next block is known. To see this, suppose block $b_m$ seals epoch $E_N$. Then the next block, $b_{m+1}$ will begin a new epoch, $E_{N+1}$ with $P_{N+1}$ equal to $b_m$'s P-Chain height, $p_m$. If instead $b_m$ does not seal $E_N$, then $b_{m+1}$ will continue to use $P_{N}$. Both candidates for $b_{m+1}$'s P-Chain height ($p_m$ and $P_N$) are known at $b_m$ build time.

## Use Cases

### ICM Verification Optimization

For a validator to verify an ICM message, the signing L1/Subnet's validator set must be retrieved during block execution by traversing backward from the current P-Chain height to the P-Chain height provided by the ProposerVM. The traversal depth is highly variable, so to account for the worst case, VM implementations charge a large fixed amount of gas to perform this verification.

With epochs, validator set retrieval occurs at fixed P-Chain heights that increment at regular intervals, which provides opportunities to optimize this retrieval. For instance, validator retrieval may be done asynchronously from block execution as soon as $D$ time has passed since the current epoch's start time, allowing the verification gas cost to be safely reduced by a significant amount.

### Improved Relayer Reliability

Current ICM VM implementations verify ICM messages against the local P-Chain state, as determined by the P-Chain height set by the ProposerVM. Off-chain relayers perform the following steps to deliver ICM messages:
1. Fetch the sending chain's validator set at the verifying chain's current proposed height
1. Collect BLS signatures from that validator set to construct the signed ICM message
1. Submit the transaction containing the signed message to the verifying chain

If the validator set changes between steps 1 and 3, the ICM message will fail verification.

Epochs improve upon this by fixing the P-Chain height used to verify ICM messages for a duration of time that is predictable to off-chain relayers. A relayer should be able to derive the epoch boundaries based on the specification above, or they could retrieve that information via a node API. Relayers could use that information to decide the validator set to query, knowing that it will be stable for the duration of the epoch. Further, VMs could relax the verification rules to allow ICM messages to be verified against the previous epoch as a fallback, eliminating edge cases around the epoch boundary.

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
	parentTimestamp := parent.Timestamp()
	parentEpoch := parent.Epoch()
	epochEndTime := parentEpoch.StartTime.Add(D)
	if parentTimestamp.Before(epochEndTime) {
		// If the parent was issued before the end of its epoch, then it did not
		// seal the epoch.
		return parentEpoch
	}

	// The parent sealed the epoch, so the child is the first block of the new
	// epoch.
	return Epoch{
		PChainHeight: parent.PChainHeight(),
		Number:       parentEpoch.Number + 1,
		StartTime:    parentTimestamp,
	}
}
```

- If the parent sealed its epoch, the current block [advances the epoch](#advancing-an-epoch), refreshing the epoch height, incrementing the epoch number, and setting the epoch starting time.
- Otherwise, the current block uses the current epoch height, number, and starting time, regardless of whether it seals the epoch.

A full reference implementation that implements this ACP in the ProposerVM is available in [AvalancheGo](https://github.com/ava-labs/avalanchego/pull/3746), and must be merged before this ACP may be considered `Implementable`.

### Setting the Epoch Duration

The epoch duration $D$ is set on a network-wide level. For both Fuji (network ID 5) and Mainnet (network ID 1), $D$ will be set to 5 minutes upon activation of this ACP. Any changes to $D$ in the future would require another network upgrade.

#### Changing the Epoch Duration

Future network upgrades may change the value of $D$ to some new duration $D'$. $D'$ should not take effect until the end of the current epoch, rather than the activation time of the network upgrade that defines $D'$. This ensures an in progress epoch at the upgrade activation time cannot have a realized duration less than both $D$ and $D'$.

## Security Considerations

### Epoch P-Chain Height Skew

Because epochs may have [unbounded duration](#epoch-duration-bounds), it is possible for a block's `PChainEpochHeight` to be arbitrarily far behind the tip of the P-Chain. This does not affect the *validity* of ICM verification within a VM that implements P-Chain epoched views, since the validator set at `PChainEpochHeight` is always known. However, the following considerations should be made under this scenario:
1. As validators exit the validator set, their physical nodes may be unavailable to serve BLS signature requests, making it more difficult to construct a valid ICM message
1. A valid ICM message may represent an attestation by a stale validator set. Signatures from validators that have exited the validator set between `PChainEpochHeight` and the current P-Chain tip will not represent active stake.

Both of these scenarios may be mitigated by having shorter epoch lengths, which limit the delay in time between when the P-Chain is updated and when those updates are taken into account for ICM verification on a given L1, and by ensuring consistent block production, so that epochs always advance soon after $D$ time has passed.

### Excessive Validator Churn

If an epoched view of the P-Chain is used by the consensus engine, then validator set changes over an epoch's duration will be concentrated into a single block at the epoch's boundary. Excessive validator churn can cause consensus failures and other dangerous behavior, so it is imperative that the amount of validator weight change at the epoch boundary is limited. One strategy to accomplish this is to queue validator set changes and spread them out over multiple epochs. Another strategy is to batch updates to the same validator together such that increases and decreases to that validator's weight cancel each other out. Given the primary use case of ICM verification improvements, which occur at the VM level, mechanisms to mitigate against this are omitted from this ACP.

## Open Questions

- What should the epoch duration $D$ be set to?

- Is it safe for `PChainEpochHeight` and `PChainHeight` to differ significantly within a block, due to [unbounded epoch duration](#epoch-duration-bounds)?

## Acknowledgements

Thanks to [@iansuvak](https://github.com/iansuvak),  [@geoff-vball](https://github.com/geoff-vball), [@yacovm](https://github.com/yacovm), [@michaelkaplan13](https://github.com/michaelkaplan13), [@StephenButtolph](https://github.com/StephenButtolph), and [@aaronbuchwald](https://github.com/aaronbuchwald) for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
