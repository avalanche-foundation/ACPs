```text
ACP: 108
Title: EVM Event Importing Standard
Author(s): Michael Kaplan (https://github.com/mkaplan13)
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/109
Status: Proposed
Track: Best Practices Track
```

## Abstract

Defines a standard smart contract interface and abstract implementation for importing EVM events from any blockchain within Avalanche using [Avalanche Warp Messaging](https://docs.avax.network/build/cross-chain/awm/overview).

## Motivation

The implementation of Avalanche Warp Messaging within `coreth` and `subnet-evm` exposes a [mechanism for getting authenticated hashes of blocks](https://github.com/ava-labs/subnet-evm/blob/master/contracts/contracts/interfaces/IWarpMessenger.sol#L43) that have been accepted on blockchains within Avalanche. There is currently no clear standard for using authenticated block hashes in smart contracts within Avalanche, making it difficult to build applications that leverage this mechanism. In order to make effective use of authenticated block hashes, contracts must be provided encoded block headers that match the authenticated block hashes and also Merkle proofs that are verified against the state or receipts root contained in the block header. 

With a standard interface and abstract contract implemetation that handles the authentication of block hashes and verification of Merkle proofs, smart contract developers on Avalanche will be able to much more easily create applications that leverage data from other Avalanche blockchains. These type of cross-chain application do not require any direct interaction on the source chain.

## Specification

### Event Importing Interface

We propose that smart contracts importing EVM events emitted by other blockchains within Avalanche implement the following interface.

#### Methods

Imports the EVM event uniquely identified by the source blockchain ID, block header, transaction index, and log index. The event data is provided in the `receiptProof`, which must be a valid Merkle proof of the receipt for the given `txIndex` against the receipt root of the `blockHeader`. Must emit an `EventImported` event upon success. Should validate that the `blockHeader` matches an authenticated block hash from the `sourceBlockchainID`.
```solidity
function importEvent(
    bytes32 sourceBlockchainID,
    bytes calldata blockHeader,
    uint256 txIndex,
    bytes[] calldata receiptProof,
    uint256 logIndex
) external;
```

This interface does not require that the Warp precompile is used to authenticate block hashes. Implementations could:
- Use an external block hash verification mechanism, such as trusted oracles.
- Use the Warp precompile to authenticate block hashes provided in the transaction calling `importEvent`.
- Look up previously authenticated block hashes from another contract, such that multiple events could be imported from the same block hash, which would only need to be authenticated once.

#### Events

Must trigger when an EVM event is imported.
```solidity
event EventImported(
    bytes32 indexed sourceBlockchainID,
    bytes32 indexed sourceBlockHash,
    address indexed loggerAddress,
    uint256 txIndex,
    uint256 logIndex
);
```

### Event Importing Abstract Contract

Applications importing EVM events emitted by other blockchains within Avalanche should be able to use a standard abstract implementation of the `importEvent` interface. This abstract implementation should handle:
- Authenticating block hashes from other chains using Warp precompile.
- Verifying that the encoded `blockHeader` matches the imported block hash.
- Verifying the Merkle `receiptProof` for the given `txIndex` against the receipt root of the provided `blockHeader`.
- Decoding the event log identified by `logIndex` from the receipt obtained from verifying the `receiptProof`.

Inheriting contracts should only need to define the logic to be executed when an event is imported. This is done by providing an implementation of the following internal function, called by `importEvent`.

```solidity
function _onEventImport(EVMEventInfo memory eventInfo) internal virtual;
```

Where the `EVMEventInfo` struct is defined as:

```solidity
struct EVMLog {
    address loggerAddress;
    bytes32[] topics;
    bytes data;
}

struct EVMEventInfo {
    bytes32 blockchainID;
    uint256 blockNumber;
    uint256 txIndex;
    uint256 logIndex;
    EVMLog log;
}
```

## Reference Implementation

See reference implementation on [Github here](https://github.com/ava-labs/event-importer-poc).

In addition to implementing the interface and abstract contract described above, the reference implementation shows how transactions can be constructed to import events using Warp block hash signatures.

## Open Questions

See [here](https://github.com/ava-labs/event-importer-poc?tab=readme-ov-file#open-questions-and-considerations).

## Security Considerations

The correctness of a contract using block hashes to prove that a specific event was emitted within that block depends on the correctness of:
1. The mechanism for authenticating that a block hash was finalized on another blockchain.
2. The Merkle proof validation library used to prove that a specific transactoin receipt was included in the given block.

For considerations on using Avalanche Warp Messaging to authenticate block hashes, see [here](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/30-avalanche-warp-x-evm#security-considerations).

To improve confidence in the correctness of the Merkle proof validation used in implementations, well-audited and widely used libraries should be used.

## Acknowledgements

Using Merkle proofs to verify events/state against root hashes is not a new idea. Protocols such as [IBC](https://ibc.cosmos.network/v8/), [Rainbow Bridge](https://github.com/Near-One/rainbow-bridge), and [LayerZero](https://layerzero.network/publications/LayerZero_Whitepaper_V1.1.0.pdf), among others, have previously suggested using Merkle proofs in a similar manner.

Thanks to [@aaronbuchwald](https://github.com/aaronbuchwald) for proposing the `getVerifiedWarpBlockHash` interface be included in the AWM implemenation within Avalanche EVMs, which enables this type of use case.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
