```text
ACP: 31
Title: Enable Subnet Ownership Transfer
Author(s): Dhruba Basu <https://github.com/dhrubabasu>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/35
Status: Activated
Track: Standards
```

## Abstract

Allow the current owner of a Subnet to transfer ownership to a new owner.

## Motivation

Once a Subnet is created on the P-chain through a [CreateSubnetTx](https://github.com/ava-labs/avalanchego/blob/v1.10.15/vms/platformvm/txs/create_subnet_tx.go#L14-L19), the `Owner` of the subnet is currently immutable. Subnet operators may want to transition ownership of the Subnet to a new owner for a number of reasons, not least of all being rotating their control key(s) periodically.

## Specification

Implement a new transaction type (`TransferSubnetOwnershipTx`) that:
1. Takes in a `Subnet`
2. Verifies that the `SubnetAuth` has the right to remove the node from the subnet by verifying it against the `Owner` field in the `CreateSubnetTx` that created the `Subnet`.
3. Takes in a new `Owner` and assigning it as the new owner of `Subnet`

This transaction type should have the following format (code below is presented in Golang):

```golang
type TransferSubnetOwnershipTx struct {
	// Metadata, inputs and outputs
	BaseTx `serialize:"true"`
	// ID of the subnet this tx is modifying
	Subnet ids.ID `serialize:"true" json:"subnetID"`
	// Proves that the issuer has the right to remove the node from the subnet.
	SubnetAuth verify.Verifiable `serialize:"true" json:"subnetAuthorization"`
	// Who is now authorized to manage this subnet
	Owner fx.Owner `serialize:"true" json:"newOwner"`
}
```

This transaction type should have type ID `0x21` in codec version `0x00`.

This transaction type should have a fee of `0.001 AVAX`, equivalent to adding a subnet validator/delegator.

## Backwards Compatibility

Adding a new transaction type is an execution change and requires a mandatory upgrade for activation. Implementors must take care to reject this transaction prior to activation. This ACP only details the specification of the `TransferSubnetOwnershipTx` type.

## Reference Implementation

An implementation of `TransferSubnetOwnershipTx` was created [here](https://github.com/ava-labs/avalanchego/pull/2178) and subsequently merged into AvalancheGo. Since the "D" Upgrade is not activated, this transaction will be rejected by AvalancheGo.

If modifications are made to the specification of the transaction as part of the ACP process, the code must be updated prior to activation.

## Security Considerations

No security considerations.

## Open Questions

No open questions.

## Acknowledgements

Thank you [@friskyfoxdk](https://github.com/friskyfoxdk) for filing an [issue](https://github.com/ava-labs/avalanchego/issues/1946) requesting this feature. Thanks to [@StephenButtolph](https://github.com/StephenButtolph) and [@abi87](https://github.com/abi87) for their feedback on the reference implementation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).