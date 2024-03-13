```text
ACP: 23
Title: P-Chain Native Transfers
Author(s): Dhruba Basu <https://github.com/dhrubabasu>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/37
Status: Activated
Track: Standards
```

## Abstract

Support native transfers on P-chain. This enables users to transfer P-chain assets without leaving the P-chain or using a transaction type that's not meant for native transfers.

## Motivation

Currently, the P-chain has no simple transfer transaction type. The X-chain supports this functionality through a `BaseTx`. Although the P-chain contains transaction types that extend `BaseTx`, the `BaseTx` transaction type itself is not a valid transaction. This leads to abnormal implementations of P-chain native transfers like in the AvalancheGo wallet which abuses [`CreateSubnetTx`](https://github.com/ava-labs/avalanchego/blob/v1.10.15/wallet/chain/p/builder.go#L54-L63) to replicate the functionality contained in `BaseTx`.

With the growing number of subnets slated for launch on the Avalanche network, simple transfers will be demanded more by users. While there are work-arounds as mentioned before, the network should support it natively to provide a cheaper option for both validators and end-users.

## Specification

To support `BaseTx`, Avalanche Network Clients (like AvalancheGo) must register `BaseTx` with the type ID `0x22` in codec version `0x00`.

For the specification of the transaction itself, see [here](https://github.com/ava-labs/avalanchego/blob/v1.10.15/vms/platformvm/txs/base_tx.go#L29). Note that most other P-chain transactions extend this type, the only change in this ACP is to register it as a valid transaction itself.

## Backwards Compatibility

Adding a new transaction type is an execution change and requires a mandatory upgrade for activation. Implementors must take care to reject this transaction prior to activation. This ACP only details the specification of the added `BaseTx` transaction type.

## Reference Implementation

An implementation of `BaseTx` support was created [here](https://github.com/ava-labs/avalanchego/pull/2232) and subsequently merged into AvalancheGo. Since the "D" Upgrade is not activated, this transaction will be rejected by AvalancheGo.

If modifications are made to the specification of the transaction as part of the ACP process, the code must be updated prior to activation.

## Security Considerations

The P-chain has fixed fees which does not place any limits on chain throughput. A potentially popular transaction type like `BaseTx` may cause periods of high usage. The reference implementation in AvalancheGo sets the transaction fee to 0.001 AVAX as a deterrent (equivalent to `ImportTx` and `ExportTx`). This should be sufficient for the time being but a dynamic fee mechanism will need to be added to the P-chain in the future to mitigate this security concern. This is not addressed in this ACP as it requires a larger change to the fee dynamics on the P-chain as a whole.

## Open Questions

No open questions.

## Acknowledgements

Thanks to [@StephenButtolph](https://github.com/StephenButtolph) and [@abi87](https://github.com/abi87) for their feedback on the reference implementation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).