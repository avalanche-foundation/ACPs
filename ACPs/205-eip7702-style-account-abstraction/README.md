| ACP | 205 |
| :--- | :--- |
| **Title** | EIP7702-style Account Abstraction |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Arran Schlosberg ([@ARR4N](https://github.com/ARR4N)), [Aaron Buchwald](https://github.com/aaronbuchwald), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed, Implementable, Activated, Stale ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

[EIP-7702](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md) was activated on the Ethereum mainnet in May 2025 as part of the Pectra upgrade, and introduced a new transaction type that allows Externally Owned Accounts (EOAs) to set the code in their account. This was aimed at unlocking a handful of UX improvements around account abstraction, particularly the batching of multiple operations into single atomic transactions, the sponsoring transactions on another account's behalf, and the privilege de-escalation of various accounts.

This ACP proposes adding a similiar transaction type and functionality to Avalanche EVM chains in order to have them support the same style of UX available on Ethereum. Modifications to the mechanism are required in order for it to be safe when used in conjunction with the streaming asynchronous execution (SAE) mechanism proposed in [ACP-194](../194-streaming-asynchronous-execution/README.md).

## Motivation

The top-level motivation for this ACP is the same as the motivation described in original [EIP-7702 specification](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md#motivation). However, modifications to the original EIP-7702 mechanism are required in order for them to be safely used on EVM chains that use the ACP-194 SAE mechanism. 

There has been strong community feedback in support of ACP-194 for its potential to:
- Allow for increasing the target gas rate of Avalanche EVM chains, including the C-Chain
- Enable the use of an encrypted mempool to prevent front-running
- Enable the use of real time VRF during transaction execution

Given this strong support for SAE, in order to bring an EIP-7702-like mechanism (and its UX improvements) to Avalanche EVM chains in the future, the mechanism must not break the invariants required for SAE described below. 

### Invariants needed for ACP-194

There are [two invariants explicitly broken by the standard EIP-7702 specification](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md#backwards-compatibility) that are required for SAE. They are:

1. An account balance can only decrease as a result of a transaction originating from that account.
1. An EOA nonce may not increase after transaction execution has begun.

These invariants are required for SAE in order to be able to statically analyze that a transaction has the proper nonce and will have sufficient balance to pay for its worst case transaction fee, without executing the transaction. As described in the ACP-194 specification, this lightweight analysis of transactions in blocks allows blocks to be accepted by consensus with the guarantee that can be executed successfully. Only after block acceptance are the transactions within the block then put into a queue to be executed asynchronously. If the execution of transactions in the queue can decrease EOA account balances or change an EOA's current nonce, then block verification is unable to ensure that transactions in the block are valid and can pay for their fees.

Notably, EIP-7702 breaking these two invariants poses similar issues for mempool verification on Ethereum. As [noted in the specification](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md#transaction-propagation), EIP-7702 makes it "possible to cause transactions from other accounts to become stale" and this "poses some challenges for transaction propagation" because nodes now cannot "statically determine the validity of transactions for that account". In syncrhonous execution environments such as Ethereum, these issues only pose potential DOS risks to the public transaction mempool. Under an asynchronous execution scheme, the issues pose DOS risks to the chain itself since the invalidated transactions can be included in blocks prior to their execution.

## Specification

The same ["set code transaction" as specicied in EIP-7702](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md?ref=blockhead.co#set-code-transaction) will be added to Avalanache EVM implementations. The behavior of the transaction is the same as specified in EIP-7702 with one modification: any native token balances transferred from a delegated account via code execution will be deducted from a new "reserved balance", which can be managed for any account via new `ReservedBalanceManager` precompile, as specified below. This restores an invariant that non-reserved EOA account balances can only decrease as a result of a transaction originating from that account.

In addition, the handling of account nonces during execution is separated from the verification of nonces during block verification, as specified below. This prevents transactions from being invalidated as a result of an EOA nonce increasing after transaction execution begins.

### Reserve balances

TODO

### Handling of nonces

To account for EOA account nonces being incremented during contract execution and potentially invalidating transactions from that EOA that have already been accepted, we separate the rules for how nonces are verified during block verification and handled during execution.

During block verification, all transactions must be verified to have a correct nonce value that based on the latest "settled" state root, as defined in ACP-194, and the number of transactions from the sender account in the queue pending execution. Specifically, the required nonce value is calculated by looking up the latest nonce for the sender account from the state root settled by the block being verified, and incrementing that value by one for each transaction from the sender account in the queue of transactions pending execution and prior transactions from the sender account in the block being verified. 

During execution, the nonce used must be one greater than latest nonce used by the account, accounting for both all transactions from the account and all contracts created by the account. This means that the actual nonce used by a transaction may differ from the nonce assigned in the raw transaction itself and used in verification.

Separating the nonce values used for block verification and execution ensures that transactions accepted in blocks cannot be invalidated by the execution of transactions before them in the pending execution queue. It still provides the same level of replay protection to transactions, as a transaction with a given nonce from an EOA can be accepted at most once. However, this separation has a subtle potential impact on contract creation. Previously, the resulting address of a contract could be deterministically derived from from a contract creation transaction based on its sender address and the nonce set in the transaction. Now, since the nonce used in execution is separate from that set in the transaction, this is no longer guaranteed. 

## Backwards Compatibility

The introduction of EIP-7702-like transactions will require a network upgrade to be scheduled.

Upon activation, a few invariants will be broken:
- (From EIP-7702) `tx.origin == msg.sender` can only be true in the topmost frame of execution.
    - Once an account has been delegated, it can invoke multiple calls per transaction.
- (From EIP-7702) An EOA nonce may not increase after transaction execution has begun.
    - Once an account has been delegated, the account may call a create operation during execution, causing the nonce to increase.
- The contract address of a contract deployed by an EOA with an empty "to" address can be derived from the sender address and the transaction's nonce.
    - If the nonce of the account increases due to execution of other transactions prior to the contract creation transaction, the nonce that it uses in execution may differ from that set in the raw transaction itself.
    - Note that this can only occur for accounts that have been delegated, and whose delegated code involves contract creation.

## Reference Implementation

A reference implementation is not yet available, and must be provided in order for this ACP to be considered implementable.

## Security Considerations

All of the [security considerations from the EIP-7702 specification](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-7702.md?ref=blockhead.co#security-considerations) apply here as well, except for the considerations regarding "sponsored transaction relayers" and "transaction propagation". Those two considerations do not apply here, as they are accounted for by the modifications made to introduce reserved balances and separate the handling of nonces in execution from verification.

## Open Questions

Optional section that lists any concerns that should be resolved prior to implementation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
