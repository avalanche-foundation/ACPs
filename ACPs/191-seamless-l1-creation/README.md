| ACP | 191 |
| :- | :- |
| **Title** | Seamless L1 Creations (CreateL1Tx) |
| **Author(s)** | Martin Eckardt ([@martineckardt](https://github.com/martineckardt)), Aaron Buchwald ([@aaronbuchwald](https://github.com/aaronbuchwald)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)), Meaghan FitzGerald ([@meaghanfitzgerald](https://github.com/meaghanfitzgerald)) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This ACP introduces a new P-Chain transaction type called `CreateL1Tx` that simplifies the creation of Avalanche L1s. It consolidates three existing transaction types (`CreateSubnetTx`, `CreateChainTx`, and `ConvertSubnetToL1Tx`) into a single atomic operation. This streamlines the L1 creation process, removes the need for the intermediary Subnet creation step, and eliminates the management of temporary `SubnetAuth` credentials.

## Motivation

[ACP-77](../77-reinventing-subnets/README.md) introduced Avalanche L1s, providing greater sovereignty and flexibility compared to Subnets. However, creating an L1 currently requires a three-step process:

1. `CreateSubnetTx`: Create the Subnet record on the P-Chain and specify the `SubnetAuth`
2. `CreateChainTx`: Add a blockchain to the Subnet (can be called multiple times)  
3. `ConvertSubnetToL1Tx`: Convert the Subnet to an L1, specifying the initial validator set and the validator manager location

This process has several drawbacks:

* It requires orchestrating three separate transactions that could be handled in one.
* The `SubnetAuth` must be managed during creation but becomes irrelevant after conversion.
* The multi-step process increases complexity and potential for errors.
* It introduces unnecessary state transitions and storage overhead on the P-Chain.

By introducing a single `CreateL1Tx` transaction, we can simplify the process, reduce overhead, and improve the developer experience for creating L1s.

## Specification

### New Transaction Type

The following new transaction type is introduced:

```go
// ChainConfig represents the configuration for a chain to be created
type ChainConfig struct {
    // A human readable name for the chain; need not be unique
    ChainName string `serialize:"true" json:"chainName"`
    // ID of the VM running on the chain
    VMID ids.ID `serialize:"true" json:"vmID"`
    // IDs of the feature extensions running on the chain
    FxIDs []ids.ID `serialize:"true" json:"fxIDs"`
    // Byte representation of genesis state of the chain
    GenesisData []byte `serialize:"true" json:"genesisData"`
}

// CreateL1Tx is an unsigned transaction to create a new L1 with one or more chains
type CreateL1Tx struct {
    // Metadata, inputs and outputs
    BaseTx `serialize:"true"`
    
    // Chain configurations for the L1 (can be multiple)
    Chains []ChainConfig `serialize:"true" json:"chains"`
    
    // Chain where the L1 validator manager lives
    ManagerChainID ids.ID `serialize:"true" json:"managerChainID"`
    
    // Address of the L1 validator manager
    ManagerAddress types.JSONByteSlice `serialize:"true" json:"managerAddress"`
    
    // Initial pay-as-you-go validators for the L1
    Validators []*L1Validator `serialize:"true" json:"validators"`
}
```

The `L1Validator` structure follows the same definition as in [ACP-77](../77-reinventing-subnets/README.md#convertsubnettol1tx).

### Transaction Processing

When a `CreateL1Tx` transaction is processed, the P-Chain performs the following operations atomically:

1. Create a new L1.
2. Create chain records for each chain configuration in the `Chains` array.  
3. Set up the L1 validator manager with the specified `ManagerChainID` and `ManagerAddress`.  
4. Register the initial validators specified in the `Validators` array.

### IDs

* `subnetID`: The `subnetID` of the L1 is the transaction hash.
* `blockchainID`: the `blockchainID` for each blockchain is is defined as the SHA256 hash of the 37 bytes resulting from concatenating the 32 byte `subnetID` with the `0x00` byte and the 4 byte `chainIndex` (index in the `Chains` array within the transaction)
* `validationID`: The `validationID` for the initial validators added through `CreateL1Tx` is defined as the SHA256 hash of the 36 bytes resulting from concatenating the 32 byte `subnetID` with the 4 byte `validatorIndex` (index in the `Validators` array within the transaction).

Note: Even with this updated definition of the `blockchainID`s for chains created using this new flow, the `validationID`s of the L1s initial set of validators is still compatible with the existing reference validator manager contracts as defined [here](https://github.com/ava-labs/icm-contracts/blob/4a897ba913958def3f09504338a1b9cd48fe5b2d/contracts/validator-manager/ValidatorManager.sol#L247).

### Restrictions and Validation

The `CreateL1Tx` transaction has the following restrictions and validation criteria:

1. The `Chains` array must contain at least one chain configuration  
2. The `ManagerChainID` must be a valid blockchain ID, but cannot be the P-Chain blockchain ID 
3. Validator nodes must have unique NodeIDs within the transaction  
4. Each validator must have a non-zero weight and a non-zero balance  
5. The transaction inputs must provide sufficient AVAX to cover the transaction fee and all validator balances

### Warp Message

After the transaction is accepted, the P-Chain must be willing to sign a `SubnetToL1ConversionMessage` with a `conversionID` corresponding to the new L1, similar to what would happen after a `ConvertSubnetToL1Tx`. This ensures compatibility with existing systems that expect this message, such as the validator manager contracts.

## Backwards Compatibility

This ACP introduces a new transaction type and does not modify the behavior of existing transaction types. Existing Subnets and L1s created through the three-step process will continue to function as before. This change is purely additive and does not require any changes to existing L1s or Subnets.

The existing transactions `CreateSubnetTx`, `CreateChainTx` and `ConvertSubnetToL1Tx` remain unchanged for now, but may be removed in a future ACP to ensure systems have sufficient time to update to the new process.

## Reference Implementation

A reference implementation must be provided in order for this ACP to be considered implementable.

## Security Considerations

The `CreateL1Tx` transaction follows the same security model as the existing three-step process. By making the L1 creation atomic, it reduces the risk of partial state transitions that could occur if one of the transactions in the three-step process fails.

The same continuous fee mechanism introduced in ACP-77 applies to L1s created through this new transaction type, ensuring proper metering of validator resources.

The transaction verification process must ensure that all validator properties are properly validated, including unique NodeIDs, valid BLS signatures, and sufficient balances.

## Rationale and Alternatives

The primary alternative is to maintain the status quo \- requiring three separate transactions to create an L1. However, this approach has clear disadvantages in terms of complexity, transaction overhead, and user experience.

Another alternative would be to modify the existing `ConvertSubnetToL1Tx` to allow specifying chain configurations directly. However, this would complicate the conversion process for existing Subnets and would not fully address the desire to eliminate the Subnet intermediary step for new L1 creation.

The chosen approach of introducing a new transaction type provides a clean solution that addresses all identified issues while maintaining backward compatibility.

## Acknowledgements

The idea for this PR was originally formulated by Aaron Buchwald in our discussion about the creation of L1s. Special thanks to the authors of ACP-77 for their groundbreaking work on Avalanche L1s, and to the projects that have shared their experiences and challenges with the current validator manager framework.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
