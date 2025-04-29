| ACP           | 186                                                 |
| :------------ | :-------------------------------------------------- |
| **Title**     | Update Validator Manager                            |
| **Author(s)** | [Martin Eckardt](https://github.com/martineckardt)  |
| **Status**    | Proposed                                            |
| **Track**     | Standards                                           |
| **Related**   | [ACP-77](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/77-reinventing-subnets/README.md)       |

## Abstract

This ACP proposes an extension to [ACP-77](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/77-reinventing-subnets/README.md)  by introducing a new transaction type `UpdateValidatorManagerTx` to facilitate validator manager migration and upgrading without requiring proxy contracts or network upgrades. This enhancement allows L1s to:

- Migrate validator management between different chains (e.g., from C-Chain to a native L1)
- Recover from misconfigurations where validator managers were set to inaccessible chains
- Avoid the need for proxy contracts and their associated security concerns
- Implement smooth upgrade paths without requiring network upgrades

By providing a flexible mechanism for validator manager migration, this proposal significantly enhances the sovereignty, security, and adaptability of Avalanche L1s.

## Motivation

ACP-77 introduced the concept of Avalanche Layer 1s (L1s) with the `ConvertSubnetToL1Tx` transaction that specifies a validator manager contract:

```golang
type ConvertSubnetToL1Tx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the Subnet to convert
    // Restrictions:
    // - Must not be the Primary Network ID
    Subnet ids.ID `json:"subnetID"`
    // BlockchainID where the validator manager lives
    ChainID ids.ID `json:"chainID"`
    // Address of the validator manager
    Address []byte `json:"address"`
    // Initial continuous-fee-paying validators for the L1
    Validators []L1Validator `json:"validators"`
    // Authorizes this conversion
    SubnetAuth verify.Verifiable `json:"subnetAuthorization"`
}
```

Currently, standard tools like AvaCloud, Avalanche CLI, and L1 Launcher typically recommend deploying proxy contracts via genesis to ensure the validator manager can be upgraded. This proxy-based approach introduces several challenges:

1. **Centralization of Power**: The proxy owner has extensive control over the validator set through the ability to change the validator manager implementation - essentially equivalent to a Proof-of-Authority admin even for Proof-of-Stake chains.

2. **Chain Migration Barriers**: The validator manager cannot move between chains. Projects like Beam that wish to start by managing their validator set on the C-Chain before eventually moving to their own L1 face a significant roadblock.

3. **No Recovery Path for Misconfiguration**: There is currently no recovery path for L1s that accidentally set their validator manager chainID to a chain they cannot control. While L1s can modify contract code on their own chain via a network upgrade, there is no recovery mechanism when the manager is set to an external chain. There have been instances of production L1s that nearly set their validator manager chainID to the P-Chain chainID, which would have rendered them permanently unrecoverable.

These limitations constrain L1 flexibility, create security risks, and impose unnecessary technical debt on projects building on Avalanche. A more flexible approach would significantly enhance the L1 ecosystem's resilience and adaptability.

## Specification

To address these challenges, this ACP proposes a new transaction type `UpdateValidatorManagerTx` that allows an L1 to update its validator manager without requiring proxy contracts or network upgrades.

### New P-Chain Transaction Type

#### `UpdateValidatorManagerTx`

The `UpdateValidatorManagerTx` will allow an L1 to update its validator manager contract `Address` and `ChainID`:

```golang
type UpdateValidatorManagerTx struct {
    // Metadata, inputs and outputs
    BaseTx
    // ID of the L1 to update the validator manager for
    // Restrictions:
    // - Must not be the Primary Network ID
    // - Must be an L1 (previously converted via ConvertSubnetToL1Tx)
    SubnetID ids.ID `json:"subnetID"`
    // New BlockchainID where the validator manager will live
    // Restrictions:
    // - Cannot be the P-Chain ChainID
    ChainID ids.ID `json:"chainID"`
    // New Address of the validator manager
    Address []byte `json:"address"`
    // A UpdateValidatorManagerMessage payload that authorizes this update
    Message warp.Message `json:"message"`
}
```

### P-Chain Warp Message Payloads

To support the validator manager update mechanism, two new Warp message types are introduced:

#### `UpdateValidatorManagerMessage`

The `UpdateValidatorManagerMessage` is used to authorize the update of an L1's validator manager. It must be signed by the current validator set of the L1 itself (not where the current validator manager is deployed).

The `UpdateValidatorManagerMessage` is specified as an `AddressedCall` with a payload of:

```golang
type UpdateValidatorManagerMessage struct {
    // Codec ID is the codec version used to serialize the payload
    CodecID uint16 `json:"codecID"`
    // TypeID is the payload type identifier (0x00000004 for this message)
    TypeID uint32 `json:"typeID"`
    // SubnetID identifies the L1 whose validator manager is being updated
    SubnetID ids.ID `json:"subnetID"`
    // NewChainID identifies the blockchain where the new validator manager will live
    NewChainID ids.ID `json:"newChainID"`
    // NewAddress is the address of the new validator manager
    NewAddress []byte `json:"newAddress"`
    // Expiry is the time at which this message becomes invalid
    Expiry uint64 `json:"expiry"`
}
```

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000004` for this message
- `subnetID` identifies the L1 whose validator manager is being updated
- `newChainID` and `newAddress` identify the new validator manager
- `expiry` is the time at which this message becomes invalid. As of a P-Chain timestamp `>= expiry`, this Avalanche Warp Message can no longer be used to update the validator manager

For the message to be considered valid, it must be signed by validators representing at least 67% of the L1's total stake weight.

#### `L1ValidatorManagerUpdatedMessage`

The P-Chain produces an `L1ValidatorManagerUpdatedMessage` when an L1's validator manager has been updated to confirm the change to the L1 and its validators.

The `L1ValidatorManagerUpdatedMessage` is specified as an `AddressedCall` with `sourceChainID` set to the P-Chain ID, the `sourceAddress` set to an empty byte array, and a payload of:

```golang
type L1ValidatorManagerUpdatedMessage struct {
    // Codec ID is the codec version used to serialize the payload
    CodecID uint16 `json:"codecID"`
    // TypeID is the payload type identifier (0x00000005 for this message)
    TypeID uint32 `json:"typeID"`
    // SubnetID identifies the L1 whose validator manager was updated
    SubnetID ids.ID `json:"subnetID"`
    // OldChainID identifies the blockchain where the old validator manager lived
    OldChainID ids.ID `json:"oldChainID"`
    // OldAddress is the address of the old validator manager
    OldAddress []byte `json:"oldAddress"`
    // NewChainID identifies the blockchain where the new validator manager lives
    NewChainID ids.ID `json:"newChainID"`
    // NewAddress is the address of the new validator manager
    NewAddress []byte `json:"newAddress"`
    // ValidatorList is the serialized list of validators at the time of update
    Validators []SubnetToL1ConversionValidatorData `json:"validators"`
}
```

- `codecID` is the codec version used to serialize the payload, and is hardcoded to `0x0000`
- `typeID` is the payload type identifier and is `0x00000005` for this message
- `subnetID` identifies the L1 whose validator manager was updated
- `oldChainID` and `oldAddress` identify the previous validator manager
- `newChainID` and `newAddress` identify the new validator manager
- `validators` contains the list of validators at the time of the update, using the same `SubnetToL1ConversionValidatorData` structure from the [SubnetToL1ConversionData](https://github.com/ava-labs/avalanchego/blob/d36f7d392bdae99a68ced5c4f6ef345ff3ce25da/vms/platformvm/warp/message/subnet_to_l1_conversion.go#L25C19-L25C52)

### Transaction Validation and Processing

When an `UpdateValidatorManagerTx` is submitted to the P-Chain, the following validation criteria apply:

1. The `subnetID` must refer to a valid L1 (previously converted via `ConvertSubnetToL1Tx`)
2. The `Message` must be a valid `UpdateValidatorManagerMessage` with:
   - The correct `subnetID` that matches the transaction
   - A sufficient signature threshold (≥67% of the L1's validator weight)
   - An `expiry` timestamp that has not passed
3. The `ChainID` and `Address` in the transaction must match the `NewChainID` and `NewAddress` in the message

If the transaction is valid, the P-Chain updates the validator manager information for the L1 and emits an `L1ValidatorManagerUpdatedMessage` to confirm the change.

### Implementation Considerations

The `UpdateValidatorManagerTx` is designed to be compatible with the existing L1 framework introduced in ACP-77. All existing validator operations (registration, weight updates, etc.) continue to work through the new validator manager after the update.

When migrating to a new validator manager, the L1 community must ensure:

1. The new validator manager is properly initialized with the current validator set state
2. The update process is well-coordinated to prevent disruption to validator operations
3. The old validator manager is properly decommissioned after the update

## Backwards Compatibility

This ACP introduces a new transaction type and warp message types that extend the functionality provided by ACP-77. It does not modify any existing behaviors and is fully backward compatible with the current L1 implementation.

Existing L1s that use proxy contracts can continue to do so, but new L1s can choose to use direct validator manager contracts and rely on this mechanism for future upgrades.

## Reference Implementation

A reference implementation will be developed and submitted as a pull request to AvalancheGo behind an appropriate upgrade flag. The implementation will follow the patterns established in ACP-77 for transaction processing and warp message handling.

## Security Considerations

### Authorization Threshold

The 67% threshold for authorizing validator manager updates matches the consensus threshold for the L1 and ensures that a supermajority of validators must agree to any change in the validator management system. This provides strong security against malicious attempts to take control of an L1.

### Timing Considerations

The `expiry` field in the `UpdateValidatorManagerMessage` ensures that update attempts cannot be replayed indefinitely. However, care must be taken to ensure that the expiry period is long enough to accommodate the coordination required for a smooth transition but not so long as to create an extended vulnerability window.

### Migration Risks

When migrating between validator managers or between chains, there is a risk of state inconsistency if the migration process is not carefully managed. L1 communities should:

1. Develop and test thorough migration plans
2. Consider implementing a temporary halt in validator set changes during the migration
3. Verify the state of the new validator manager before and after the update

### Recovery from Misconfiguration

This proposal provides a crucial safety mechanism for L1s that have accidentally configured their validator manager to point to a chain they cannot control. Without this mechanism, such a misconfiguration would be catastrophic and potentially unrecoverable, especially if the validator manager was set to a chain like the P-Chain where arbitrary contract deployment is not possible.

### Centralization Risks

While this proposal reduces the centralization risks associated with proxy owners, it introduces a different trust assumption: the L1 validators themselves must coordinate to authorize validator manager updates. This is arguably more aligned with the decentralized governance principles of an L1 but does require careful consideration of the social coordination aspects.

## Acknowledgements

This proposal builds upon the foundation laid by ACP-77 and was inspired by discussions within the Avalanche community about the challenges of validator manager upgrades and migrations.

Special thanks to the authors of ACP-77 for their groundbreaking work on Avalanche L1s, and to the projects that have shared their experiences and challenges with the current validator manager framework.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).