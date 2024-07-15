```text
ACP: 77
Title: Reinventing Subnets
Author(s): Dhruba Basu <https://github.com/dhrubabasu>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/78
Status: Proposed
Track: Standards
Replaces: 13
```

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

### New Registration Flow

#### Step 1: Retrieve a BLS multisig from the Subnet

To provide increased optionality over Subnet Validator requirements, the P-Chain will only require an Avalanche Warp Message using the `AddressedCall` format with the below payload to join the Subnet Validator set. A `typeID` for distinguishing between the different payload types on the P-Chain must be included prior to this ACP being considered "Implementable".

```text
+-----------+----------+-----------+
|  subnetID : [32]byte |  32 bytes |
+-----------+----------+-----------+
|    nodeID : [32]byte |  32 bytes |
+-----------+----------+-----------+
|    weight :   uint64 |   8 bytes |
+-----------+----------+-----------+
|    expiry :   uint64 |   8 bytes |
+-----------+----------+-----------+
| signature : [64]byte |  64 bytes |
+-----------+----------+-----------+
                       | 144 bytes |
                       +-----------+
```

- `subnetID`, `nodeID`, and `weight` are for the Subnet Validator being added
- `expiry` is the time after which this message is invalid. After the P-Chain timestamp is past `expiry`, this Avalanche Warp Message can no longer be used to add the `nodeID` to the validator set of `subnetID`. For replay protection, the P-Chain will store used `messageID`s (hash of the entire Avalanche Warp Message). To prevent the P-Chain from having to store an unbounded number of `messageID`s, the `expiry` is required to be no longer than 48 hours in the future of the time the transaction is issued on the P-Chain. After `RegisterSubnetValidatorTx` is accepted, the P-Chain state will store a `minNonce` of `0` corresponding to the warp `messageID`. The `minNonce` will be used to validate the `SetSubnetValidatorWeightTx` below.
- `signature` is the raw bytes of the Ed25519 signature over the concatenated bytes of `[subnetID]+[nodeID]+[blsPublicKey]+[weight]+[balance]+[expiry]`. This signature must correspond to the Ed25519 public key that is used for the `nodeID`. This approach prevents NodeIDs from being unwillingly added to Subnets. `balance` is the minimum initial $nAVAX balance that must be attached to the validator serialized as a uint64. The `signature` field will be validated by the P-Chain in Step 2. A Subnet may choose to validate that the `signature` field is well-formed but it is not required.

Subnets are responsible for defining the procedure on how to retrieve the above information from prospective validators.

An EVM Subnet may choose to implement this step like so:

- Use the number of tokens the user has staked into a smart contract on the Subnet to determine the weight of their validator
- Require the user to submit an on-chain transaction with their validator information
- Generate the warp message

#### Step 2: Issue a `RegisterSubnetValidatorTx` on the P-Chain

```golang
type RegisterSubnetValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee
    Balance uint64 `json:"balance"`
    // [Signer] is the BLS key for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID does uniquely map to a BLS key
    Signer signer.Signer `json:"signer"`
    // Leftover $AVAX from the [Balance] will be issued to this
    // owner after it is removed from the validator set.
    ChangeOwner fx.Owner `json:"changeOwner"`
    // Warp message should include:
    //   - SubnetID
    //   - NodeID (must be Ed25519 NodeID)
    //   - Weight of the validator
    //   - Expiry
    //   - Ed25519 Signature over "[subnetID]+[nodeID]+[blsPublicKey]+[weight]+[balance]+[expiry]"
    //   - BLS multisig over the above payload
    Message warp.Message `json:"message"`
}
```

After the `RegisterSubnetValidatorTx` is accepted on the P-Chain, the Subnet Validator is added to the Subnet's validator set. The P-Chain will then send a Warp message to the Subnet notifying of the validator set addition.

`RegisterSubnetValidatorTx` will be considered invalid if the `NodeID` included in the message is already a validator for the given `SubnetID`.

When any validator is removed from the set (whether forcefully or per the validator's request), the P-Chain will also send a warp message to the Subnet notifying it of the validator set removal. It is up to the Subnet on how to handle such a message, especially if unexpected. A validator's stake could continue to remain locked for an extended period of time after this point, for example.

This transaction is, by design, not required to be submitted by the validator themselves. With the Ed25519 signature, the validator guarantees that they can only be added to the validator set if `Signer` corresponds to the `blsPublicKey` and `Balance` >= `balance`. The `RegisterSubnetValidatorTx` is considered invalid if those two properties are not satisfied.

For a `RegisterSubnetValidatorTx` to be valid, `Balance` must be >= the greater of 5 $AVAX or two weeks of the current fee. This prevents Subnet Validators from being added with too low of a `Balance` where they become immediately delinquent based on the continous fee mechanism defined below. A Subnet Validator can leave at any time before the initial $AVAX is consumed and claim the remaining `Balance` to the `ChangeOwner` defined in the transaction.

Note: There is no `EndTime` specified in this transaction. Subnet Validators are only removed when a `SetSubnetValidatorWeightTx` sets a validator's weight to `0` or `ExitValidatorTx` is issued.

#### SetSubnetValidatorWeightTx

`SetSubnetValidatorWeightTx` is used to modify the voting weight of a Subnet Validator. For this transaction to be valid, 67% of the current Subnet Validator weight must sign the warp message included in `Message`. Applications of this transaction could include:

- Increase the voting weight of a Subnet Validator if a delegation is made on the Subnet
- Increase the voting weight of a Subnet Validator if the stake amount is increased (by staking rewards for example)
- Decrease the voting weight of a misbehaving Subnet Validator
- Remove an inactive Subnet Validator

Since there are no `EndTime`s enforced by the P-Chain, Subnets must use this transaction to set an expired validator's weight to `0`. When a validator's weight is set to `0`, they will be removed from the validator set. After the transaction is accepted, any $AVAX remaining in the `Balance` for the Subnet Validator being removed will be returned to the `ChangeOwner` defined in the `RegisterSubnetValidatorTx`.

```golang
type SetSubnetValidatorWeightTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Warp message should include:
    //   - MessageID corresponding to the warp message in the RegisterSubnetValidatorTx that added the validator (SHA256)
    //   - Nonce
    //   - Weight of the validator
    //   - BLS multisig over the above payload
    Message warp.Message `json:"message"`
}
```

The `Message` field in the above transaction must be an Avalanche Warp Message using the `AddressedCall` format with the below payload. A `typeID` for distinguishing between the different payload types on the P-Chain must be included prior to this ACP being considered "Implementable".

```text
+-----------+----------+----------+
| messageID : [32]byte | 32 bytes |
+-----------+----------+----------+
|     nonce :   uint64 |  8 bytes |
+-----------+----------+----------+
|    weight :   uint64 |  8 bytes |
+-----------+----------+----------+
                       | 48 bytes |
                       +----------+
```

- `messageID` is the sha256 hash of the warp message in the `RegisterSubnetValidatorTx` that registered the Subnet Validator
- `nonce` is a strictly increasing number that denotes the latest validator weight update and provides replay protection for this transaction. The P-Chain state will store a `minNonce` associated with `messageID`. When accepting the `RegisterSubnetValidatorTx`, the `minNonce` was set to `0` for `messageID`. `nonce` must satisfy `nonce >= minNonce` for `SetSubnetValidatorWeightTx` to be valid. Note that `nonce` is not required to be incremented by `1` with each successive validator weight update. If `minNonce` is `MaxUint64`, the `weight` in the `SetSubnetValidatorWeightTx` is required to be `0` to prevent Subnets from being unable to remove `nodeID` in a future transaction. When a Subnet Validator is removed from the active set (`weight == 0`), the `minNonce` and `messageID` will be removed from the P-Chain state. This state can be reaped during validator exit since `messageID` can never be re-initialized as a result of the replay protection provided by `expiry` in `RegisterSubnetValidatorTx`. `minNonce` will be set to `nonce + 1` when `SetSubnetValidatorWeightTx` is executed and `weight != 0`.

#### ExitValidatorSetTx

Subnet Validators can use `ExitValidatorSetTx` to exit the validator set without interacting with the Subnet. An Ed25519 Signature is required using the Subnet Validator's Ed25519 private key for this transaction to be valid. Remaining $AVAX in the Subnet Validator's `Balance` will be issued to the `ChangeOwner` defined when adding this validator to the validator set.

This transaction can be issued on the P-Chain without explicit consent of the Subnet. It is expected that validators should be removed from the Subnet's validator set through a `SetSubnetValidatorWeightTx` with weight `0` by initiating Subnet Validator removal on the Subnet. However, the ability to exit a Subnet Validator set is critical for censorship-resistance and/or failed Subnets. If a validator ever wishes to stop participating in Subnet consensus, they will be able to do so through this transaction. This is enforced on the P-Chain to prevent Subnets from locking Validators into participating in consensus indefinitely.

Subnet creators should be aware that there is no notion of `MinStakeDuration` that is enforced by the P-Chain. It is expected that Subnets who choose to enforce a `MinStakeDuration` will lock the validator's Stake for the Subnet's desired `MinStakeDuration`.

```golang
type ExitValidatorSetTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the tx that created the validator being removed
    TxID ids.ID `json:"txID"`
    // Ed25519 Signature on [TxID]
    Signature []byte `json:"signature"`
}
```

#### SetSubnetValidatorManagerTx

To participate in this new registration flow, Subnets must set the `(blockchainID, address)` pair by using `SetSubnetValidatorManagerTx`. This pair will be used to validate the `AddressedCall` in `SetSubnetValidatorWeightTx` and `RegisterSubnetValidatorTx`.

When a Subnet is created with `CreateSubnetTx`, a validator manager is not specified. To solve this problem, the first `SetSubnetValidatorManagerTx` will expect the Warp `AddressedCall` to have the `sourceChainID` be set to the `SubnetID` and the `sourceAddress` to be zero'd out. Once this transaction is accepted, `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` are enabled for use on the Subnet.

```golang
type SetSubnetValidatorManagerTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Warp message should include:
    //   - SubnetID
    //   - ChainID (where the validator manager lives)
    //   - Address (address of the validator manager)
    //   - BLS multisig over the above payload
    Message warp.Message `json:"message"`
}
```

A `typeID` for distinguishing between the different payload types on the P-Chain must be included prior to this ACP being considered "Implementable".

```text
+----------+----------+-------------------+
| subnetID : [32]byte |          32 bytes |
+----------+----------+-------------------+
|  chainID : [32]byte |          32 bytes |
+----------+----------+-------------------+
|  address :   []byte |  4 + len(address) |
+----------+----------+-------------------+
                      | 64 + len(address) |
                      +-------------------+
```

- `subnetID` is the Subnet that is being modified
- `chainID` and `address` is the location of the new validator manager for `subnetID`

#### IncreaseBalanceTx

This transaction can be issued by anybody to add additional $AVAX to the `Balance` for `NodeID` validating `Subnet`.

Note: The $AVAX added to `Balance` can be claimed by the Subnet Validator using `ExitValidatorSetTx`.

```golang
type IncreaseBalanceTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Balance <= sum($AVAX inputs) - sum($AVAX outputs) - TxFee
    Balance uint64 `json:"balance"`
    // Must be Ed25519 NodeID
    NodeID ids.NodeID `json:"nodeID"`
    Subnet ids.ID     `json:"subnetID"`
}
```

#### Sidebar: Subnet Sovereignty

After this ACP is activated, the P-Chain will no longer support staking of any assets other than $AVAX for the Primary Network. The P-Chain will no longer support distribution of staking rewards for Subnets. All staking-related operations for Subnet Validation must be managed by the Subnet on the Subnet. The P-Chain simply requires a continuous fee per Subnet Validator. If a Subnet would like to manage their Validator's balances on the P-Chain, it can cover the cost for all Subnet Validators by posting the $AVAX balance on the P-Chain. Subnets can implement any mechanism they want to pay the continuous fee charged by the P-Chain for its participants.

By moving ownership of the Subnet's validator set from the P-Chain to the Subnet, Subnet creators have no restrictions on what requirements they have to join their Subnet as a validator. Any stake that is required to join the Subnet Validator set is locked on the Subnet. If a validator is removed from the Subnet Validator set via a `SetSubnetValidatorWeightTx` with weight `0` or an `ExitValidatorSetTx` on the P-Chain, the stake on the Subnet will continue to be locked. How each Subnet handles stake associated with the Subnet Validator is entirely left up to the Subnet and can be treated independently to what happens on the P-Chain.

This new relationship between the P-Chain and Subnets provides a dynamic where Subnets can use the P-Chain as an impartial judge to modify parameters (in addition to its existing role of helping to validate incoming Avalanche Warp Messages). If a Validator is misbehaving, the Subnet Validators can collectively generate a BLS multisig to reduce the voting weight of a misbehaving validator. This operation is fully secured by the Avalanche Primary Network (225M AVAX or $8.325B at the time of writing).

Follow-up ACPs could extend the P-Chain <> Subnet relationship to include parametrization of the 67% threshold in `SetSubnetValidatorWeightTx` to enable Subnets to choose a different threshold based on their security model (e.g. a simple majority of 51%).

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
    def validator_exit(self, vdrNodeID):
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

After this ACP is activated, Elastic Subnets will be disabled. `TransformSubnetTx` will not be accepted post-activation. Any existing Elastic Subnet will be deleted with all Subnet assets being refunded to the `SubnetAuth` defined in the `TransformSubnetTx`.

As there are no Mainnet Elastic Subnets, there should be no production impact with this deactivation.

### New Transactions

- P-Chain
  - `RegisterSubnetValidatorTx`
  - `SetSubnetValidatorWeightTx`
  - `ExitValidatorSetTx`
  - `SetSubnetValidatorManagerTx`
  - `IncreaseBalanceTx`

Once a Subnet issues a `SetSubnetValidatorManagerTx`, `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` can no longer be used to modify that Subnet's validator set. A Subnet Validator added with an `AddSubnetValidatorTx` will continue to validate the Subnet until their `EndTime` is reached. After expiry, those validators (if they want to continue validating) must use the `RegisterSubnetValidatorTx` flow outlined in this ACP to register as a Subnet Validator.

## Reference Implementation

A full reference implementation has not been provided yet. It will be provided once this ACP is considered `Implementable`.

## Security Considerations

This ACP significantly reduces the cost of becoming a Subnet Validator. This can lead to a large increase in the number of Subnet Validators going forward. Each additional Subnet Validator adds consistent RAM usage to the P-Chain. However, this should be appropriately metered by the continuous fee mechanism outlined above.

With the additional sovereignty Subnets gain from the P-Chain, Subnet staking tokens are no longer locked on the P-Chain for Permissionless Subnets. This poses a new security consideration for Subnet Validators: Malicious Subnets can choose to remove validators at will and take any funds that the Subnet Validator has on the Subnet. The P-Chain only provides the guarantee that Subnet Validators can retrieve the remaining $AVAX Balance for their Validator via an `ExitValidatorSetTx`. Any assets on the Subnet is entirely under the purview of the Subnet. The onus is now on Subnet Validators to vet the Subnet's security.

With a long window of expiry (48 hours) for the warp message in `RegisterSubnetValidatorTx`, spam of Subnet Validator registration could lead to high memory pressure on the P-Chain. A future ACP can reduce the window of expiry if 48 hours proves to be a problem.

The continuous fee mechanism outlined above does not apply to inactive Subnet Validators since they are not stored in memory. However, inactive Subnet Validators are persisted on disk which can lead to persistent P-Chain state growth. The initial balance requirement (the greater of 5 $AVAX or two weeks of the current fee) should be a sufficient deterrent as it must be fully burned for the Subnet Validator to become inactive. A future ACP can increase the initial balance to decrease the rate of P-Chain state growth or provide a state expiry path to reduce the amount of P-Chain state.

## Open Questions

### Should they still be called Subnets?

Through this ACP, Subnets have far more sovereignty than they did before. The Avalanche Community could take this opportunity to re-brand Subnets.

## Acknowledgements

Special thanks to [@StephenButtolph](https://github.com/StephenButtolph), [@aaronbuchwald](https://github.com/aaronbuchwald), and [@patrick-ogrady](https://github.com/patrick-ogrady) for their feedback on these ideas. Thank you to the broader Ava Labs Platform Engineering Group for their feedback on this ACP prior to publication.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
