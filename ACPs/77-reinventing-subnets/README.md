| ACP | 77 |
| :--- | :--- |
| **Title** | Reinventing Subnets |
| **Author(s)** | Dhruba Basu ([@dhrubabasu](https://github.com/dhrubabasu)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/78)) |
| **Track** | Standards |
| **Replaces** | [ACP-13](../13-subnet-only-validators/README.md) |

## Abstract

Overhaul Subnet creation and management to unlock increased flexibility for Subnet creators by:

- Separating Subnet Validators from Primary Network Validators (Primary Network Partial Sync, Removal of 2000 $AVAX requirement)
- Moving ownership of Subnet Validator set management from P-Chain to Subnets (ERC-20/ERC-721/Arbitrary Staking, Staking Reward Management)
- Introducing a continuous P-Chain fee mechanism for Subnet Validators (Continuous Subnet Staking)

This ACP supersedes [ACP-13](../13-subnet-only-validators/README.md) and borrows some of its language.

## Motivation

Each node operator must stake at least 2000 $AVAX ($70k at time of writing) to first become a Primary Network Validator before they qualify to become a Subnet Validator. Most Subnets aim to launch with at least 8 Subnet Validators, which requires staking 16000 $AVAX ($560k at time of writing). All Subnet Validators, to satisfy their role as Primary Network Validators, must also [allocate 8 AWS vCPU, 16 GB RAM, and 1 TB storage](https://github.com/ava-labs/avalanchego/blob/master/README.md#installation) to sync the entire Primary Network (X-Chain, P-Chain, and C-Chain) and participate in its consensus, in addition to whatever resources are required for each Subnet they are validating.

Regulated entities that are prohibited from validating permissionless, smart contract-enabled blockchains (like the C-Chain) cannot launch a Subnet because they cannot opt-out of Primary Network Validation. This deployment blocker prevents a large cohort of Real World Asset (RWA) issuers from bringing unique, valuable tokens to the Avalanche Ecosystem (that could move between C-Chain <> Subnets using Avalanche Warp Messaging/Teleporter).

A widely validated Subnet that is not properly metered could destabilize the Primary Network if usage spikes unexpectedly. Underprovisioned Primary Network Validators running such a Subnet may exit with an OOM exception, see degraded disk performance, or find it difficult to allocate CPU time to P/X/C-Chain validation. The inverse also holds for Subnets with the Primary Network (where some undefined behavior could bring a Subnet offline).

Although the fee paid to the Primary Network to operate a Subnet does not go up with the amount of activity on the Subnet, the fixed, upfront cost of setting up a Subnet Validator on the Primary Network deters new projects that prefer smaller, even variable, costs until demand is observed. _Unlike L2s that pay some increasing fee (usually denominated in units per transaction byte) to an external chain for data availability and security as activity scales, Subnets provide their own security/data availability and the only cost operators must pay from processing more activity is the hardware cost of supporting additional load._

Elastic Subnets, introduced in [Banff](https://medium.com/avalancheavax/banff-elastic-subnets-44042f41e34c), enabled Subnet creators to activate Proof-of-Stake validation and uptime-based rewards using their own token. However, this token is required to be an ANT (created on the X-Chain) and locked on the P-Chain. All staking rewards were distributed on the P-Chain with the reward curve being defined in the `TransformSubnetTx` and, once set, was unable to be modified.

With no Elastic Subnets live on Mainnet, it is clear that Permissionless Subnets as they stand today could be more desirable. There are many successful Permissioned Subnets in production but many Subnet creators have raised the above as points of concern. In summary, the Avalanche community could benefit from a more flexible and affordable mechanism to launch Permissionless Subnets.

## Specification

### Separate Subnet Validators from Primary Network Validators

Subnet Validators will only be required to sync the P-chain (not X/C-Chain) to track any validator set changes in their Subnet and to support Cross-Subnet communication via Avalanche Warp Messaging (see "Primary Network Partial Sync" mode introduced in [Cortina 8](https://github.com/ava-labs/avalanchego/releases/tag/v1.10.8)). The lower resource requirement in this "minimal mode" will provide Subnets with greater flexibility of validation hardware requirements as operators are not required to reserve any resources for C-Chain/X-Chain operation. Since Subnet Validators are no longer required to validate or sync the Primary Network, the 2000 $AVAX requirement can be removed. To meter the number of Subnet Validators on the network, a significantly lower, continuous fee is introduced later in this ACP.

#### Bootstrapping Subnet Nodes

Bootstrapping a node/validator is the process of securely recreating the latest state of the blockchain locally. At the end of this process, the local state of a node/validator must be in sync with the local state of other virtuous nodes/validators. The node/validator can then verify new incoming transactions and reach consensus with other nodes/validators.

To bootstrap a node/validator, a few critical questions must be answered: How does one discover peers in the network? How does one determine that a discovered peer is honestly participating in the network?

For standalone networks like the Avalanche Primary Network, this is done by connecting to a hardcoded [set](https://github.com/ava-labs/avalanchego/blob/master/genesis/bootstrappers.json) of trusted bootstrappers to then discover new peers. Ethereum calls their set [bootnodes](https://ethereum.org/developers/docs/nodes-and-clients/bootnodes).

By separating Subnet Validators from Primary Network Validators, a list of validator IPs to connect to (the functional bootstrappers of the Subnet) is no longer provided by simply connecting to the Primary Network Validators. However, the Primary Network can enable nodes tracking a Subnet to seamlessly connect to the Subnet Validators by tracking and gossiping Subnet Validator IPs. Subnets will not need to operate and maintain a set of bootstrappers and can continue to rely on the Primary Network for peer discovery.

### Setting a Subnet Manager

To create a Subnet, a `CreateSubnetTx` must be issued on the P-Chain. This transaction includes an `Owner` field which defines the key that must be used to authorize any validator set additions (`AddSubnetValidatorTx`) or removals (`RemoveSubnetValidatorTx`).

To be a Permissionless Subnet, this `Owner` key must no longer have the ability to modify the Subnet's validator set. A `ConvertSubnetTx` must first be issued to explicitly convert a Subnet from Permissioned to Permissionless. This transaction will set the `(chainID, address)` pair that will manage the Subnet going forward. After `ConvertSubnetTx` is issued, the `Owner` from the `CreateSubnetTx` that created the Subnet will no longer have the ability to modify the Subnet's validator set.

To provide maximal flexibility for Permissionless Subnets, the BLS multisignature approach in Avalanche Warp Messaging (AWM) is re-used to approve modifications to the Subnet's validator set. Using the `(chainID, address)` pair defined in the `ConvertSubnetTx`, a Warp Message with an [`AddressedCall`](https://github.com/ava-labs/avalanchego/tree/master/vms/platformvm/warp/payload#addressedcall) payload can be constructed.

To validate an `AddressedCall` payload in Avalanche Warp Messaging, the `(chainID, address)` pair is used to lookup the validators of `chainID` and verify that the BLS multi-sig includes a quorum (set to 67%) of the Subnet's validator set. The P-Chain will use Warp Messages with `AddressedCall` payloads to support modifications of the Subnet's validator set using this `(chainID, address)` pair (using the `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` defined later in the specification).

#### ConvertSubnetTx

```golang
type SubnetValidator struct {
    // Must be Ed25519 NodeID
    NodeID ids.NodeID `json:"nodeID"`
    // Weight of this validator used when sampling
    Weight uint64 `json:"weight"`
    // Initial balance for this validator
    Balance uint64 `json:"balance"`
    // [Signer] is the BLS key for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID + Subnet does uniquely map to a BLS key
    Signer signer.Signer `json:"signer"`
    // Leftover $AVAX from the [Balance] will be issued to this
    // owner once it is removed from the validator set.
    ChangeOwner fx.Owner `json:"changeOwner"`
}

type ConvertSubnetTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the Subnet to transform
    // Restrictions:
    // - Must not be the Primary Network ID
    Subnet ids.ID `json:"subnetID"`
    // Chain where the Subnet manager lives
    ChainID ids.ID `json:"chainID"`
    // Address of the Subnet manager
    Address []byte `json:"address"`
    // Initial pay-as-you-go validators for the Subnet
    Validators []SubnetValidator `json:"validators"`
    // Authorizes this conversion
    SubnetAuth verify.Verifiable `json:"subnetAuthorization"`
}
```

Once this transaction is accepted, the `Owner` key is powerless. Additionally, `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` are disabled on the Subnet. `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` must be used to manage the Subnet's validator set going forward.

The `validationID` for validators added in `ConvertSubnetTx` is defined as the SHA256 hash of `(convertSubnetTxID,validatorIndex)`, where `validatorIndex` refers to the index into the `Validators` array of within the `ConvertSubnetTx`.

The following serialization is defined as an `InitialSubnetValidator`:
```text
+--------------+----------+-----------+
|       nodeID : [32]byte |  32 bytes |
+--------------+----------+-----------+
|       weight :   uint64 |   8 bytes |
+--------------+----------+-----------+
| blsPublicKey : [48]byte |  48 bytes |
+--------------+----------+-----------+
                          |  88 bytes |
                          +-----------+
```
The following serialization is defined as the `SubnetConversionData`
```text
+-------------------+--------------------------+--------------------------------------------------------------+
| convertSubnetTxID :                 [32]byte |                                                     32 bytes |
+-------------------+--------------------------+--------------------------------------------------------------+
|    managerChainID :                 [32]byte |                                                     32 bytes |
+-------------------+--------------------------+--------------------------------------------------------------+
|    managerAddress :                   []byte |                                4 + len(managerAddress) bytes |
+-------------------+--------------------------+--------------------------------------------------------------+
| initialValidators : []InitialSubnetValidator |                        4 + len(initialValidators) * 88 bytes |
+-------------------+--------------------------+--------------------------------------------------------------+
                                               | 72 + len(managerAddress) + len(initialValidators) * 88 bytes |
                                               +--------------------------------------------------------------+
```
The `subnetConversionID` is defined as the SHA256 hash of the `SubnetConversionData` from a given `ConvertSubnetTx`.

Once a `ConvertSubnetTx` is accepted, P-Chain validators will be willing to sign a `SubnetConversionMessage`, specified as an `AddressedCall` with a empty `originSenderAddress` with the following payload.
```text
+--------------------+----------+----------+
|            codecID :   uint16 |  2 bytes |
+--------------------+----------+----------+
|             typeID :   uint32 |  4 bytes |
+--------------------+----------+----------+
| subnetConversionID : [32]byte | 32 bytes |
+--------------------+----------+----------+
                                | 38 bytes |
                                +----------+
```

### Adding Subnet Validators

#### RegisterSubnetValidatorTx

After a `ConvertSubnetTx` has been accepted, the `RegisterSubnetValidatorTx` can be used to add Subnet validators.

```golang
type RegisterSubnetValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee.
    Balance uint64 `json:"balance"`
    // [Signer] is the BLS key for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID does uniquely map to a BLS key
    Signer signer.Signer `json:"signer"`
    // Leftover $AVAX from the Subnet Validator's Balance will be issued to
    // this owner after it is removed from the validator set.
    ChangeOwner fx.Owner `json:"changeOwner"`
    // AddressedCall with Payload:
    //   - SubnetID
    //   - NodeID (must be Ed25519 NodeID)
    //   - Weight
    //   - BLS public key
    //   - Expiry
    Message warp.Message `json:"message"`
}
```

The `Message` field must be an `AddressedCall` with the payload:

```text
+--------------+----------+-----------+
|      codecID :   uint16 |   2 bytes |
+--------------+----------+-----------+
|       typeID :   uint32 |   4 bytes |
+--------------+----------+-----------+
|     subnetID : [32]byte |  32 bytes |
+--------------+----------+-----------+
|       nodeID : [32]byte |  32 bytes |
+--------------+----------+-----------+
|       weight :   uint64 |   8 bytes |
+--------------+----------+-----------+
| blsPublicKey : [48]byte |  48 bytes |
+--------------+----------+-----------+
|       expiry :   uint64 |   8 bytes |
+--------------+----------+-----------+
                          | 134 bytes |
                          +-----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000000` for this transaction
- `subnetID`, `nodeID`, `weight`, and `blsPublicKey` are for the Subnet Validator being added
- `expiry` is the time after which this message is invalid. After the P-Chain timestamp is past `expiry`, this Avalanche Warp Message can no longer be used to add the `nodeID` to the validator set of `subnetID`.

    `validationID` of validators added via `RegisterSubnetValidatorTx` is defined as the SHA256 hash of the `Payload` of the `AddressedCall`. This SHA256 hash will be used for replay protection. Used `validationID`s will be stored on the P-Chain. If a `RegisterSubnetValidatorTx`'s `validationID` has already been used, the transaction will be considered invalid. To prevent storing an unbounded number of `validationID`s, the `expiry` is required to be no longer than 48 hours in the future of the time the transaction is issued on the P-Chain. Any `validationIDs` with an `expiry` in the past can be flushed from the P-Chain's state.

Subnets are responsible for defining the procedure on how to retrieve the above information from prospective validators.

An EVM Subnet may choose to implement this step like so:

- Use the number of tokens the user has staked into a smart contract on the Subnet to determine the weight of their validator
- Require the user to submit an on-chain transaction with their validator information
- Generate the Warp message

After the `RegisterSubnetValidatorTx` is accepted on the P-Chain, the Subnet Validator is added to the Subnet's validator set. A `minNonce` field corresponding to the `validationID` will be stored on addition to the validator set (initially set to `0`). This field will be used when validating the `SetSubnetValidatorWeightTx` defined below.

For a `RegisterSubnetValidatorTx` to be valid:

- `Balance` must be >= the greater of 5 $AVAX or two weeks of the current fee

    This prevents Subnet Validators from being added with too low of an initial balance where they become immediately delinquent based on the continuous fee mechanism defined below. A Subnet Validator can leave at any time before the initial $AVAX is consumed and claim the remaining balance to the `ChangeOwner` defined in the transaction.

- `Signer` must correspond to the `blsPublicKey` defined in the warp message

Note: There is no `EndTime` specified in this transaction. Subnet Validators are only removed when a `SetSubnetValidatorWeightTx` sets a validator's weight to `0`.

### Modifying Subnet Validators

#### SetSubnetValidatorWeightTx

`SetSubnetValidatorWeightTx` is used to modify the voting weight of a Subnet Validator. For this transaction to be valid, 67% of the current Subnet Validator weight must sign the Warp message included in `Message`. Applications of this transaction could include:

- Increase the voting weight of a Subnet Validator if a delegation is made on the Subnet
- Increase the voting weight of a Subnet Validator if the stake amount is increased (by staking rewards for example)
- Decrease the voting weight of a misbehaving Subnet Validator
- Remove an inactive Subnet Validator

Since there are no `EndTime`s enforced by the P-Chain, Subnets must use this transaction to set an expired validator's weight to `0`. When a validator's weight is set to `0`, they will be removed from the validator set. After the transaction is accepted, any $AVAX remaining in the `Balance` for the Subnet Validator being removed will be returned to the `ChangeOwner` defined in the `RegisterSubnetValidatorTx`.

```golang
type SetSubnetValidatorWeightTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // AddressedCall with Payload:
    //   - ValidationID (SHA256 of the AddressedCall Payload of the RegisterSubnetValidatorTx adding the validator)
    //   - Nonce
    //   - Weight
    Message warp.Message `json:"message"`
}
```

The `Message` field must be an `AddressedCall` with the payload:

```text
+--------------+----------+----------+
|      codecID :   uint16 |  2 bytes |
+--------------+----------+----------+
|       typeID :   uint32 |  4 bytes |
+--------------+----------+----------+
| validationID : [32]byte | 32 bytes |
+--------------+----------+----------+
|        nonce :   uint64 |  8 bytes |
+--------------+----------+----------+
|       weight :   uint64 |  8 bytes |
+--------------+----------+----------+
                          | 54 bytes |
                          +----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000001` for this transaction
- `validationID` is the SHA256 of the `Payload` of the `AddressedCall` in the `RegisterSubnetValidatorTx` adding the validator to the Subnet's validator set
- `nonce` is a strictly increasing number that denotes the latest validator weight update and provides replay protection for this transaction

    The P-Chain state will store a `minNonce` associated with the `validationID`. When accepting the `RegisterSubnetValidatorTx`, the `minNonce` will be set to `0` for `validationID`. `nonce` must satisfy `nonce >= minNonce` for the `SetSubnetValidatorWeightTx` to be valid. Note that `nonce` is not required to be incremented by `1` with each successive validator weight update. If `minNonce` is `MaxUint64`, the `weight` in the `SetSubnetValidatorWeightTx` is required to be `0` to prevent Subnets from being unable to remove `nodeID` in a subsequent `SetSubnetValidatorWeightTx`. When a Subnet Validator is removed from the active validator set (`weight == 0`), the `minNonce` and `validationID` will be removed from the P-Chain state. This state can be reaped during validator removal since `validationID` can never be re-initialized as a result of the replay protection provided by `expiry` in `RegisterSubnetValidatorTx`. `minNonce` will be set to `nonce + 1` when `SetSubnetValidatorWeightTx` is executed and `weight != 0`
- `weight` is the new `weight` of the validator

### Removing Subnet Validators

#### SetSubnetValidatorWeightTx (Weight = 0)

Issuing a `SetSubnetValidatorWeightTx` with `Weight` of `0` will remove the Subnet Validator from the Subnet's validator set.

All state related to the Subnet Validator being removed will be removed from the P-Chain's active state (BLS key, NodeID etc). This validator can be newly added to the Subnet's validator set using the `RegisterSubnetValidatorTx` flow.

Since altering a Subnet's validator set after a `ConvertSubnetTx` requires a valid Warp message from the Subnet's current validator set, it is explicitly disallowed for a Subnet to remove it's only validator, which would render the Subnet unable to produce any further valid Warp messages. If a Subnet only has a single validator, it is invalid to set that validator's weight to 0 via a `SetSubnetValidatorWeightTx`.

### Disabling Subnet Validators

#### DisableValidatorTx

Subnet Validators can use `DisableValidatorTx` to mark their validator as inactive. Notably, there is no requirement of any interaction on the Subnet to issue this transaction. The only requirement is an Ed25519 Signature (signed with the Subnet Validator's Ed25519 private key) for this transaction to be valid. Any remaining $AVAX in the Subnet Validator's `Balance` will be issued to the `ChangeOwner` defined when this validator was added to the validator set.

The expected path for full removal from a Subnet's validator set is via a `SetSubnetValidatorWeightTx` with weight `0` which requires a Warp message produced by the Subnet. However, the ability to stop participating in Subnet validation is critical for censorship-resistance and/or failed Subnets. If a Subnet Validator wishes to stop participating in its Subnet's consensus, they can do so through this transaction. This is enabled on the P-Chain to prevent Subnets from locking Subnet Validators into participating in consensus indefinitely.

Note that this does not modify a Subnet's total staking weight. This transaction only marks the Subnet Validator as inactive but does not remove it from the Subnet's validator set. Inactive Subnet Validators can re-activate at any time by increasing their balance with an `IncreaseBalanceTx`.

Subnet creators should be aware that there is no notion of `MinStakeDuration` that is enforced by the P-Chain. It is expected that Subnets who choose to enforce a `MinStakeDuration` will lock the validator's Stake for the Subnet's desired `MinStakeDuration`.

```golang
type DisableValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID corresponding to the validator
    ValidationID ids.ID `json:"validationID"`
    // Ed25519 Signature on [TxID]
    Signature []byte `json:"signature"`
}
```

### Proof of Subnet Validator Set Change

To track whether a Subnet Validator addition/modification/removal occured on the P-Chain, Subnet Validators must be willing to sign an `AddressedCall` attesting to the validator set change. Since all Subnet Validators sync the P-Chain, only the Subnet Validators need to sign the `AddressedCall`, not all Primary Network Validators.

Two `AddressedCall`s are defined with the below payloads. For each of them, the `sourceChainID` must be set to the P-Chain ID and the `sourceAddress` must be set to an empty byte array.

The method of requesting is left unspecified as it can be more generic. A viable option for supporting this functionality is laid out in [ACP-118](../118-warp-signature-request/README.md) with the `SignatureRequest` message.

#### SubnetValidatorRegistrationMessage

```text
+--------------+----------+----------+
|      codecID :   uint16 |  2 bytes |
+--------------+----------+----------+
|       typeID :   uint32 |  4 bytes |
+--------------+----------+----------+
| validationID : [32]byte | 32 bytes |
+--------------+----------+----------+
|   registered :     bool |  1 byte  | 
+--------------+----------+----------+
                          | 39 bytes |
                          +----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000002` for this message
- `validationID` is the SHA256 of the `Payload` of the `AddressedCall` in the `RegisterSubnetValidatorTx` adding the validator to the Subnet's validator set
- `registered` indicates whether or not the `validationID` corresponds to a valid `AddressedCall` payload. If true, `validationID` corresponds to an active validator. If false, `validationID` does not correspond to an active validator, and never will as the `expiry` in the `AddressedCall` payload is in the past.

The P-Chain nodes must refuse to sign any `SubnetValidatorRegistrationMessage` where the `validationID` does not correspond to an active validator and the `expiry` is in the future.

#### SubnetValidatorWeightUpdateMessage

```text
+--------------+----------+----------+
|      codecID :   uint16 |  2 bytes |
+--------------+----------+----------+
|       typeID :   uint32 |  4 bytes |
+--------------+----------+----------+
| validationID : [32]byte | 32 bytes |
+--------------+----------+----------+
|        nonce :   uint64 |  8 bytes |
+--------------+----------+----------+
|       weight :   uint64 |  8 bytes |
+--------------+----------+----------+
                          | 54 bytes |
                          +----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000003` for this message
- `validationID` is the SHA256 of the `Payload` of the `AddressedCall` in the `RegisterSubnetValidatorTx` adding the validator to the Subnet's validator set
- `nonce` is the latest nonce stored with the `validationID` on the P-Chain
- `weight` is the current weight of the Subnet Validator with `validationID`

### Managing Subnet Validator Balance

#### IncreaseBalanceTx

This transaction can be issued by anybody to add additional $AVAX to the `Balance` for `NodeID` validating `Subnet`. If the Subnet Validator corresponding to `ValidationID` is currently inactive (`Balance` was exhausted or `DisableValidatorTx` was issued), this transaction will move them back to the active validator set.

Note: The $AVAX added to `Balance` can be claimed at any time by the Subnet Validator using `DisableValidatorTx`.

```golang
type IncreaseBalanceTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID corresponding to the validator
    ValidationID ids.ID `json:"validationID"`
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee
    Balance uint64 `json:"balance"`
}
```

### Sidebar: Subnet Sovereignty

After this ACP is activated, the P-Chain will no longer support staking of any assets other than $AVAX for the Primary Network. The P-Chain will no longer support distribution of staking rewards for Subnets. All staking-related operations for Subnet Validation must be managed by the Subnet on the Subnet. The P-Chain simply requires a continuous fee per Subnet Validator. If a Subnet would like to manage their Validator's balances on the P-Chain, it can cover the cost for all Subnet Validators by posting the $AVAX balance on the P-Chain. Subnets can implement any mechanism they want to pay the continuous fee charged by the P-Chain for its participants.

By moving ownership of the Subnet's validator set from the P-Chain to the Subnet, Subnet creators have no restrictions on what requirements they have to join their Subnet as a validator. Any stake that is required to join the Subnet's validator set is locked on the Subnet. If a validator is removed from the Subnet's validator set via a `SetSubnetValidatorWeightTx` with weight `0`, the stake on the Subnet will continue to be locked. How each Subnet handles stake associated with the Subnet Validator is entirely left up to the Subnet and can be treated independently to what happens on the P-Chain.

This new relationship between the P-Chain and Subnets provides a dynamic where Subnets can use the P-Chain as an impartial judge to modify parameters (in addition to its existing role of helping to validate incoming Avalanche Warp Messages). If a Validator is misbehaving, the Subnet Validators can collectively generate a BLS multisig to reduce the voting weight of a misbehaving validator. This operation is fully secured by the Avalanche Primary Network (225M $AVAX or $8.325B at the time of writing).

Follow-up ACPs could extend the P-Chain <> Subnet relationship to include parametrization of the 67% threshold to enable Subnets to choose a different threshold based on their security model (e.g. a simple majority of 51%).

### Continuous Fee Mechanism

Currently, the P-Chain operates under a fixed fee mechanism for each transaction type. There are discussions to introduce a [dynamic multi-dimensional fee](https://github.com/avalanche-foundation/ACPs/discussions/69) to the P-Chain. The total number of Subnet Validators on the P-Chain should be considered in that dynamic fee mechanism. Through these dimensions, a standard charge that all Subnet Validators will pay will be calculated by the network based on network activity, commonly referred to as a "base fee". This base fee will move up when the total number of Subnet Validators is above target utilization and move down when the total number of Subnet Validators is below target utilization.

Each additional Subnet Validator on the P-Chain adds load to the network. The discussions for the dynamic multi-dimensional fee charge when a transaction is instantiated but there isn't a solution for charging Subnet Validators for the space they are using over the time they are validating on the network (which may be indefinitely). This is a common problem in blockchains and there have been many state rent proposals in the broader blockchain space to address it. This fee mechanism takes advantage of the fact that each Subnet Validator uses the same amount of state and charges each Subnet Validator the dynamic base fee for every discrete unit of time it is active.

To charge each Subnet Validator, the notion of a `Balance` is introduced. The `Balance` of a Subnet Validator will be continuously charged during the time they are active to cover the cost of storing the associated validator properties (BLS key, weight, nonce) in memory and to track IPs (in addition to other services provided by the Primary Network). This `Balance` is initialized with the `RegisterSubnetValidatorTx` that added them to the active validator set. `Balance` can be increased at any time using the `IncreaseBalanceTx`. When this `Balance` reaches `0`, the Subnet Validator will be considered "inactive" and will no longer participate in validating the Subnet. Inactive Subnet Validators can be moved back to the active validator set at any time using the same `IncreaseBalanceTx`. Once a Subnet Validator is considered inactive, the P-Chain will remove these properties from memory and only retain them on disk. All messages from that validator will be considered invalid until it is revived using the `IncreaseBalanceTx`. Subnets can reduce the amount of inactive weight by removing inactive validators with the `SetSubnetValidatorWeightTx` (`Weight` = 0).

Since each Subnet Validator is charged the same amount at each point in time, tracking the fees for the entire validator set is straight-forward. The accumulated dynamic base fee for the entire network is tracked in a single uint. This accumulated value should be equal to the fee charged if a Subnet Validator was active from the time the accumulator was instantiated. The validator set is maintained in a priority queue. A pseudocode implementation of the continuous fee mechanism is provided below.

```python
# Pseudocode
class ValidatorQueue:
    def __init__(self, fee_getter):
        self.acc = 0
        self.queue = PriorityQueue()
        self.fee_getter = fee_getter
    
    # At each time period, increment the accumulator and 
    # pop all validators from the top of the queue that 
    # ran out of funds.
    
    # Note: The amount of work done in a single block
    # should be bounded to prevent a large number of
    # validator operations from happening at the same
    # time.
    def time_elapse(self, t):
        self.acc = self.acc + self.fee_getter(t)
        while True:
            vdr = self.queue.peek()
            if vdr.balance < self.acc:
                self.queue.pop()
                continue
            return

    # Validator was added
    def validator_enter(self, vdr):
        vdr.balance = vdr.balance + self.acc
        self.queue.add(vdr)

    # Validator was removed
    def validator_remove(self, vdrNodeID):
        vdr = find_and_remove(self.queue, vdrNodeID)
        vdr.balance = vdr.balance - self.acc
        vdr.refund() # Refund [vdr.balance] to [ChangeOwner]
        self.queue.remove()

    # Validator's balance was topped up
    def validator_increase(self, vdrNodeID, balance):
        vdr = find_and_remove(self.queue, vdrNodeID)
        vdr.balance = vdr.balance + balance
        self.queue.add(vdr)
```

#### User Experience

Instead of a fixed up-front cost of 2000 $AVAX, Subnet Validators are now continuously charged a fee, albeit a small one. This poses a new challenge for Subnet Validators: How do they maintain the Subnet Validator balance?

Node clients should expose an API to track how much balance is remaining in the Subnet Validator's account. This will provide a way for Subnet Validators to track how quickly it is going down and top-up when needed. A nice byproduct of the above design is the balance in the Subnet Validator's account is claimable. This means users can top-up as much $AVAX as they want and rest-assured knowing they can always retrieve it if there is an excessive amount.

The expectation is that most users will not interact with node clients or track when or how much they need to top-up their Subnet Validator account. Wallet providers will abstract away most of this process. For users who desire more convenience, Subnet-as-a-Service providers will abstract away all of this process.

## Backwards Compatibility

This new design for Subnets proposes a large re-work to all Subnet-related mechanics. Rollout should be done on a going-forward basis to not cause any service disruption for live Subnets. All current Subnet Validators will be able to continue validating both the Primary Network and whatever Subnets they are validating.

Any state execution changes must be coordinated through a mandatory upgrade. Implementors must take care to continue to verify the existing ruleset until the upgrade is activated. After activation, nodes should verify the new ruleset. Implementors must take care to only verify the presence of 2000 $AVAX prior to activation.

### Deactivated Transactions

- P-Chain
  - `TransformSubnetTx`

    After this ACP is activated, Elastic Subnets will be disabled. `TransformSubnetTx` will not be accepted post-activation. As there are no Mainnet Elastic Subnets, there should be no production impact with this deactivation.

### New Transactions

- P-Chain
  - `ConvertSubnetTx`
  - `RegisterSubnetValidatorTx`
  - `SetSubnetValidatorWeightTx`
  - `DisableValidatorTx`
  - `IncreaseBalanceTx`

Once a Subnet issues a `ConvertSubnetTx`, `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` can no longer be used to modify that Subnet's validator set. A Subnet Validator added with an `AddSubnetValidatorTx` will continue to validate the Subnet until their `EndTime` is reached. After expiry, those validators (if they want to continue validating) must use the `RegisterSubnetValidatorTx` flow outlined in this ACP to register as a Subnet Validator.

## Reference Implementation

A full reference implementation has not been provided yet. It will be provided once this ACP is considered `Implementable`.

## Security Considerations

This ACP significantly reduces the cost of becoming a Subnet Validator. This can lead to a large increase in the number of Subnet Validators going forward. Each additional Subnet Validator adds consistent RAM usage to the P-Chain. However, this should be appropriately metered by the continuous fee mechanism outlined above.

With the additional sovereignty Subnets gain from the P-Chain, Subnet staking tokens are no longer locked on the P-Chain for Permissionless Subnets. This poses a new security consideration for Subnet Validators: Malicious Subnets can choose to remove validators at will and take any funds that the Subnet Validator has on the Subnet. The P-Chain only provides the guarantee that Subnet Validators can retrieve the remaining $AVAX Balance for their Validator via a `DisableValidatorTx`. Any assets on the Subnet is entirely under the purview of the Subnet. The onus is now on Subnet Validators to vet the Subnet's security.

With a long window of expiry (48 hours) for the Warp message in `RegisterSubnetValidatorTx`, spam of Subnet Validator registration could lead to high memory pressure on the P-Chain. A future ACP can reduce the window of expiry if 48 hours proves to be a problem.

NodeIDs can be added to a Subnet's validator set involuntarily. However, it is important to note that any stake/rewards are _not_ at risk. For a node operator who was added to a validator set involuntarily, they would only need to generate a new NodeID via key rotation as there is no lock-up of any stake to create a NodeID. This is an explicit tradeoff for easier on-boarding of NodeIDs. This mirrors the Primary Network Validators guarantee of no stake/rewards at risk.

The continuous fee mechanism outlined above does not apply to inactive Subnet Validators since they are not stored in memory. However, inactive Subnet Validators are persisted on disk which can lead to persistent P-Chain state growth. The initial balance requirement (the greater of 5 $AVAX or two weeks of the current fee) should be a sufficient deterrent as it must be fully burned for the Subnet Validator to become inactive. A future ACP can increase the initial balance to decrease the rate of P-Chain state growth or provide a state expiry path to reduce the amount of P-Chain state.

## Open Questions

### Should they still be called Subnets?

Through this ACP, Subnets have far more sovereignty than they did before. The Avalanche Community could take this opportunity to re-brand Subnets.

## Acknowledgements

Special thanks to [@StephenButtolph](https://github.com/StephenButtolph), [@aaronbuchwald](https://github.com/aaronbuchwald), and [@patrick-ogrady](https://github.com/patrick-ogrady) for their feedback on these ideas. Thank you to the broader Ava Labs Platform Engineering Group for their feedback on this ACP prior to publication.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
