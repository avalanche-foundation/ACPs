```text
ACP: <TODO>
Title: Subnet-Only Validators (SOVs)
Author(s): Patrick O'Grady <patrickogrady.xyz>
Discussions-To: <TODO>
Status: Proposed
Track: Standards
```

## Abstract

Introduce Subnet-Only Validators (SOVs), a new type of staker that can validate any Avalanche Subnet but is not required to validate the Primary Network. SOVs retain the ability to participate in Avalanche Warp Messaging (AWM) but do not need to sync the X/C-Chains must post an $AVAX-denominated bond...

Introduce a new type of Validator that is not required to validate the Primary Network...

Remove the requirement that Subnet Validators must validate the Primary Network without revoking support for Subnet Validators to send/verify Avalanche Warp Messages (AWM). Allow Subnet Validators to stake on a Subnet by posting an $AVAX-deonimated bond.

and instead require bond...

Validators without restricting their ability to send or verify Avalanche Warp Messages (AWM). Preview a future transition to Pay-As-You-Go Subnet Validation and $AVAX-Augmented Subnet Security.

## Motivation

Each node operator must stake at least 2000 $AVAX (~$20k at the time of writing) to first become a Primary Network validator before they qualify to become a Subnet Validator. All Subnet Validators, to satisfy their role as Primary Network Validators, must also [allocate 8 AWS vCPU, 16 GB RAM, and 1 TB storage](https://github.com/ava-labs/avalanchego/blob/master/README.md#installation) to sync the entire Primary Network (X-Chain, P-Chain, and C-Chain) and participate in its consensus, in addition to whatever resources are required for each Subnet they are validating.

TODO: work in 2000 $AVAX here? 
Although the fee paid to the Primary Network to operate a Subnet does not go up with the amount of activity on the Subnet, the fixed, upfront cost of setting up a Subnet Validator on the Primary Network deters new projects that prefer smaller, even variable, costs until demand is observed. _Unlike L2s that pay some increasing fee (usually denominated in units per transaction byte) to an external chain for data availability and security as activity scales, Subnets provide their own security/data availability and the only cost operators must pay from processing more activity is the hardware cost of supporting additional load._

Regulated entities that are prohibited from validating permissionless, smart contract-enabled blockchains (like the C-Chain) can’t launch a Subnet because they can’t opt-out of Primary Network Validation. This deployment blocker prevents a large cohort of Real World Asset (RWA) issuers from bringing unique, valuable tokens to the Avalanche Ecosystem (that could move between C-Chain <> Subnets using AWM/Teleporter).

TODO: fault isolation
A popular Subnet ("popular" meaning validated by many nodes) could destabilitze the Primary Network if usage spikes unexpectedly (cause an OOM, disk IO, CPU exhaustion on a large number of Primary Network validators) or the inverse on the Primary Network (where some undefined behavior could bring a Subnet offline). Allowing optional separation IMO is a step in the right direction to making Primary Network/Subnets more resilient to regressions in the other.

TODO: don't give up ability to use AWM
Don't give up ability


## Specification

### Overview

* Remove the requirement for a Subnet Validator to also be a Primary Network Validator but do not prevent it (current behavior not deprecated)
* Introduce a new transaction type on the P-Chain for Subnet Validators that only want to validate a Subnet (`AddSubnetOnlyValidatorTx`)
* Require Subnet-Only Validators to bond X $AVAX per validation per Subnet (to account for P-Chain Load)
* Track IPs for Subnet-Only Validators in AvalancheGo to allow ....
* Provide a guaranteed RL allotment for subnet validators
  * https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/config/keys.go#L181-L188
  * https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/config/keys.go#L198-L203

Without the requirement to validate the Primary Network, the need for Subnet Validators to instantiate and sync the C-Chain and X-Chain can be relaxed. Subnet Validators will only be required to sync the P-chain to track any validator set changes in their Subnet and to support Cross-Subnet communication via AWM (see “Primary Network Partial Sync” mode introduced in [Cortina 8](https://github.com/ava-labs/avalanchego/releases/tag/v1.10.8)). The lower resource requirement in this "minimal mode" will provide Subnets with greater flexibility of validation hardware requirements as operators are not required to reserve any resources for C-Chain/X-Chain operation.

_The value of the required "bond" (X) is open for debate. To avoid impacting network stability, I think it should be at least 250-750 \$AVAX. To set this "bond" lower, I think the PlatformVM should be futher optimized (assumes that lower fees lead to a corresponding increase in Subnets)._

### TODO

#### `AddSubnetOnlyValidatorTx`

_This is the same as [`AddPermissionlessValidatorTx`](https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/vms/platformvm/txs/add_permissionless_validator_tx.go#L33-L58). The exception being that the minimum staked tokens are ...._

```text
type AddSubnetOnlyValidatorTx struct {
	// Metadata, inputs and outputs
	BaseTx `serialize:"true"`
	// Describes the validator
	// The NodeID included in [Validator] must be the Ed25519 public key.
	Validator `serialize:"true" json:"validator"`
	// ID of the subnet this validator is validating
	Subnet ids.ID `serialize:"true" json:"subnetID"`
	// [Signer] is the BLS key for this validator.
	// Note: We do not enforce that the BLS key is unique across all validators.
	//       This means that validators can share a key if they so choose.
	//       However, a NodeID does uniquely map to a BLS key
	Signer signer.Signer `serialize:"true" json:"signer"`
	// Where to send staked tokens when done validating
	StakeOuts []*avax.TransferableOutput `serialize:"true" json:"stake"`
	// Where to send validation rewards when done validating
	ValidatorRewardsOwner fx.Owner `serialize:"true" json:"validationRewardsOwner"`
	// Where to send delegation rewards when done validating
	DelegatorRewardsOwner fx.Owner `serialize:"true" json:"delegationRewardsOwner"`
	// Fee this validator charges delegators as a percentage, times 10,000
	// For example, if this validator has DelegationShares=300,000 then they
	// take 30% of rewards from delegators
	DelegationShares uint32 `serialize:"true" json:"shares"`
}
```

### P2P
TODO

#### Version

https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/proto/p2p/p2p.proto#L93C1-L117C2

Trscked Subnets?

```text
// Version is the first outbound message sent to a peer when a connection is
// established to start the p2p handshake.
//
// Peers must respond to a Version message with a PeerList message to allow the
// peer to connect to other peers in the network.
//
// Peers should drop connections to peers with incompatible versions.
message Version {
  // Network the peer is running on (e.g local, testnet, mainnet)
  uint32 network_id = 1;
  // Unix timestamp when this Version message was created
  uint64 my_time = 2;
  // IP address of the peer
  bytes ip_addr = 3;
  // IP port of the peer
  uint32 ip_port = 4;
  // Avalanche client version
  string my_version = 5;
  // Timestamp of the IP
  uint64 my_version_time = 6;
  // Signature of the peer IP port pair at a provided timestamp
  bytes sig = 7;
  // Subnets the peer is tracking
  repeated bytes tracked_subnets = 8;
}
```

#### GetPeers
https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/proto/p2p/p2p.proto#L135-L165

```text
// Return SubnetOnlyValidators?
message GetPeers {
    bytes subnet_id = 1;
}
```

#### -> PeerList
https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/proto/p2p/p2p.proto#L135-L148

```text
// PeerList contains network-level metadata for a set of validators.
//
// PeerList must be sent in response to an inbound Version message from a
// remote peer a peer wants to connect to. Once a PeerList is received after
// a version message, the p2p handshake is complete and the connection is
// established.

// Peers should periodically send PeerList messages to allow peers to
// discover each other.
//
// PeerListAck should be sent in response to a PeerList.
message PeerList {
  repeated ClaimedIpPort claimed_ip_ports = 1;
}
```

#### -> PeerListAck
https://github.com/ava-labs/avalanchego/blob/638000c42e5361e656ffbc27024026f6d8f67810/proto/p2p/p2p.proto#L160-L165

```text
// PeerListAck is sent in response to PeerList to acknowledge the subset of
// peers that the peer will attempt to connect to.
message PeerListAck {
  reserved 1; // deprecated; used to be tx_ids
  repeated PeerAck peer_acks = 2;
}
```

### Future Work

The following ideas require their own ACPs...

Shared to (TODO: if not blocked by this, should share?)

...once relationship is decoupled, can consider 

#### Pay-As-You-Go Subnet Validation Fees
_To lower validation fees, should rely on BLS Multi-Singature Voting: https://hackmd.io/@patrickogrady/100k-subnets#How-will-BLS-Multi-Signature-uptime-voting-work_

Transition Subnet Validation fees to a dynamically priced, continuously charged mechanism that doesn't require locking/staking large amounts of $AVAX upfront _(this could be broken out as a separate step but makes sense to include in the voting code refactor, so kept here)_

It would be possible to just transition to a lower required "bond" amount, but many (myself included) think that it would be more competitve to transition to a dynamically priced, continuous payment mechanism. This new mechanism would target some $Y nAVAX fee that would be paid by each Subnet Validator per Subnet per second (pulling from a "Subnet Validator's Account") instead of requiring a large upfront lockup of \$AVAX.

_The rate of nAVAX/second should be set by the demand for validating Subnets on Avalanche compared to some usage target per Subnet and across all Subnets. This rate should be locked for each Subnet Validation period to ensure operators are not subject to suprise costs if demand rose significantly over time. All fees would still be paid to the network and burned, like all other P-Chain, X-Chain, and C-Chain transactions. The optimization work outlined here should allow the min rate to be set as low as ~512-4096 nAVAX/second (or 1.3-10.6 \$AVAX/month)._

#### $AVAX-Augmented Subnet Security
Some believe the relaxing of the validator relationship would be bad for $AVAX...


* Enable unstaked \$AVAX to be locked to a Subnet Validator on an Elastic Subnet and slashed if said Subnet Validator commits an attributable fault (i.e. proposes/signs conflicting blocks/AWM payloads)
* Reward locked \$AVAX associated with Subnet Validators that were not slashed with Elastic Subnet staking rewards

Currently, the only way to secure an Elastic Subnet is to stake its staking token. Many have requested the option to use \$AVAX for this token, however, this could easily allow an adversary to takeover small Elastic Subnets (where the amount of \$AVAX staked may be much less than the circulating supply).

\$AVAX-Augmented Subnet Security would allow anyone holding \$AVAX to lock it to specific Subnet Validators and earn Elastic Subnet reward tokens for supporting honest participants. Recall, all stake management on the Avalanche Network (even for Subnets) occurs on the P-Chain. Thus, staked tokens (\$AVAX and/or custom staking tokens used in Elastic Subnets) and stake weights (used for AWM verification) are secured by the full \$AVAX stake of the Primary Network. \$AVAX-Augmented Subnet Security, like staking, would be implemented on the P-Chain and enjoy the full security of the Primary Network. This approach means locking \$AVAX occurs on the Primary Network (no need to transfer \$AVAX to a Subnet, which may not be secured by meaningful value yet) and proofs of malicious behavior are processed on the Primary Network (a colluding Subnet could otherwise chose not to process a proof that would lead to their "lockers" being slashed).

_This approach is comparable to the idea of using \$ETH to secure DA on [EigenLayer](https://www.eigenlayer.xyz/) (without reusing stake) or \$BTC to secure Cosmos Zones on [Babylon](https://babylonchain.io/) (but not using an external ecosystem)._


## Backwards Compatibility

* All existing Subnet Validators can continue validating both the Primary Network and whatever Subnets they are validating. This change would just provide a new option for Subnet Validators that allows them to sacrifice their staking rewards for a smaller upfront $AVAX commitment and lower infrastructure costs.
* TODO: Support for this ACP would require adding a new transaction type to the P-Chain. This new transaction would not be compatible with Avalanche Cortina and require a mandatory Avalanche Network upgrade.

## Reference Implementation

`TODO` (once considered "Implementable")

## Security Considerations

* Any Subnet Validator running in "Partial Sync Mode" will not be able to verify Atomic Imports on the P-Chain and will rely entirely on Primary Network consensus to only accept valid P-Chain blocks.
* High-throughput Subnets will be better isolated from the Primary Network and should improve its resilience (i.e. surges of traffic on some Subnet cannot destabilize a Primary Network Validator).
* AvalancheGo must support tracking IPs for non-validators

## Open Questions

* Bond amount?

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Appendix

This ACP was previously posted as part of another discussion here: https://github.com/avalanche-foundation/ACPs/discussions/10#discussioncomment-7373486

All feedback is here: https://hackmd.io/@patrickogrady/100k-subnets#Feedback-to-Draft-Proposal

## Acknowledgements

Thanks to @luigidemeo1, @stephenbuttolph, @aaronbuchwald, @dhrubabasu, and @abi87 for their feedback on these ideas.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

