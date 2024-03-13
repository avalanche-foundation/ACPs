```text
ACP: 100
Title: Warp Addressed Transactions
Author(s): Aaron Buchwald <https://github.com/aaronbuchwald>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/68
Status: Implementable
Track: Standards
```

## Abstract

Add a standard for using Warp `AddressedCall` payloads as an account capable of signing transactions. Warp Addressed Transactions will define a standard that can be implemented on any VM throughout the Avalanche Network. This ACP will define the general standard that can be applied to any VM and the first instance of using it on the P-Chain.

## Motivation


Subnets are currently created and orchestrated entirely on the Avalanche P-Chain. The original motivation for Warp Addressed Transactions is to provide Subnets the ability to completely define their own validator set management from the location of their choice. Warp Addressed Transactions enable Subnet Ownership to be transferred to a (Subnet, Account) pair, which can be a smart contract deployed to the same Subnet or an account on the C-Chain if desired.

By transferring ownership to a contract or program on another chain, it provides Subnets the ability to orchestrate their own validator set against a generic API.

In the immediate term, this accomplishes two critical goals for Subnets:

1) Enable Token Standard PoS Subnets (ERC-20 Staking)
2) Enable fully generic validator set management

This is a significant improvement over the existing P-Chain standard, which does not offer ERC-20 Staking, programmable PoS Subnets, and restricts PoS Subnets to a single parameterizable reward curve. The parameterizable reward curve has the additional drawback that it requires locking all Subnet staking rewards for the lifetime of the Subnet at the creation of the Subnet. This forces Subnet developers to make a one-way decision about the tokenomics of their project at the very start. Although it's normal for PoS networks to commit to their tokenomics, many projects have found it necessary to make adjustments throughout their lifetime. This ACP ensures that Subnets have complete and generic control of their own tokenomics and should never depend on future changes to the P-Chain.

Further, this standard could be implemented by any VM that wants to enable Warp cross-chain accounts.

## Specification

Warp Addressed Transactions utilize an alternative signature/credential scheme, which we'll denote a `WarpAddressedSignature`.

A `WarpAddressedSignature` includes a valid Warp Signature, networkID, sourceChainID, sourceAddress, and the txHash of the transaction it's authorizing. The transaction itself is assumed to contain replay protection, so that a `destinationChainID` is unnecessary.

```go
Message{
	UnsignedMessage: UnsignedMessage{
		NetworkID: networkID,
		SourceChainID: sourceChainID,
		Payload: AddressedCall{
			SourceAddress: sourceAddress,
			Payload: txHashBytes,
		},
	},
	Signature: warpSignature,
}
```

TODO: re-define this in a python spec
Verification will perform the following checks:

1. Verify networkID
2. Verify TxHash matches the attached Tx payload
3. Verify Address(sourceChainID, SourceAddress) == expectedAddress
4. Verify Warp signature against ProposerVM wrapper's P-Chain height


## Backwards Compatibility

Previous signature schemes will remain unaffected by these changes. This proposal adds an alternative signature scheme, which requires a coordinated network upgrade to activate on the P-Chain.

Activating Warp Addressed Transactions on another chain requires a separate coordinated network upgrade.

## Reference Implementation

Here we provide a reference implementation introducing `WarpAddressedSignatures` to the P-Chain's PlatformVM in the form of a `WarpAddressedCredential`. `WarpAddressedCredential` is a direct replacement for the only currently supported signature scheme credential: [secp256k1fx.Credential](https://github.com/ava-labs/avalanchego/blob/ddf66eaed1659c2caa8531bbda83b360ca85900e/vms/secp256k1fx/credential.go#L17).

This credential type is used on the P-Chain through the PlatformVM's [Fx interface](https://github.com/ava-labs/avalanchego/blob/master/vms/platformvm/fx/fx.go).

Each of these secp256k1fx verification checks performs basic transaction validity checks, type checks on the credential type, and eventually passes through to call [VerifyCredentials](https://github.com/ava-labs/avalanchego/blob/ddf66eaed1659c2caa8531bbda83b360ca85900e/vms/secp256k1fx/fx.go#L180).

This reference implementation assumes that the type checks are changed to allow for an alternative credential type, the remaining transaction validity checks are kept identical, and that it only needs to modify the credential verification.

The credential verification is defined on a new type `WarpAddressedCredential` to authorize the use of any inputs that specify a single address matching the `WarpAddressedCredential`. We assume that deriving an address from a standard `WarpAddressedCredential` is specific to the VM implemenation, and define that address on the P-Chain to be given by ripemd160(sha256(sourceChainID, sourceAddress)). This is chosen to resemble the address derivation from a secp256k1 public key and to maintain the same 20 byte address format currently used on the P-Chain to minimize the necessary changes.

```go
// Copyright (C) 2019-2024, Ava Labs, Inc. All rights reserved.
// See the file LICENSE for licensing terms.

package fx

import (
	"bytes"
	"context"
	"fmt"

	"github.com/ava-labs/avalanchego/snow/validators"
	"github.com/ava-labs/avalanchego/utils/hashing"
	"github.com/ava-labs/avalanchego/utils/timer/mockable"
	"github.com/ava-labs/avalanchego/vms/platformvm/warp"
	"github.com/ava-labs/avalanchego/vms/platformvm/warp/payload"
	"github.com/ava-labs/avalanchego/vms/secp256k1fx"
)

type Fx struct {
	networkID    uint32
	clock        mockable.Clock
	bootstrapped bool

	pChainState          validators.State
	quorumNum, quorumDen uint64
}

// VerifyCredentials ensures that the output can be spent by the input with the
// credential. A nil return values means the output can be spent.
func (fx *Fx) VerifyCredentials(utx secp256k1fx.UnsignedTx, in *secp256k1fx.Input, warpAddressedCredential *warp.Message, out *secp256k1fx.OutputOwners, pChainHeight uint64) error {
	numSigs := len(in.SigIndices)
	switch {
	case fx.networkID != warpAddressedCredential.NetworkID:
		return fmt.Errorf("incorrect networkID %d for WarpAddressedCall with networkID %d", warpAddressedCredential.NetworkID, fx.networkID)
	case out.Locktime > fx.clock.Unix():
		return secp256k1fx.ErrTimelocked
	case len(out.Addrs) != 1:
		return fmt.Errorf("incorrect number of output addresses for WarpAddressedCall: %d", len(out.Addrs))
	case out.Threshold != 1:
		return fmt.Errorf("incorrect threshold for WarpAddressedCall: %d", out.Threshold)
	case numSigs != 1:
		return fmt.Errorf("incorrect number of signatures for WarpAddressedCall: %d", numSigs)
	case !fx.bootstrapped: // disable signature verification during bootstrapping
		return nil
	}

	txHash := hashing.ComputeHash256(utx.Bytes())
	addressedCall, err := payload.ParseAddressedCall(warpAddressedCredential.Payload)
	if err != nil {
		return err
	}

	// Verify that the addressedCall specifies the correct txHash
	if !bytes.Equal(addressedCall.Payload, txHash) {
		return fmt.Errorf("unexpected txHash: %x, expected: %x", addressedCall.Payload, txHash)
	}

	// Calculate the corresponding address of the credential by taking a hash of the
	// concatenated fields: sourceChainID | sourceAddress
	credentialInputSlice := make([]byte, 32+len(addressedCall.SourceAddress))
	copy(credentialInputSlice[0:32], warpAddressedCredential.SourceChainID[:])
	copy(credentialInputSlice[32:], addressedCall.SourceAddress)
	credentialAddress := hashing.ComputeHash160Array(credentialInputSlice)
	if credentialAddress != out.Addrs[0] {
		return fmt.Errorf("expected signature from %s but got from (%s, %s) = %s",
			out.Addrs[0],
			warpAddressedCredential.SourceChainID,
			addressedCall.SourceAddress,
			credentialAddress,
		)
	}

	// Verify the Warp Signature
	warpAddressedCredential.Signature.Verify(
		context.Background(),
		&warpAddressedCredential.UnsignedMessage,
		fx.networkID,
		fx.pChainState,
		pChainHeight,
		fx.quorumNum,
		fx.quorumDen,
	)

	return nil
}
```

## Security Considerations

Warp Addressed Transactions assume that there will never be a hash collision between the 20-byte address generated from a `WarpAddressedCredential` and a secp256k1fx public key derived address.

This change enables a Warp `AddressedCall` payload to authorize an action on the P-Chain. If an existing VM or Subnet has already started using Warp, it's possible that validators have already begun signing valid `AddressedCall` payloads. This warrants calling out, but does not represent a security concern.

P-Chain UTXOs including funds or Subnet Control must be transferred to a Warp Address before an already signed `AddressedCall` payload can authorize any meaningful action on the P-Chain. This means users only need to be aware that they should only transfer P-Chain UTXOs to a Warp Address if the program at that address was designed to correctly handle Warp Addressed Transactions.

## Introduced Terminology

Warp Transaction Signature - a Warp signature used as the signature scheme for an arbitrary transaction payload
Warp Addressed Transaction - an arbitrary transaction using a Warp Transaction Signature as its signature scheme
Warp Derived Address - the address derived from a Warp Transaction Signature for which the signature can authorize actions

## Open Questions

How can Warp Addressed Transactions perform UTXO set management on the P-Chain?

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Acknowledgements

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
