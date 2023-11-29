```text
ACP: 24
Title: Activate Shanghai EIPs on C-Chain
Author(s): Darioush Jalali <https://github.com/darioush>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/27
Status: Implementable
Track: Standards
```

## Abstract

This ACP proposes the adoption of the following EIPs on the Avalanche C-Chain network:
- [EIP-3651: Warm COINBASE](https://eips.ethereum.org/EIPS/eip-3651)
- [EIP-3855: PUSH0 instruction](https://eips.ethereum.org/EIPS/eip-3855)
- [EIP-3860: Limit and meter initcode](https://eips.ethereum.org/EIPS/eip-3860)
- [EIP-6049: Deprecate SELFDESTRUCT](https://eips.ethereum.org/EIPS/eip-6049)

## Motivation

The listed EIPs were activated on Ethereum mainnet as part of the [Shanghai upgrade](https://github.com/ethereum/execution-specs/blob/master/network-upgrades/mainnet-upgrades/shanghai.md#included-eips). This ACP proposes their activation on the Avalanche C-Chain in the next network upgrade. This maintains compatibility with upstream EVM tooling, infrastructure, and developer experience (e.g., Solidity compiler >= [0.8.20](https://github.com/ethereum/solidity/releases/tag/v0.8.20)).

## Specification & Reference Implementation

This ACP proposes the EIPs be adopted as specified in the EIPs themselves. ANCs (Avalanche Network Clients) can adopt the implementation as specified in the [coreth](https://github.com/ava-labs/coreth) repository, which was adopted from the [go-ethereum v1.12.0](https://github.com/ethereum/go-ethereum/releases/tag/v1.12.0) release in this [PR](https://github.com/ava-labs/coreth/pull/277). In particular, note the following code:

- [Activation of new opcode and dynamic gas calculations](https://github.com/ava-labs/coreth/blob/bf2051729c7aa0c4ed8848ad3a78e241a791b968/core/vm/jump_table.go#L92)
- [EIP-3860 intrinsic gas calculations](https://github.com/ava-labs/coreth/blob/bf2051729c7aa0c4ed8848ad3a78e241a791b968/core/state_transition.go#L112-L113)
- [EIP-3651 warm coinbase](https://github.com/ava-labs/coreth/blob/bf2051729c7aa0c4ed8848ad3a78e241a791b968/core/state/statedb.go#L1197-L1199)
- Note EIP-6049 marks SELFDESTRUCT as deprecated, but does not remove it. The implementation in coreth is unchanged.

## Backwards Compatibility

The following backward compatibility considerations were highlighted by the original EIP authors:

- [EIP-3855](https://eips.ethereum.org/EIPS/eip-3855#backwards-compatibility): "... introduces a new opcode which did not exists previously. Already deployed contracts using this opcode could change their behaviour after this EIP".
- [EIP-3860](https://eips.ethereum.org/EIPS/eip-3860#backwards-compatibility) "Already deployed contracts should not be effected, but certain transactions (with initcode beyond the proposed limit) would still be includable in a block, but result in an exceptional abort."

Adoption of this ACP modifies consensus rules for the C-Chain, therefore it requires a network upgrade.

## Security Considerations

Refer to the original EIPs for security considerations:
- [EIP 3855](https://eips.ethereum.org/EIPS/eip-3855#security-considerations)
- [EIP 3860](https://eips.ethereum.org/EIPS/eip-3860#security-considerations)

## Open Questions

No open questions.

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
