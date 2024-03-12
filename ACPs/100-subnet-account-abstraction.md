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

Warp Addresed Transactions would require a new credential type on the P-Chain, which we'll denote as a `WarpAddressedCredential`. The P-Chain currently only uses the [secp256k1fx.Credential](https://github.com/ava-labs/avalanchego/blob/ddf66eaed1659c2caa8531bbda83b360ca85900e/vms/secp256k1fx/credential.go#L17). This credential type is used to authorize all actions on the P-Chain through the following function calls:

1. VerifySpend (passes through to VerifySpendUTXOs)
2. VerifySpendUTXOs (passes through to VerifyTransfer on vm.fx on each UTXO)
3. VerifyPermission (subnet tx verification)

Each of these pass eventually perform a type check to ensure that the credential is the expected `secp256k1fx.Credential` type and then call [VerifyCredentials](https://github.com/ava-labs/avalanchego/blob/ddf66eaed1659c2caa8531bbda83b360ca85900e/vms/secp256k1fx/fx.go#L180) to verify the credential matches the authorization on the UTXO itself.

Warp Addressed Transactions will create an alternative implementation of the same Fx interface the P-Chain currently uses: https://github.com/ava-labs/avalanchego/blob/ddf66eaed1659c2caa8531bbda83b360ca85900e/vms/platformvm/fx/fx.go#L20. The new implementation will replace the Credential type check with a switch to determine if the credential should be verified as a `secp256k1.Credential` (verification is identical) or a `WarpAddressedCredential`.

A reference implementation is provided below for a full example specification.

The verification conditions are modified to handle the assumption that a `WarpAddressedCredential` only authorizes the use of outputs that are controlled by a single address.

The Warp Address


The sample implementation uses the ripemd160(sha256(sourceChainID | sourceAddress)) as the corresponding P-Chain address. This is done to ensure compatibility with the existing P-Chain UTXO format, which specifies output addresses as 20 byte values. This is secure assuming there are no hash collisions between secp256k1 derived addresses and Warp Addressed Call derived addresses.

The produced address is already replay protected since it includes the sourceChainID and additionally 
This enables any P-Chain UTXO to be controlled directly by an account on any chain within the Avalanche Ecosystem


## Backwards Compatibility

Adding an alternative signature scheme to the P-Chain changes transaction validity and requires a coordinated network upgrade.

## Reference Implementation

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

## Open Questions

How can Warp Addressed Transactions perform UTXO set management on the P-Chain?

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Acknowledgements

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
