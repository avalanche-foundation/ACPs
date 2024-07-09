```text
ACP: 118
Title: Warp Signature Interface Standard
Author(s): Cam Schultz (https://github.com/cam-schultz)
Discussions-To: 
Status: Proposed
Track: Best Practices Track
```

## Abstract

Proposes a standard Warp signature request AppRequest types so that Warp signatures may be requested in a VM-agnostic manner. To make this concrete, this standard type should be defined in AvalancheGo such that VMs can import it at the source code level. This will also simplify external signature aggregator implementations by allowing them to depend only on AvalancheGo for message construction, rather than individual VM codecs.

## Motivation

Warp message signatures consist of an aggregate BLS signature composed of the individual signatures of a subnet's validators. Individual signatures need to be retreivable by the party that wishes to construct an aggregate signature. At present, this is left to VMs to implement, as is the case with [Subnet EVM](https://github.com/ava-labs/subnet-evm/blob/v0.6.6/plugin/evm/message/signature_request.go#20) and [Coreth](https://github.com/ava-labs/coreth/blob/v0.13.6-rc.0/plugin/evm/message/signature_request.go#L20)

This creates friction in applications that are intended to operate across many VMs (or distinct implementations of the same VM). As an example, the reference Warp message relayer implementation, [awm-relayer](https://github.com/ava-labs/awm-relayer), fetches individual signatures from validators and aggregates them before sending the Warp message to its destination chain for verification. However, Subnet EVM and Coreth have distinct codecs, requiring the relayer to [switch](https://github.com/ava-labs/awm-relayer/blob/v1.4.0-rc.0/relayer/application_relayer.go#L372) according to the target codebase.

Another example is ACP-75, which aims to implement acceptance proofs using Warp. The signature aggregation mechanism is not [specified](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/75-acceptance-proofs/README.md#signature-aggregation), which is a blocker for that ACP to be marked implementable.

Standardizing the Warp Signature Request format by implementing it as an AppRequest type in AvlanahceGo would simplify the implementation of ACP-75, and streamline signature aggregation for out-of-protocol services such as Warp message relayers.

## Specification

Subnet EVM and Coreth currently handle Warp message signature requests from peers by decoding the `AppRequest` bytes (forwarded from Avalanchego via the `AppHandler` interface) using a types registered with their message codec:

```
type MessageSignatureRequest struct {
	MessageID ids.ID `serialize:"true"`
}
```
and
```
type BlockSignatureRequest struct {
	blockID ids.ID `serialize:"true"`
}
```

`MessageID` represents the VM-agnostic Warp message ID, and could be ported as-is to AvalancheGo. `BlockID` will instead be generalized to represent a 32-byte hash over some data. Interpreting that hash is left to VMs. For example, `Hash` could represent a transaction hash that a node only signs on receiving a `HashSignatureRequest`. 
```
type MessageSignatureRequest struct {
	MessageID ids.ID `serialize:"true"`
}
```
and
```
type HashSignatureRequest struct {
	Hash ids.ID `serialize:"true"`
}
```

These types should be implemented in AvalancheGo and exported for VMs and arbitrary aggregators to consume. A possible extension to encourage proper use would be to initialize a `WarpCodec` with these types, and export it such that VMs can extend it with additional `AppRequest` message tyeps.

## Security Considerations
TODO

## Backwards Compatibility
This change is backwards compatible for VMs, as nodes running older versions that do not support the new message types will simply drop incoming messages.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable".

## Acknowledgements
Thanks to @joshua-kim and @iansuvak for discussion and feedback on this ACP.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).