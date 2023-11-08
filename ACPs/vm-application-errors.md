```text
ACP: 25
Title: Virtual Machine Application Errors
Author(s): Joshua Kim <https://github.com/joshua-kim>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/TODO //TODO
Status: Proposed
Track: Standards
```

## Abstract

Support a way for a Virtual Machine (VM) to signal application-defined error conditions to another VM.

## Motivation

The motivation is critical for ACPs that want to change the functionality of the Avalanche Network. It should clearly explain why the existing Avalanche Network implementation is inadequate to address the problem that the ACP solves.

VMs are able to build their own peer-to-peer application protocols using the `AppRequest`, `AppResponse`, and `AppGossip` primitives. When a peer makes a request to another peer, an error can only be propagated by encoding an error inside of `AppResponse`.

Having a native application error type would offer a more powerful abstraction where Avalanche nodes would be able to score peers based on perceived errors. This is not currently possible because Avalanche networking isn't aware of the specific implementation details of the messages being delivered to VMs. A native application error type would also make it more clear that all `AppRequest` messages _should_ either receive a corresponding response or error, so that clients in the common unhappy path do not have to resort to timing out failed requests.

## Specification

### Message 

This modifies the p2p specification by introducing a new [protobuf](https://protobuf.dev/) message type:

```
message AppRequestFailed {
  bytes chain_id = 1;
  uint32 request_id = 2;
  uint32 error_code = 3;
  string error_message = 4;
}
```

1. `chain_id`: Reserves field 1. Senders **must** use the same chain id of from the original `AppRequest` this `AppRequestFailed` message is being sent in response to.
2. `request_id`: Reserves field 2. Senders **must** use the same request id from the original `AppRequest` this `AppRequestFailed` message is being sent in response to.
3. `error_code`: Reserves field 3. Application defined error code. Implementations _should_ use the same error codes for the same conditions to allow clients to error match. Negative error codes are reserved for protocol defined errors. VMs may reserve any error code greater than zero.
4. `error_message`: Reserves field 4. Application defined human-readable error message that _should not_ be used for error matching. For error matching, use `error_code`.

### Reserved Errors
The following error codes are currently reserved by the Avalanche protocol:
| Error Code | Description     |
| ---------- | --------------- |
| 0          | undefined       |
| -1         | network timeout |

### Handling

Clients **must** respond to an inbound `AppRequest` message with either a corresponding `AppResponse` to indicate a successful response, or an `AppRequestFailed` to indicate an error condition.

## Backwards Compatibility

This new message type requires an network activation to require either an `AppResponse` or an `AppRequestFailed` as a required response to an `AppRequest`.

## Reference Implementation

- Message definition: https://github.com/ava-labs/avalanchego/pull/2111
- Handling: https://github.com/ava-labs/avalanchego/pull/2248

## Security Considerations

Optional section that discusses the security implications/considerations relevant to the proposed change.

Clients should be aware that peers can aribtrarily send `AppRequestFailed` messages to invoke error handling logic in a VM.

## Open Questions

Optional section that lists any concerns that should be resolved prior to implementation.

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). This PR must include a message + signature indicating ownership of a given amount of $AVAX.

### Supporters
                                                                                                                                                                                                                                                                      * `<message>/<signature>`

### Objectors
                                                                                                                                                                                                                                                                      * `<message>/<signature>`

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
