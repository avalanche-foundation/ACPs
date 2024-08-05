| ACP | 118 |
| :--- | :--- |
| **Title** | Warp Signature Interface Standard |
| **Author(s)** | Cam Schultz ([@cam-schultz](https://github.com/cam-schultz)) |
| **Status** | Implementable ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/123)) |
| **Track** | Best Practices Track |

## Abstract

Proposes a standard [AppRequest](https://github.com/ava-labs/avalanchego/blob/master/proto/p2p/p2p.proto#L385) payload format type for requesting Warp signatures for the provided bytes, such that signatures may be requested in a VM-agnostic manner. To make this concrete, this standard type should be defined in AvalancheGo such that VMs can import it at the source code level. This will simplify signature aggregator implementations by allowing them to depend only on AvalancheGo for message construction, rather than individual VM codecs.

## Motivation

Warp message signatures consist of an aggregate BLS signature composed of the individual signatures of a subnet's validators. Individual signatures need to be retreivable by the party that wishes to construct an aggregate signature. At present, this is left to VMs to implement, as is the case with [Subnet EVM](https://github.com/ava-labs/subnet-evm/blob/v0.6.7/plugin/evm/message/signature_request.go#20) and [Coreth](https://github.com/ava-labs/coreth/blob/v0.13.6-rc.0/plugin/evm/message/signature_request.go#L20)

This creates friction in applications that are intended to operate across many VMs (or distinct implementations of the same VM). As an example, the reference Warp message relayer implementation, [awm-relayer](https://github.com/ava-labs/awm-relayer), fetches individual signatures from validators and aggregates them before sending the Warp message to its destination chain for verification. However, Subnet EVM and Coreth have distinct codecs, requiring the relayer to [switch](https://github.com/ava-labs/awm-relayer/blob/v1.4.0-rc.0/relayer/application_relayer.go#L372) according to the target codebase.

Another example is ACP-75, which aims to implement acceptance proofs using Warp. The signature aggregation mechanism is not [specified](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/75-acceptance-proofs/README.md#signature-aggregation), which is a blocker for that ACP to be marked implementable.

Standardizing the Warp Signature Request interface by defining it as a format for `AppRequest` message payloads in AvalancheGo would simplify the implementation of ACP-75, and streamline signature aggregation for out-of-protocol services such as Warp message relayers.

## Specification

We propose the following types, implemented as Protobuf types that may be decoded from the `AppRequest`/`AppResponse` `app_bytes` field. By way of example, this approach is currently used to [implement](https://github.com/ava-labs/avalanchego/blob/v1.11.10-status-removal/proto/sdk/sdk.proto#7) and [parse](https://github.com/ava-labs/avalanchego/blob/v1.11.10-status-removal/network/p2p/gossip/message.go#22) gossip `AppRequest` types.

- `SignatureRequest` includes two fields. `message` specifies the payload that the returned signature should correspond to, namely a serialized unsigned Warp message. `justification` specifies arbitrary data that the requested node may use to decide whether or not it is willing to sign `message`. `justification` may not be required by every VM implementation, but `message` should always contain the bytes to be signed. It is up to the VM to define the validity requirements for the `message` and `justification` payloads.

    ```protobuf
    message SignatureRequest {
        bytes message = 1;
        bytes justification = 2;
    }
    ```

- `SignatureResponse` is the corresponding `AppResponse` type that returns the requested signature. 

    ```protobuf
    message SignatureResponse {
        bytes signature = 1;
    }
    ```

### Handlers

For each of the above types, VMs must implement corresponding `AppRequest` and `AppResponse` handlers. The `AppRequest` handler should be [registered](https://github.com/ava-labs/avalanchego/blob/v1.11.10-status-removal/network/p2p/network.go#L173) using the canonical handler ID, defined as `2`.

## Use Cases

Generally speaking, `SignatureRequest` can be used to request a signature over a Warp message by serializing the unsigned Warp message into `message`, and populating `justification` as needed.

### Sign a known Warp Message

Subnet EVM and Coreth store messages that have been seen (i.e. on-chain message sent through the [Warp Precompile](https://github.com/ava-labs/subnet-evm/tree/v0.6.7/precompile/contracts/warp) and [off-chain](https://github.com/ava-labs/subnet-evm/blob/v0.6.7/plugin/evm/config.go#L226) Warp messages) such that a signature over that message can be provided on request. `SignatureRequest` can be used for this case by specifying the Warp message in `message`. The queried node may then look up the Warp message in its database and return the signature. In this case, `justification` is not needed.

### Attest to an on-chain event

Subnet EVM and Coreth also support attesting to block hashes via Warp, by serving signature requests made using the following `AppRequest` type:

```
type BlockSignatureRequest struct {
	BlockID ids.ID
}
```

`SignatureRequest` can achieve this by specifying an unsigned Warp message with the `BlockID` as the payload, and serializing that message into `message`. `justification` may optionally be used to provide additional context, such as a the block height of the given block ID.

### Confirm that an event did not occur

With [ACP-77](https://github.com/avalanche-foundation/ACPs/tree/main/ACPs/77-reinventing-subnets), Subnets will have the ability to manage their own validator sets. The Warp message payload contained in a `RegisterSubnetValidatorTx` includes an `expiry`, after which the specified validation ID (i.e. a unique hash over the Subnet ID, node ID, stake weight, and expiry) becomes invalid. The Subnet needs to know that this validation ID is expired so that it can keep its locally tracked validator set in sync with the P-Chain. We also assume that the P-Chain will not persist expired or invalid validation IDs. 

We can use `SignatureRequest` to construct a Warp message attesting that the validation ID expired. We do so by serializing an unsigned Warp message containing the validation ID into `message`, and providing the validation ID hash preimage in `justification` for the P-Chain to reconstruct the expired validation ID.

## Security Considerations

VMs have full latitude when implementing `SignatureRequest` handlers, and should take careful consideration of what `message` payloads their implementation should be willing to sign, given a `justification`. Some considerations include, but are not limited to:
- Input validation. Handlers should validate `message` and `justification` payloads to ensure that they decode to coherent types, and that they contain only expected data.
- Signature DoS. AvalancheGo's peer-to-peer networking stack implements message rate limiting to mitigate the risk of DoS, but VMs should also consider the cost of parsing and signing a `message` payload.
- Payload collision. `message` payloads should be implemented as distinct types that do not overlap with one another within the context of signed Warp messages from the VM. For instance, a `message` payload specifying 32-byte hash may be interpreted as a transaction hash, a block hash, or a blockchain ID.

## Backwards Compatibility

This change is backwards compatible for VMs, as nodes running older versions that do not support the new message types will simply drop incoming messages.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable".

A reference implementation containing the Protobuf types and the canonical handler ID can be found [here](https://github.com/ava-labs/avalanchego/pull/3218).

## Acknowledgements

Thanks to @joshua-kim, @iansuvak, @aaronbuchwald, @michaelkaplan13, and @StephenButtolph for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
