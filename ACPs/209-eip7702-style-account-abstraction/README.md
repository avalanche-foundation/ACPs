| ACP | 209 |
| :--- | :--- |
| **Title** | EIP-7702-style Set Code for EOAs |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Arran Schlosberg ([@ARR4N](https://github.com/ARR4N)), Aaron Buchwald ([@aaronbuchwald](https://github.com/aaronbuchwald)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/216)) |
| **Track** | Standards |

## Abstract

[EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md) was activated on the Ethereum mainnet in May 2025 as part of the Pectra upgrade, and introduced a new "set code transaction" type that allows Externally Owned Accounts (EOAs) to set the code in their account. This enabled several UX improvements, including batching multiple operations into a single atomic transaction, sponsoring transactions on behalf of another account, and privilege de-escalation for EOAs.

This ACP proposes adding a similar transaction type and functionality to Avalanche EVM implementations in order to have them support the same style of UX available on Ethereum. Modifications to the handling of account nonce and balances are required in order for it to be safe when used in conjunction with the streaming asynchronous execution (SAE) mechanism proposed in [ACP-194](https://github.com/avalanche-foundation/ACPs/tree/4a9408346ee408d0ab81050f42b9ac5ccae328bb/ACPs/194-streaming-asynchronous-execution).

## Motivation

The motivation for this ACP is the same as the motivation described in [EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#motivation). However, EIP-7702 as implemented for Ethereum breaks invariants required for EVM chains that use the ACP-194 SAE mechanism.

There has been strong community feedback in support of ACP-194 for its potential to:
- Allow for increasing the target gas rate of Avalanche EVM chains, including the C-Chain
- Enable the use of an encrypted mempool to prevent front-running
- Enable the use of real time VRF during transaction execution

Given the strong support for ACP-194, bringing EIP-7702-style functionality to Avalanche EVMs requires modifications to preserve its necessary invariants, described below.

### Invariants needed for ACP-194

There are [two invariants explicitly broken by EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#backwards-compatibility) that are required for SAE. They are:

1. An account balance can only decrease as a result of a transaction originating from that account.
1. An EOA nonce may not increase after transaction execution has begun.

These invariants are required for SAE in order to be able to statically analyze (i.e. determine without executing the transaction) that a transaction:
- Has the proper nonce
- Will have sufficient balance to pay for its worst case transaction fee plus the balance it sends

As described in the ACP-194, this lightweight analysis of transactions in blocks allows blocks to be accepted by consensus with the guarantee that they can be executed successfully. Only after block acceptance are the transactions within the block then put into a queue to be executed asynchronously. If the execution of transactions in the queue can decrease an EOA's account balance or change an EOA's current nonce, then block verification is unable to ensure that transactions in the block will be valid when executed. If transactions accepted into blocks can be invalidated prior to their execution, this poses DOS vulnerabilities because the invalidated transactions use up space in the pending execution queue according to their gas limits, but they do not pay any fees.

Notably, EIP-7702's violation of these invariants already presents challenges for mempool verification on Ethereum. As [noted in the security considerations section](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#transaction-propagation), EIP-7702 makes it "possible to cause transactions from other accounts to become stale" and this "poses some challenges for transaction propagation" because nodes now cannot "statically determine the validity of transactions for that account". In synchronous execution environments such as Ethereum, these issues only pose potential DOS risks to the public transaction mempool. Under an asynchronous execution scheme, the issues pose DOS risks to the chain itself since the invalidated transactions can be included in blocks prior to their execution.

## Specification

The same [set code transaction as specified in EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md?ref=blockhead.co#set-code-transaction) will be added to Avalanche EVM implementations. The behavior of the transaction is the same as specified in EIP-7702. However, in order to keep the guarantee of transaction validity upon inclusion in an accepted block, two modifications are made to the transaction verification and execution rules.
1. Delegated accounts must maintain a "reserved balance" to ensure they can always pay for the transaction fees and transferred balance of transactions sent from the account. The reserved balances are managed via a new `ReservedBalanceManager` precompile, as specified below.
1. The handling of account nonces during execution is separated from the verification of nonces during block verification, as specified below.

### Reserved balances

To ensure that all transactions can cover their worst case transaction fees and transferred balances upon inclusion in an accepted block, a "reserved balance" mechanism is introduced for accounts. Reserved balances are required for delegated accounts to guarantee that subsequent transactions they send after setting code for their account can still cover their fees and transfer amounts, even if transactions from other accounts reduce the account's balance prior to their execution.

To allow for managing reserved balances, a new `ReservedBalanceManager` stateful precompile will be added at address `0x0200000000000000000000000000000000000006`. The `ReservedBalanceManager` precompile will have the following interface:

```solidity
interface IReservedBalanceManager {
    /// @dev Emitted whenever an account's reserved balance is modified. 
    event ReservedBalanceUpdated(address indexed account, uint256 newBalance);

    /// @dev Called to deposit the native token balance provided into the account's
    /// reserved balance.
    function depositReservedBalance(address account) external payable;

    /// @dev Returns the current reserved balance for the given account.
    function getReservedBalance(address account) external view returns (uint256 balance);
}
```

The precompile will maintain a mapping of accounts to their current reserved balances. The precompile itself intentionally only allows for _increasing_ an account's reserved balance. Reducing an account's reserved balance is only ever done by the EVM when a transaction is sent from the account, as specified below.

During transaction verification, the following rules are applied:
- If the sender EOA account has not set code via an EIP-7702 transaction, no reserved balance is required.
    - The transaction is confirmed to be able to pay for its worst case transaction fee and transferred balance by looking at the sender account's regular balance and accounting for prior transactions it has sent that are still in the pending execution queue, as specified in ACP-194.
- Otherwise, if the sender EOA account has previously been delegated via an EIP-7702 transaction (even if that transaction is still in the pending execution queue), then the account's current "[settled](https://github.com/avalanche-foundation/ACPs/tree/4a9408346ee408d0ab81050f42b9ac5ccae328bb/ACPs/194-streaming-asynchronous-execution#settling-blocks)" reserved balance must be sufficient to cover the sum of the worst case transaction fees and balances sent for all of the transactions in the pending execution queue after the set code transaction.

During transaction execution, the following rules are applied:
- When initially deducting balance from the sender EOA account for the maximum transaction fee and balance sent with the transaction, the account's regular balance is used first. The account's reserved balance is only reduced if the regular balance is insufficient.
- In the execution of code as part of a transaction, only regular account balances are available. The only possible modification to reserved balances during code execution is increases via calls to the `ReservedBalanceManager` precompile `depositReservedBalance` function.
- If there is a gas refund at the end of the transaction execution, the balance is first credited to the sender account's reserved balance, up to a maximum of the account's reserved balance prior to the transaction. Any remaining refund is credited to the account's regular balance.

### Handling of nonces

To account for EOA account nonces being incremented during contract execution and potentially invalidating transactions from that EOA that have already been accepted, we separate the rules for how nonces are verified during block verification and how they are handled during execution.

During block verification, all transactions must be verified to have a correct nonce value based on the latest "settled" state root, as defined in ACP-194, and the number of transactions from the sender account in the pending execution queue. Specifically, the required nonce is derived from the settled state root and incremented by one for each of the sender’s transactions already accepted into the pending execution queue or current block.

During execution, the nonce used must be one greater than the latest nonce used by the account, accounting for both all transactions from the account and all contracts created by the account. This means that the actual nonce used by a transaction may differ from the nonce assigned in the raw transaction itself and used in verification.

Separating the nonce values used for block verification and execution ensures that transactions accepted in blocks cannot be invalidated by the execution of transactions before them in the pending execution queue. It still provides the same level of replay protection to transactions, as a transaction with a given nonce from an EOA can be accepted at most once. However, this separation has a subtle potential impact on contract creation. Previously, the resulting address of a contract could be deterministically derived from a contract creation transaction based on its sender address and the nonce set in the transaction. Now, since the nonce used in execution is separate from that set in the transaction, this is no longer guaranteed. 

## Backwards Compatibility

The introduction of EIP-7702 transactions will require a network upgrade to be scheduled.

Upon activation, a few invariants will be broken:
- (From EIP-7702) `tx.origin == msg.sender` can only be true in the topmost frame of execution.
    - Once an account has been delegated, it can invoke multiple calls per transaction.
- (From EIP-7702) An EOA nonce may not increase after transaction execution has begun.
    - Once an account has been delegated, the account may call a create operation during execution, causing the nonce to increase.
- The contract address of a contract deployed by an EOA (via transaction with an empty "to" address) can be derived from the sender address and the transaction's nonce.
    - If earlier transactions cause the nonce to increase before execution, the actual nonce used in a contract creation transaction may differ from the one in the transaction payload, altering the resulting contract address.
    - Note that this can only occur for accounts that have been delegated, and whose delegated code involves contract creation.

Additionally, at all points after the acceptance of a set code transaction, an EOA must have sufficient reserved balance to cover the sum of the worst case transactions fees and balances sent for all transactions in the pending execution queue after the set code transaction. Notably, this means that:
- If a delegated account has zero reserved balance at any point, it will be unable to send any further transactions until a different account provides it with reserved balance via the `ReservedBalanceManager` precompile. 
    - In order to initially "self-fund" its own reserved balance, an account must deposit reserved balance via the `ReservedBalanceManager` precompile prior to sending a set code transaction.
- In order to transfer its full (regular + reserved) account balance, a delegated account must first deposit all of its regular balance into reserved balance.

In order to support wallets as seamlessly as possible, the `eth_getBalance` RPC implementations should be updated to return the sum of an accounts regular and reserved balances. Additionally, clients should provide a new `eth_getReservedBalance` RPC method to allow for querying the reserved balance of a given account.

## Reference Implementation

A reference implementation is not yet available and must be provided for this ACP to be considered implementable.

## Security Considerations

All of the [security considerations from the EIP-7702 specification](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md?ref=blockhead.co#security-considerations) apply here as well, except for the considerations regarding "sponsored transaction relayers" and "transaction propagation". Those two considerations do not apply here, as they are accounted for by the modifications made to introduce reserved balances and separate the handling of nonces in execution from verification.

Additionally, given that an account's reserved balance may need be updated in state when a transfer is sent from the account it must be confirmed that 21,000 gas is still a sufficiently high cost for the potential more expensive operation. Charging more gas for basic transfer transactions in this case could otherwise be an option, but would likely cause further backwards compatibility issues for smart contracts and off-chain services.

## Open Questions

1. Are the implementation and UX complexities regarding the `ReservedBalanceManager` precompile worth the UX improvements introduced by the new set code transaction type?
   - Except for having a contract spend an account's native token balance, most, if not all, of the UX improvements associated with the new transaction type could theoretically be implemented at the contract layer rather than the protocol layer. However, not all contracts provide support for account abstraction functionality via standards such as [ERC-2771](https://eips.ethereum.org/EIPS/eip-2771).
2. Are the implementation and UX complexities regarding the `ReservedBalanceManager` precompile worth giving delegate contracts the ability to spend native token balances?
   - An alternative may be to disallow delegate contracts from spending native token balances at all, and revert if they attempt to. They could use "wrapped native token" ERC20 implementations (i.e. WAVAX) to achieve the same effect. However, this may be equally or more complex at the implementation level, and would cause incompatibilies in delegate contract implementations for Ethereum.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
