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

### Setting a Subnet Manager

To create a Subnet, a `CreateSubnetTx` must be issued on the P-Chain. This transaction includes an `Owner` field which defines the key that must be used to authorize any validator set additions (`AddSubnetValidatorTx`) or removals (`RemoveSubnetValidatorTx`).

To be a Permissionless Subnet, this `Owner` key must no longer have the ability to modify the Subnet's validator set. A `ConvertSubnetTx` must first be issued to explicitly convert a Subnet from Permissioned to Permissionless. This transaction will set the `(blockchainID, address)` pair that will manage the Subnet going forward. After `ConvertSubnetTx` is issued, the `Owner` from the `CreateSubnetTx` that created the Subnet will no longer have the ability to modify the Subnet's validator set.

To provide maximal flexibility for Permissionless Subnets, the BLS multisignature approach in Avalanche Warp Messaging (AWM) is re-used to approve modifications to the Subnet's validator set. Using the `(blockchainID, address)` pair defined in the `ConvertSubnetTx`, a Warp Message with an [`AddressedCall`](https://github.com/ava-labs/avalanchego/tree/master/vms/platformvm/warp/payload#addressedcall) payload can be constructed.

To validate an `AddressedCall` payload in Avalanche Warp Messaging, the `(blockchainID, address)` pair is used to lookup the validators of `blockchainID` and verify that the BLS multi-sig includes a quorum (set to 67%) of the Subnet's validator set. The P-Chain will use Warp Messages with `AddressedCall` payloads to support modifications of the Subnet's validator set using this `(blockchainID, address)` pair (using the `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` defined later in the specification).

#### ConvertSubnetTx

```golang
type ConvertSubnetTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the Subnet to transform
    // Restrictions:
    // - Must not be the Primary Network ID
    Subnet ids.ID `json:"subnetID"`
    // Chain where the Subnet manager lives
    BlockchainID ids.ID `json:"blockchainID"`
    // Address of the Subnet manager
    Address []byte `json:"address"`
    // Authorizes this conversion
    SubnetAuth verify.Verifiable `json:"subnetAuthorization"`
}
```

Once this transaction is accepted, `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` are disabled on the Subnet. `RegisterSubnetValidatorTx` and `SetSubnetValidatorWeightTx` must be used to manage the Subnet's validator set going forward.

### Recovering a Subnet

In the event that a Subnet has no validators, a valid BLS multi-signature cannot be produced. This situation can arise immediately after a Subnet is created or if all the Subnet's validators were removed from the validator set via `SetSubnetValidatorWeightTx` with `Weight = 0`.

A `RecoverSubnetTx` can be used in this situation to instantiate the Subnet's validator set using the `Owner` key defined in `CreateSubnetTx`. In all other situations, the `Owner` key is powerless after a `ConvertSubnetTx` is issued.

#### RecoverSubnetTx

```golang
type SubnetValidator struct {
    // Must be Ed25519 NodeID
    NodeID ids.NodeID `json:"nodeID"`
    // Weight of this validator used when sampling
    Weight uint64 `json:"weight"`
    // [Signer] is the BLS key for this validator.
    // Note: We do not enforce that the BLS key is unique across all validators.
    //       This means that validators can share a key if they so choose.
    //       However, a NodeID + Subnet does uniquely map to a BLS key
    Signer signer.Signer `json:"signer"`
    // Balance of the validator
    Balance uint64 `json:"balance"`
    // Leftover $AVAX from the [Balance] will be issued to this
    // owner once it is removed from the validator set.
    ChangeOwner fx.Owner `json:"changeOwner"`
}

type RecoverSubnetTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Validator set for the Subnet
    Validators []SubnetValidator `json:"validators"`
    // Auth that will be allowing these validators into the network
    SubnetAuth verify.Verifiable `json:"subnetAuthorization"`
}
```

### Adding Subnet Validators

#### RegisterSubnetValidatorTx

A `RegisterSubnetValidatorTx` can be used to add a Subnet Validator.

```golang
type RegisterSubnetValidatorTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // Balance must be equal to the balance uint64 in the [Message].
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
    //   - Expiry
    //   - Ed25519 Signature over "[subnetID]+[nodeID]+[blsPublicKey]+[weight]+[balance]+[expiry]"
    Message warp.Message `json:"message"`
}
```

The `Message` field must be an `AddressedCall` with the payload:

```text
+-----------+----------+-----------+
|   codecID :   uint16 |   2 bytes |
+-----------+----------+-----------+
|    typeID :   uint32 |   4 bytes |
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
                       | 150 bytes |
                       +-----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000000` for this transaction
- `subnetID`, `nodeID`, and `weight` are for the Subnet Validator being added
- `expiry` is the time after which this message is invalid. After the P-Chain timestamp is past `expiry`, this Avalanche Warp Message can no longer be used to add the `nodeID` to the validator set of `subnetID`.

    `validationID` is defined as the SHA256 hash of the `Payload` of the `AddressedCall`. This SHA256 hash will be used for replay protection. Used `validationID`s will be stored on the P-Chain. If a `RegisterSubnetValidatorTx`'s `validationID` has already been used, the transaction will be considered invalid. To prevent storing an unbounded number of `validationID`s, the `expiry` is required to be no longer than 48 hours in the future of the time the transaction is issued on the P-Chain. Any `validationIDs` with `expiry` more than 48 hours in the past can be flushed from the P-Chain's state.
- `signature` is the raw bytes of the Ed25519 signature over the concatenated bytes of `[subnetID]+[nodeID]+[blsPublicKey]+[weight]+[balance]+[expiry]`. This signature must correspond to the Ed25519 public key that is used for the `nodeID`. This approach prevents NodeIDs from being unwillingly added to Subnets. `balance` is the minimum initial $nAVAX balance that must be attached to the validator serialized as a uint64. A Subnet may choose to validate that the `signature` field is well-formed but it is not required.

Subnets are responsible for defining the procedure on how to retrieve the above information from prospective validators.

An EVM Subnet may choose to implement this step like so:

- Use the number of tokens the user has staked into a smart contract on the Subnet to determine the weight of their validator
- Require the user to submit an on-chain transaction with their validator information
- Generate the warp message

After the `RegisterSubnetValidatorTx` is accepted on the P-Chain, the Subnet Validator is added to the Subnet's validator set. A `minNonce` field (initially set to `0`) corresponding to the `validationID` will be stored on addition to the validator set. This field will be used when validating the `SetSubnetValidatorWeightTx` defined below.

When any validator is removed from the set (whether forcefully or per the validator's request), the P-Chain will also send a warp message to the Subnet notifying it of the validator set removal. It is up to the Subnet on how to handle such a message, especially if unexpected. A validator's stake could continue to remain locked for an extended period of time after this point, for example.

This transaction is, by design, not required to be submitted by the validator themselves. With the Ed25519 signature, the validator guarantees that they can only be added to the validator set if `Signer` corresponds to the `blsPublicKey` and `sum($AVAX inputs) - sum($AVAX outputs) - TxFee` >= `balance`. The `RegisterSubnetValidatorTx` is considered invalid if those two properties are not satisfied.

For a `RegisterSubnetValidatorTx` to be valid, `sum($AVAX inputs) - sum($AVAX outputs) - TxFee` must be >= the greater of 5 $AVAX or two weeks of the current fee. This prevents Subnet Validators from being added with too low of an initial balance where they become immediately delinquent based on the continous fee mechanism defined below. A Subnet Validator can leave at any time before the initial $AVAX is consumed and claim the remaining balance to the `ChangeOwner` defined in the transaction.

Note: There is no `EndTime` specified in this transaction. Subnet Validators are only removed when a `SetSubnetValidatorWeightTx` sets a validator's weight to `0` or `ExitValidatorSetTx` is issued.

### Modifying Subnet Validators

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
    // AddressedCall with Payload:
    //   - MessageID (SHA256 of the AddressedCall Payload of the RegisterSubnetValidatorTx adding the validator)
    //   - Weight
    //   - Nonce
    Message warp.Message `json:"message"`
}
```

The `Message` field must be an `AddressedCall` with the payload:

```text
+--------------+----------+----------+
|      codecID :   uint16 |  2 bytes |
+--------------+----------+----------+
|       typeID :   uint32 |  4 bytes |
+-----------+----------+----------+
| validationID : [32]byte | 32 bytes |
+--------------+----------+----------+
|       weight :   uint64 |  8 bytes |
+--------------+----------+----------+
|        nonce :   uint64 |  8 bytes |
+--------------+----------+----------+
                          | 54 bytes |
                          +----------+
```

- `codecID` is the codec version used to serialize the payload and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000001` for this transaction
- `validationID` is the SHA256 of the `Payload` of the `AddressedCall` in the `RegisterSubnetValidatorTx` adding the validator to the Subnet's validator set
- `nonce` is a strictly increasing number that denotes the latest validator weight update and provides replay protection for this transaction

    The P-Chain state will store a `minNonce` associated with the `validationID`. When accepting the `RegisterSubnetValidatorTx`, the `minNonce` will be set to `0` for `validationID`. `nonce` must satisfy `nonce >= minNonce` for the `SetSubnetValidatorWeightTx` to be valid. Note that `nonce` is not required to be incremented by `1` with each successive validator weight update. If `minNonce` is `MaxUint64`, the `weight` in the `SetSubnetValidatorWeightTx` is required to be `0` to prevent Subnets from being unable to remove `nodeID` in a subsequent `SetSubnetValidatorWeightTx`. When a Subnet Validator is removed from the active validator set (`weight == 0`), the `minNonce` and `validationID` will be removed from the P-Chain state. This state can be reaped during validator exit since `validationID` can never be re-initialized as a result of the replay protection provided by `expiry` in `RegisterSubnetValidatorTx`. `minNonce` will be set to `nonce + 1` when `SetSubnetValidatorWeightTx` is executed and `weight != 0`

### Removing Subnet Validators

#### SetSubnetValidatorWeightTx (Weight = 0)

Issuing a `SetSubnetValidatorWeightTx` with `Weight` of `0` will remove the Subnet Validator from the Subnet's Validator set.

All state related to the Subnet Validator being removed will be removed from the P-Chain's active state (BLS key, NodeID etc). This validator can be newly added to the Subnet's validator set using the `RegisterSubnetValidatorTx` flow.

#### ExitValidatorSetTx

Subnet Validators can use `ExitValidatorSetTx` to exit the validator set without interacting with the Subnet. An Ed25519 Signature is required using the Subnet Validator's Ed25519 private key for this transaction to be valid. Remaining $AVAX in the Subnet Validator's `Balance` will be issued to the `ChangeOwner` defined when adding this validator to the validator set.

This transaction can be issued on the P-Chain without explicit consent of the Subnet. It is expected that validators should be removed from the Subnet's validator set through a `SetSubnetValidatorWeightTx` with weight `0` by initiating Subnet Validator removal on the Subnet. However, the ability to exit a Subnet Validator set is critical for censorship-resistance and/or failed Subnets. If a validator ever wishes to stop participating in Subnet consensus, they will be able to do so through this transaction. This is enforced on the P-Chain to prevent Subnets from locking Validators into participating in consensus indefinitely. Note that this does not modify a Subnet's total staking weight, this transaction moves the Validator from Active to Inactive.

After issuing this transaction, this validator can no longer be re-added to the active Subnet Validator set via `IncreaseBalanceTx`. The only transaction that be used for this validator is `SetSubnetValidatorWeightTx` with weight `0`.

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

### Managing Subnet Validator Balance

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

### Sidebar: Subnet Sovereignty

After this ACP is activated, the P-Chain will no longer support staking of any assets other than $AVAX for the Primary Network. The P-Chain will no longer support distribution of staking rewards for Subnets. All staking-related operations for Subnet Validation must be managed by the Subnet on the Subnet. The P-Chain simply requires a continuous fee per Subnet Validator. If a Subnet would like to manage their Validator's balances on the P-Chain, it can cover the cost for all Subnet Validators by posting the $AVAX balance on the P-Chain. Subnets can implement any mechanism they want to pay the continuous fee charged by the P-Chain for its participants.

By moving ownership of the Subnet's validator set from the P-Chain to the Subnet, Subnet creators have no restrictions on what requirements they have to join their Subnet as a validator. Any stake that is required to join the Subnet Validator set is locked on the Subnet. If a validator is removed from the Subnet Validator set via a `SetSubnetValidatorWeightTx` with weight `0` or an `ExitValidatorSetTx` on the P-Chain, the stake on the Subnet will continue to be locked. How each Subnet handles stake associated with the Subnet Validator is entirely left up to the Subnet and can be treated independently to what happens on the P-Chain.

This new relationship between the P-Chain and Subnets provides a dynamic where Subnets can use the P-Chain as an impartial judge to modify parameters (in addition to its existing role of helping to validate incoming Avalanche Warp Messages). If a Validator is misbehaving, the Subnet Validators can collectively generate a BLS multisig to reduce the voting weight of a misbehaving validator. This operation is fully secured by the Avalanche Primary Network (225M AVAX or $8.325B at the time of writing).

Follow-up ACPs could extend the P-Chain <> Subnet relationship to include parametrization of the 67% threshold to enable Subnets to choose a different threshold based on their security model (e.g. a simple majority of 51%).

### Continuous Fee Mechanism

Every additional Subnet Validator on the P-Chain adds persistent load to the Avalanche Network. When a validator transaction is issued on the P-Chain, it is charged for the computational cost of the transaction itself but is not charged for the cost of an active Subnet Validator over the time they are validating on the network (which may be indefinitely). This is a common problem in blockchains, spawning many state rent proposals in the broader blockchain space to address it. The following fee mechanism takes advantage of the fact that each Subnet Validator uses the same amount of computation and charges each Subnet Validator the dynamic base fee for every discrete unit of time it is active.

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

#### Fee Algorithm

[ACP-103](../103-dynamic-fees/README.md) proposes a dynamic fee mechanism for transactions on the P-Chain. This mechanism is repurposed with minor modifications for the active Subnet Validator continuous fee.

At activation, the number of excess active Subnet Validators $x$ is set to `0`.

The fee rate per second for an active Subnet Validator is:

$$M \cdot \exp\left(\frac{x}{K}\right)$$

Where:

- $M$ is the minimum price for an active Subnet Validator
- $\exp\left(x\right)$ is an approximation of $e^x$ following the EIP-4844 specification

  ```python
  # Approximates factor * e ** (numerator / denominator) using Taylor expansion
  def fake_exponential(factor: int, numerator: int, denominator: int) -> int:
    i = 1
    output = 0
    numerator_accum = factor * denominator
    while numerator_accum > 0:
        output += numerator_accum
        numerator_accum = (numerator_accum * numerator) // (denominator * i)
        i += 1
    return output // denominator
  ```

- $K$ is a constant to control the rate of change for the Subnet Validator price

After every second, $x$ will be updated:

$$x = \max(x + (V - T), 0)$$

Where:

- $V$ is the number of active Subnet Validators
- $T$ is the target number of active Subnet Validators

Whenever $x$ increases by $K$, the price per active Subnet Validator increases by a factor of `~2.7`. If the price per active Subnet Validator gets too expensive, some active Subnet Validators will exit the active validator set, decreasing $x$, dropping the price. The price per active Subnet Validator constantly adjusts to make sure that, on average, the P-Chain has no more than $T$ active Subnet Validators.

#### Block Timestamp Validity Change

To easily track $x$ between blocks, the difference between $V$ and $T$ must be constant in the $\Delta t$ between blocks. $x$ can then simply be updated to $\max(x + \Delta t \cdot (V - T), 0)$ after each valid block. If $V-T$ is not constant between blocks, this optimization cannot be made.

Suppose the network currently has `11000` Subnet Validators. With $T$ being `10000`, $x$ will increase by $V-T$ or `1000` every second. If the next block's timestamp is `5` seconds after the current block's timestamp. $x$ will increase by `5000`. This is expected behavior _unless_ Subnet Validators should have been removed within those `5` seconds. If `100` Subnet Validators would exhaust their balance `1` second after the current block timestamp, $x$ should only increase by `900` every second after the first second elapses. $x$ should have increased by `1000` + `4 * 900` = `4500`, not `5000`.

To ensure that the correct fee is always charged per unit of time, blocks are considered valid if their timestamps are no greater than the time at which the first Subnet Validator gets removed from a lack of funds. This upholds the invariant that the number of Subnet Validators remains constant between blocks.

The block building protocol is modified to account for this change by first checking if the wall clock time removes any Subnet Validator due to a lack of funds. If the wall clock time does not remove any Subnet Validators, the wall clock time is used to build the block. If it does, a binary search may be performed between the parent block's timestamp and the wall clock time to find the time at which the first Subnet Validator gets removed.

#### Fee Calculation

Since $V-T$ is guaranteed to be constant between blocks, a simple formula can be derived. Prior to processing the next block, the total Subnet Validator fee assessed in the $\Delta t$ between the current block and the next block is::

$$\sum_{n=1}^{\Delta t} M \cdot \exp \left(\frac{x+n\cdot(V-T)}{K}\right)$$

However, this formula does not hold when $V-T < 0$ and $x < \Delta t \cdot (V-T)$. This is because the $x$ is nonnegative according to its update function $x = \max(x + \Delta t \cdot (V - T), 0)$. The above formula can be used during the time period where $x$ is decreasing by $T-V$ and is not floored to $0$:

$$\min\left(\Delta t, \left\lfloor{\frac{x}{T-V}}\right\rfloor\right)$$

For the remaining time period where $x$ would be floored to $0$, the fee is:

$$M \cdot \max\left(\Delta t - \left\lfloor{\frac{x}{T-V}}\right\rfloor, 0\right)$$

After the Subnet Validator fee is calculated, $x$ can be updated:

$$x = \max(x + \Delta t \cdot (V - T), 0)$$

#### Parameters

The parameters at activation are:

| Parameter | Value |
| - | - |
| $T$ | `10000` Subnet Validators |
| $C$ | `20000` Subnet Validators |
| $M$ | `2048` nAVAX/s (`0.1769472` AVAX/day) |
| $K$ | `60_480_000_000` |

These parameters air on the conservative side as a precautionary measure to ensure network stability during activation. `60_480_000_000` was chosen for $K$ to have the validator fee increase by a factor of `~2.7` after a full week of being at $C$ or `20000` Subnet Validators. At activation, this should strike a good balance between preventing squatting and spikiness of the continuous fee.

A future ACP can adjust the parameters to increase $T$, reduce $M$, and/or modify $K$.

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
  - `RecoverSubnetTx`
  - `RegisterSubnetValidatorTx`
  - `SetSubnetValidatorWeightTx`
  - `ExitValidatorSetTx`
  - `IncreaseBalanceTx`

Once a Subnet issues a `ConvertSubnetTx`, `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` can no longer be used to modify that Subnet's validator set. A Subnet Validator added with an `AddSubnetValidatorTx` will continue to validate the Subnet until their `EndTime` is reached. After expiry, those validators (if they want to continue validating) must use the `RegisterSubnetValidatorTx` flow outlined in this ACP to register as a Subnet Validator.

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
