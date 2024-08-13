| ACP | 131 |
| :--- | :--- |
| **Title** | Activate Cancun EIPs on C-Chain and Subnet-EVM chains |
| **Author(s)** | Darioush Jalali ([@darioush](https://github.com/darioush)), Ceyhun Onur ([@ceyonur](https://github.com/ceyonur)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/139)) |
| **Track** | Standards, Subnet |

## Abstract

Enable new EVM opcodes and opcode changes in accordance with the following EIPs on the Avalanche C-Chain and Subnet-EVM chains:
- [EIP-4844: BLOBHASH opcode](https://eips.ethereum.org/EIPS/eip-4844)
- [EIP-7516: BLOBBASEFEE opcode](https://eips.ethereum.org/EIPS/eip-7516)
- [EIP-1153: Transient storage](https://eips.ethereum.org/EIPS/eip-1153)
- [EIP-5656: MCOPY opcode](https://eips.ethereum.org/EIPS/eip-5656)
- [EIP-6780: SELFDESTRUCT only in same transaction](https://eips.ethereum.org/EIPS/eip-6780)

Note blob transactions from EIP-4844 are excluded and blocks containing them will still be considered invalid.

## Motivation

The listed EIPs were activated on Ethereum mainnet as part of the [Cancun upgrade](https://github.com/ethereum/execution-specs/blob/master/network-upgrades/mainnet-upgrades/cancun.md#included-eips). This proposal is to activate them on the Avalanche C-Chain in the next network upgrade, to maintain compatibility with upstream EVM tooling, infrastructure, and developer experience (e.g., Solidity compiler defaults >= [0.8.25](https://github.com/ethereum/solidity/releases/tag/v0.8.25)). Additionally, it recommends the activation of the same EIPs on Subnet-EVM chains.

## Specification & Reference Implementation

The opcodes (EVM exceution modifications) and block header modifications should be adopted as specified in the EIPs themselves. Other changes such as enabling new transaction types or mempool modifications are not in scope (specifically blob transactions from EIP-4844 are excluded and blocks containing them are considered invalid). ANCs (Avalanche Network Clients) can adopt the implementation as specified in the [coreth](https://github.com/ava-labs/coreth) repository, which was adopted from the [go-ethereum v1.13.8](https://github.com/ethereum/go-ethereum/releases/tag/v1.13.8) release in this [PR](https://github.com/ava-labs/coreth/pull/550). In particular, note the following code:

- [Activation of new opcodes](https://github.com/ava-labs/coreth/blob/7b875dc21772c1bb9e9de5bc2b31e88c53055e26/core/vm/jump_table.go#L93)
- Activation of Cancun in next Avalanche upgrade:
  - [C-Chain](https://github.com/ava-labs/coreth/pull/610)
  - [Subnet-EVM chains](https://github.com/ava-labs/subnet-evm/blob/fa909031ed148484c5072d949c5ed73d915ce1ed/params/config_extra.go#L186)
- `ParentBeaconRoot` is enforced to be included and the zero value [here](https://github.com/ava-labs/coreth/blob/7b875dc21772c1bb9e9de5bc2b31e88c53055e26/plugin/evm/block_verification.go#L287-L288). This field is retained for future use and compatibility with upstream tooling.
- Forbids blob transactions by enforcing `BlobGasUsed` to be 0 [here](https://github.com/ava-labs/coreth/pull/611/files#diff-532a2c6a5365d863807de5b435d8d6475552904679fd611b1b4b10d3bf4f5010R267).

_Note:_ Subnets are sovereign in regards to their validator set and state transition rules, and can choose to opt out of this proposal by making a code change in their respective Subnet-EVM client.

## Backwards Compatibility

The original EIP authors highlighted the following considerations. For full details, refer to the original EIPs:

- [EIP-4844](https://eips.ethereum.org/EIPS/eip-4844#backwards-compatibility): Blob transactions are not proposed to be enabled on Avalanche, so concerns related to mempool or transaction data availability are not applicable.
- [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780#backwards-compatibility) "Contracts that depended on re-deploying contracts at the same address using CREATE2 (after a SELFDESTRUCT) will no longer function properly if the created contract does not call SELFDESTRUCT within the same transaction."

Adoption of this ACP modifies consensus rules for the C-Chain, therefore it requires a network upgrade. It is recommended that Subnet-EVM chains also adopt this ACP and follow the same upgrade time as Avalanche's next network upgrade.

## Security Considerations

Refer to the original EIPs for security considerations:
- [EIP 1153](https://eips.ethereum.org/EIPS/eip-1153#security-considerations)
- [EIP 4788](https://eips.ethereum.org/EIPS/eip-4788#security-considerations)
- [EIP 4844](https://eips.ethereum.org/EIPS/eip-4844#security-considerations)
- [EIP 5656](https://eips.ethereum.org/EIPS/eip-5656#security-considerations)
- [EIP 6780](https://eips.ethereum.org/EIPS/eip-6780#security-considerations)
- [EIP 7516](https://eips.ethereum.org/EIPS/eip-7516#security-considerations)

## Open Questions

No open questions.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
