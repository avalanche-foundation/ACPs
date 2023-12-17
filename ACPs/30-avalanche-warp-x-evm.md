```text
ACP: 30
Title: Integrate Avalanche Warp Messaging into the EVM
Author(s): Aaron Buchwald <aaron.buchwald56@gmail.com>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/33
Status: Implementable
Track: Standards
```

## Abstract

Integrate Avalanche Warp Messaging into the C-Chain and Subnet-EVM in order to bring Cross-Subnet Communication to the EVM on Avalanche.

## Motivation

Avalanche Subnets enable the creation of independent blockchains within the Avalanche Network. Each Avalanche Subnet registers its validator set on the Avalanche P-Chain, which serves as an effective "membership chain" for the entire Avalanche Ecosystem.

By providing read access to the validator set of every Subnet on the Avalanche Network, any Subnet can look up the validator set of any other Subnet within the Avalanche Ecosystem to verify an Avalanche Warp Message, which replaces the need for point-to-point exchange of validator set info between Subnets. This enables a light weight protocol that allows seamless, on-demand communication between Subnets.

For more information on the Avalanche Warp Messaging message and payload formats see here:

- [AWM Message Format](https://github.com/ava-labs/avalanchego/tree/v1.10.15/vms/platformvm/warp/README.md)
- [Payload Format](https://github.com/ava-labs/avalanchego/tree/v1.10.15/vms/platformvm/warp/payload/README.md)

This ACP proposes to activate Avalanche Warp Messaging on the C-Chain and offer compatible support in Subnet-EVM to provide the first standard implementation of AWM in production on the Avalanche Network.

## Specification

The specification will be broken down into the Solidity interface of the Warp Precompile, a Golang example implementation, the predicate verification, and the proposed gas costs for the Warp Precompile.

The Warp Precompile address is `0x0200000000000000000000000000000000000005`.

### Precompile Solidity Interface

```solidity
// (c) 2022-2023, Ava Labs, Inc. All rights reserved.
// See the file LICENSE for licensing terms.

// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

struct WarpMessage {
  bytes32 sourceChainID;
  address originSenderAddress;
  bytes payload;
}

struct WarpBlockHash {
  bytes32 sourceChainID;
  bytes32 blockHash;
}

interface IWarpMessenger {
  event SendWarpMessage(address indexed sender, bytes32 indexed messageID, bytes message);

  // sendWarpMessage emits a request for the subnet to send a warp message from [msg.sender]
  // with the specified parameters.
  // This emits a SendWarpMessage log from the precompile. When the corresponding block is accepted
  // the Accept hook of the Warp precompile is invoked with all accepted logs emitted by the Warp
  // precompile.
  // Each validator then adds the UnsignedWarpMessage encoded in the log to the set of messages
  // it is willing to sign for an off-chain relayer to aggregate Warp signatures.
  function sendWarpMessage(bytes calldata payload) external returns (bytes32 messageID);

  // getVerifiedWarpMessage parses the pre-verified warp message in the
  // predicate storage slots as a WarpMessage and returns it to the caller.
  // If the message exists and passes verification, returns the verified message
  // and true.
  // Otherwise, returns false and the empty value for the message.
  function getVerifiedWarpMessage(uint32 index) external view returns (WarpMessage calldata message, bool valid);

  // getVerifiedWarpBlockHash parses the pre-verified WarpBlockHash message in the
  // predicate storage slots as a WarpBlockHash message and returns it to the caller.
  // If the message exists and passes verification, returns the verified message
  // and true.
  // Otherwise, returns false and the empty value for the message.
  function getVerifiedWarpBlockHash(
    uint32 index
  ) external view returns (WarpBlockHash calldata warpBlockHash, bool valid);

  // getBlockchainID returns the snow.Context BlockchainID of this chain.
  // This blockchainID is the hash of the transaction that created this blockchain on the P-Chain
  // and is not related to the Ethereum ChainID.
  function getBlockchainID() external view returns (bytes32 blockchainID);
}
```

### Warp Predicates and Pre-Verification

Signed Avalanche Warp Messages are encoded in the [EIP-2930 Access List](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-2930.md) of a transaction, so that they can be pre-verified before executing the transactions in the block.

The access list can specify any number of access tuples: a pair of an address and an array of storage slots in EIP-2930. Warp Predicate verification borrows this functionality to encode signed warp messages according to the serialization format defined [here](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/predicate/Predicate.md).

Each Warp specific access tuple included in the access list specifies the Warp Precompile address as the address. The first tuple that specifies the Warp Precompile address is considered to be at index. Each subsequent access tuple that specifies the Warp Precompile address increases the Warp Message index by 1. Access tuples that specify any other address are not included in calculating the index for a specific warp message.

Avalanche Warp Messages are pre-verified (prior to block execution), and outputs a bitset for each transaction where a 1 indicates an Avalanche Warp Message that failed verification at that index. Throughout the EVM execution, the Warp Precompile checks the status of the resulting bit set to determine whether pre-verified messages are considered valid. This has the additional benefit of encoding the Warp pre-verification results in the block, so that verifying a historical block can use the encoded results instead of needing to access potentially old P-Chain state. The result bitset is encoded in the block according to the predicate result specification [here](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/predicate/Results.md).

Each Warp Message in the access list is charged gas to pay for verifying the Warp Message (gas costs are covered below) and is verified with the following steps (see [here](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/config.go#L218) for reference implementation):

1. Unpack the predicate bytes
2. Parse the signed Avalanche Warp Message
3. Verify the signature according to the AWM spec in AvalancheGo [here](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/config.go#L218) (the quorum numerator/denominator for the C-Chain is 67/100 and is configurable in Subnet-EVM)

### Precompile Implementation

All types, events, and function arguments/outputs are encoded using the ABI package according to the official [Solidity ABI Specification](https://docs.soliditylang.org/en/latest/abi-spec.html).

When the precompile is invoked with a given `calldata` argument, the first four bytes (`calldata[0:4]`) are read as the [function selector](https://docs.soliditylang.org/en/latest/abi-spec.html#function-selector). If the function selector matches the function selector of one of the functions defined by the Solidity interface, the contract invokes the corresponding execution function with the remaining calldata ie. `calldata[4:]`.

For the full specification of the execution functions defined in the Solidity interface, see the reference implementation here:

- [sendWarpMessage](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/contract.go#L226)
- [getVerifiedWarpMessage](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/contract.go#L187)
- [getVerifiedWarpBlockHash](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/contract.go#L145)
- [getBlockchainID](https://github.com/ava-labs/subnet-evm/blob/v0.5.9/x/warp/contract.go#L96)

### Gas Costs

The Warp Precompile charges gas during the verification of included Avalanche Warp Messages, which is included in the intrinsic gas cost of the transaction, and during the execution of the precompile.

#### Verification Gas Costs

Pre-verification charges the following costs for each Avalanche Warp Message:

- GasCostPerSignatureVerification: 20000
- GasCostPerWarpMessageBytes: 100
- GasCostPerWarpSigner: 500

These numbers were determined experimentally using the benchmarks available [here](https://github.com/ava-labs/subnet-evm/blob/master/x/warp/predicate_test.go#L687) to target approximately the same mgas/s as existing precompile benchmarks in the EVM, which ranges between 50-200 mgas/s.

In addition to the benchmarks, the following assumptions and goals were taken into account:

- BLS Public Key Aggregation is extremely fast, resulting in charging more for the base cost of a single BLS Multi-Signature Verification than for adding an additional public key
- The cost per byte included in the transaction should be strictly higher for including Avalanche Warp Messages than via transaction calldata, so that the Warp Precompile does not change the worst case maximum block size

#### Execution Gas Costs

The execution gas costs were determined by summing the cost of the EVM operations that are performed throughout the execution of the precompile with special consideration for added functionality that does not have an existing corollary within the EVM.

##### sendWarpMessage

`sendWarpMessage` charges a base cost of 41,875 gas

This is comprised of charging for the following components:

- 375 gas / log operation
- 4 topics * 375 gas / topic
- 20k gas to produce and serve a BLS Signature
- 20k gas to store the Unsigned Warp Message

This charges 20k gas for storing an Unsigned Warp Message although the message is stored in an independent key-value database instead of the active state. This makes it less expensive to store, so 20k gas is a conservative estimate.

Additionally, the cost of serving valid signatures is significantly cheaper than serving state sync and bootstrapping requests, so the cost to validators of serving signatures over time is not considered a significant concern.

##### getBlockchainID

`getBlockchainID` charges 2 gas to serve an already in-memory 32 byte valu commensurate with existing in-memory operations.

##### getVerifiedWarpBlockHash / getVerifiedWarpMessage

`GetVerifiedWarpMessageBaseCost` charges 2 gas for serving a Warp Message (either payload type). Warp message are already in-memory, so it charges 2 gas for access.

`GasCostPerWarpMessageBytes` charges 100 gas per byte of the Avalanche Warp Message that is unpacked into a Solidity struct.

## Backwards Compatibility

Existing EVM opcodes and precompiles are not modified by activating Avalanche Warp Messaging in the EVM. This is an additive change to activate a Warp Precompile on the Avalanche C-Chain and can be scheduled for activation in any VM running on Avalanche Subnets that are capable of sending / verifying the specified payload types.

## Reference Implementation

A full reference implementation can be found in Subnet-EVM v0.5.9 [here](https://github.com/ava-labs/subnet-evm/tree/v0.5.9/x/warp).

## Security Considerations

Verifying an Avalanche Warp Message requires reading the source subnet's validator set at the P-Chain height specified in the [Snowman++ Block Extension](https://github.com/ava-labs/avalanchego/blob/v1.10.15/vms/proposervm/README.md#snowman-block-extension). The Avalanche PlatformVM provides the current state of the Avalanche P-Chain and maintains reverse diff-layers in order to compute Subnets' validator sets at historical points in time.

As a result, verifying a historical Avalanche Warp Message that references an old P-Chain height requires applying diff-layers from the current state back to the referenced P-Chain height. As Subnets and the P-Chain continue to produce and accept new blocks, verifying the Warp Messages in historical blocks becomes increasingly expensive.

To efficiently handle historical blocks containing Avalanche Warp Messages, the EVM uses the result bitset encoded in the block to determine the validity of Avalanche Warp Messages without requiring a historical P-Chain state lookup. This is considered secure because the network already verified the Avalanche Warp Messages when they were originally verified and accepted.

## Open Questions

_How should validator set lookups in Warp Message verification be effectively charged for gas?_

The verification cost of performing a validator set lookup on the P-Chain is currently excluded from the implementation. The cost of this lookup is variable depending on how old the referenced P-Chain height is from the perspective of each validator.

[Ongoing work](https://github.com/ava-labs/avalanchego/pull/1611) can parallelize P-Chain validator set lookups and message verification to reduce the impact on block verification latency to be negligible and reduce costs to reflect the additional bandwidth of encoding Avalanche Warp Messages in the transaction.

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Acknowledgements

Avalanche Warp Messaging and this effort to integrate it into the EVM has been a monumental effort. Thanks to all of the contributors who contributed their ideas, feedback, and development to this effort.

@stephenbuttolph
@patrick-ogrady
@michaelkaplan13
@minghinmatthewlam
@cam-schultz
@xanderdunn
@darioush
@ceyonur

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

