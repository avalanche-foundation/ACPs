```text
ACP: 118
Title: Warp Signature Interface Standard
Author(s): Cam Schultz (https://github.com/cam-schultz)
Discussions-To: 
Status: Proposed
Track: Standards Track
```

## Abstract

Proposes handling Warp signature requests from peers at the protocol level, rather than leaving it to VMs to implement. This will standardize the signature request interface, allowing for more flexible, VM-agnostic signature aggregation implementations.

## Motivation

Warp message signatures consist of an aggregate BLS signature composed of the individual signatures of a subnet's validators. Individual signatures need to be retreivable by the party that wishes to construct an aggregate signature. At present, this is left to VMs to implement, as is the case with [Subnet EVM](https://github.com/ava-labs/subnet-evm/blob/v0.6.6/plugin/evm/message/signature_request.go#20) and [Coreth](https://github.com/ava-labs/coreth/blob/v0.13.6-rc.0/plugin/evm/message/signature_request.go#L20)

This creates friction in applications that are intended to operate across many VMs (or distinct implementations of the same VM). As an example, the reference Warp message relayer implementation, [awm-relayer](https://github.com/ava-labs/awm-relayer), fetches individual signatures from validators and aggregates them before sending the Warp message to its destination chain for verification. However, Subnet EVM and Coreth have distinct codecs, requiring the relayer to [switch](https://github.com/ava-labs/awm-relayer/blob/v1.4.0-rc.0/relayer/application_relayer.go#L372) according to the target codebase.

Another example is ACP-75, which aims to implement acceptance proofs using Warp. The signature aggregation mechanism is not [specified](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/75-acceptance-proofs/README.md#signature-aggregation), which is a blocker for that ACP to be marked implementable.

Standardizing the Warp Signature Request format by implementing it as a protocol-level (AvalancheGo) AppRequest type would simplify the implementation of ACP-75, and streamline signature aggregation for out-of-protocol services such as Warp message relayers.

## Specification

Subnet EVM and Coreth currently handle Warp message signature requests from peers by decoding the `AppRequest` bytes (forwarded from Avalanchego via the `AppHandler` interface) using a type registered with the message codec:

```
type MessageSignatureRequest struct {
	MessageID ids.ID `serialize:"true"`
}
```

`MessageID` represents the VM-agnostic Warp message ID, and could be ported as-is to the protocol. To do so, we would [define](https://github.com/ava-labs/avalanchego/blob/v1.11.10-status-removal/proto/p2p/p2p.proto) a new P2P `Message` type:

```diff
message Message {
    ...
    // App messages:
    AppRequest app_request = 30;
    AppResponse app_response = 31;
    AppGossip app_gossip = 32;
    AppError app_error = 34;
    
+   // Warp signature messages:
+   MessageSignatureRequest = 35;
}
```

Where `MessageSignatureRequest` is defined identically to Subnet EVM/Coreth:
```
message MessageSignature {
    bytes message_id = 1;
}
```

From there, the `MessageSignatureRequest` would be forwarded to the VM to retreive the signature.

## Open Questions

How should signature requests be forwarded to the VM? Here are a couple of candidates:
- RPC Chain VM and Platform VM use the `Signer` [service](https://github.com/ava-labs/avalanchego/blob/v1.11.10-status-removal/proto/warp/message.proto) to issue and response to signature requests. Subnet EVM and Coreth could be modified to use this service.
- Subnet EVM and Coreth currently [implement](https://github.com/ava-labs/subnet-evm/blob/v0.6.6/peer/network.go) `AppHandler` to directly receive other p2p messages. A `SignatureHandler` interface could be provided for VMs to implement.

In addition to generic Warp message signature request handling, Subnet EVM and Coreth also implement handling for Warp messages whose payload consists solely of a block hash. This is provides first class support for state verification via Warp in the VM. The proposed `MessageSignatureRequest` p2p message type does not distinguish between Warp message payload types, so it would be left to the VM to make that distinction.

## Security Considerations

## Backwards Compatibility
This change is backwards compatible, though nodes that do not implement `MessageSignatureRequest` as described will drop incoming requests. 

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable".

## Acknowledgements
Thanks to @joshua-kim and @iansuvak for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).