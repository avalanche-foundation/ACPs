| ACP | 209 |
| :--- | :--- |
| **Title** | EIP-7702-style Set Code for EOAs |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Arran Schlosberg ([@ARR4N](https://github.com/ARR4N)), Aaron Buchwald ([@aaronbuchwald](https://github.com/aaronbuchwald)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/216)) |
| **Track** | Standards |

## Abstract

[EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md) was activated on the Ethereum mainnet in May 2025 as part of the Pectra upgrade, and introduced a new "set code transaction" type that allows Externally Owned Accounts (EOAs) to set the code in their account. This enabled several UX improvements, including batching multiple operations into a single atomic transaction, sponsoring transactions on behalf of another account, and privilege de-escalation for EOAs.

This ACP proposes adding a similar transaction type and functionality to Avalanche EVM implementations in order to have them support the same style of UX available on Ethereum. Transactions that are invalidated between their acceptance and their execution under the Continuous Execution mechanism proposed in [ACP-194](../194-continuous-execution/README.md) are not executed; instead, they fail and pay a penalty equivalent to the ACP-194 minimum gas, which delegated accounts guarantee by maintaining a reserved balance.

## Motivation

The motivation for this ACP is the same as the motivation described in [EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#motivation). However, EIP-7702 as implemented for Ethereum breaks invariants required for EVM chains that use the ACP-194 Continuous Execution mechanism.

Given that ACP-194 is in production, bringing EIP-7702-style functionality to Avalanche EVMs requires modifications to preserve its necessary guarantees, described below.

### Invariants needed for ACP-194

There are [two invariants explicitly broken by EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#backwards-compatibility) that are relied upon by Continuous Execution. They are that (1) an EOA nonce increase; and (2) an EOA balance decrease; can only occur as a result of a transaction originating from the EOA in question.


These invariants allow Continuous Execution to statically analyze (i.e. determine without executing the transaction) that a transaction:

- Has the proper nonce; and
- Will have sufficient balance to pay for its worst-case transaction fee plus the balance it sends.

As described in ACP-194, this lightweight analysis of transactions allows blocks to be accepted by consensus with the guarantee that they can be executed successfully. Only after block acceptance are the transactions within the block then put into a queue to be executed asynchronously.

If the execution of other accepted but not-yet-executed transactions can then decrease an EOA's account balance or change an EOA's current nonce, then block verification is unable to ensure that transactions in the block will be valid when executed. This poses DoS vulnerabilities because the invalidated transactions consume space in the pending execution queue but they do not pay any fees.

Notably, EIP-7702's violation of these invariants already presents challenges for mempool verification on Ethereum. As [noted in the security considerations section](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#transaction-propagation), EIP-7702 makes it "possible to cause transactions from other accounts to become stale" and this "poses some challenges for transaction propagation" because nodes now cannot "statically determine the validity of transactions for that account". However in synchronous execution environments such as Ethereum, these issues only pose potential DoS risks to the public mempool, as stale transactions are never included in blocks. Under an asynchronous execution scheme, the issues pose DoS risks to the chain itself since the invalidated transactions can be included in blocks prior to their execution.

### Guiding principle

The behaviour of EIP-7702 under this ACP SHOULD differ from that on Ethereum as little as possible. On Ethereum, a transaction that has been made stale is simply never included. Under Continuous Execution, such a transaction has already been accepted by the time it becomes stale, so it can't be excluded. The _only_ difference in behaviour is therefore that a stale transaction is treated as an included-but-failed transaction: it pays a penalty and produces a failed receipt, but has no other effect. In particular, it is not executed under different conditions to those that the sender signed.

#### Resulting user experience

For delegated EOAs that never send their own transactions, user experience is identical to EIP-7702. For those that do but are "well-behaved" in that they never send concurrently with calls to themselves that result in inclusion of stale transactions, only a trivial reserve balance is required for DoS prevention.

## Specification

The same [set code transaction as specified in EIP-7702](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#set-code-transaction) will be added to Avalanche EVM implementations. The behavior of the transaction is the same as specified in EIP-7702, as is the behavior of calls _to_ a delegated EOA.

The behavior of calls _from_ a delegated EOA acting as `tx.origin` is, however, modified to keep the guarantee that every transaction accepted in a block pays for the queue space it consumes. The following high-level modifications are made to the transaction verification and execution rules:

1. A transaction that is invalid at the time of its execution is not executed. Instead, it is charged a penalty and produces a failed receipt.
1. Delegated accounts MUST maintain a "reserved balance" that is sufficient to pay the potential penalties of all of their transactions in the pending execution queue. The reserved balances are managed via a new `ReservedBalanceManager` precompile.

### Invalidated transactions

At the time of execution, a transaction is said to be _invalidated_ if either:
- Its nonce is less than the sender account's current nonce; or
- The sender account's regular (i.e. not reserved) balance is less than `gas_limit * max_fee_per_gas + value`; i.e. the upfront cost required for transaction validity in the EVM.

All other validity conditions are guaranteed by block verification, as specified in ACP-194.

An invalidated transaction:
- MUST NOT be executed. Its value is not transferred, no code is run, and its authorization list (if any) is not applied.
- MUST be charged a penalty of $g_P$ gas at the transaction's effective gas price, as defined below.
- MUST update the sender's nonce to `max(nonce, tx.nonce + 1)`, as specified [below](#handling-of-nonces).
- MUST produce a receipt with a failed status, `gasUsed` equal to $g_P$, and no logs.

The penalty gas is defined as:

$$
g_P := \max\left(g_0, \left\lceil \frac{g_L}{\lambda} \right\rceil \right)
$$

where $g_0$ is the transaction's intrinsic gas as defined in the Ethereum Yellow Paper, $g_L$ is its gas limit, and $\lambda$ is as defined in ACP-194. For the purposes of ACP-194, the gas charged by an invalidated transaction is $g_C := g_P$, which satisfies the existing lower bound $g_C \ge \frac{g_L}{\lambda}$.

The penalty is deducted from the sender's regular balance first, with any shortfall deducted from its reserved balance. Block verification guarantees that the sum of the two is sufficient (see [below](#transaction-verification)). The penalty is otherwise handled in the same way as the fees of any other transaction.

### Handling of nonces

During block verification, the nonce rules of ACP-194 are unchanged. All transactions must be verified to have a correct nonce value based on the latest "[settled](../194-continuous-execution/README.md#settling-blocks)" state root and the number of transactions from the sender account in the pending execution queue. Specifically, the required nonce is derived from the settled state root and incremented by one for each of the sender's transactions already accepted into the pending execution queue or current block.

During execution, a delegated account's nonce may also be incremented by contract creation performed by its delegated code. This can cause a transaction already in the queue to have a nonce lower than the account's current nonce, invalidating it.

When a transaction is invalidated, the sender's nonce is updated to `max(nonce, tx.nonce + 1)`. This has two cases:
- If the transaction was invalidated due to a stale nonce, the account's nonce has already been consumed and is unchanged.
- If the transaction was invalidated due to insufficient balance, the account's nonce is incremented, exactly as for any included transaction on Ethereum that fails (e.g. by reverting or running out of gas). Without this, all of the account's subsequent transactions in the queue would also be invalidated, each paying a penalty.

In both cases, each accepted transaction consumes exactly one nonce, which is
the assumption made by block verification, even if one or more of them is also
consumed by delegated contract creation. Replay protection is unaffected because
a transaction with a given nonce from an account can be accepted at most once.

### Reserved balances

To guarantee that delegated accounts can always pay the penalty for any of their transactions that are invalidated, a "reserved balance" mechanism is introduced for accounts. A reserved balance is used _only_ to pay penalties. It is never available to code execution, nor to pay the gas or value of a transaction that is not invalidated, which are paid exclusively from the regular balance. This simplifies implementation as the execution of non-invalidated transactions is unchanged.

To allow for managing reserved balances, a new `ReservedBalanceManager` stateful precompile will be added at address `0x0200000000000000000000000000000000000006`. The `ReservedBalanceManager` precompile will have the following interface:

```solidity
interface IReservedBalanceManager {
    /// @dev Emitted whenever an account's reserved balance is modified.
    event ReservedBalanceUpdated(address indexed account, uint256 newBalance);

    /// @dev Called to deposit the native token balance provided into the account's
    /// reserved balance.
    function depositReservedBalance(address account) external payable;

    /// @dev Moves the caller's entire reserved balance into its regular balance.
    /// MUST revert unless called directly by a transaction, i.e. the transaction's
    /// origin is the caller and its `to` address is this precompile.
    function withdrawAll() external;

    /// @dev Returns the current reserved balance for the given account.
    function getReservedBalance(address account) external view returns (uint256 balance);
}
```

The precompile will maintain a mapping of accounts to their current reserved balances. Any account MAY deposit into the reserved balance of any account, but only the account itself MAY withdraw from its own reserved balance, and only via a transaction sent directly to the precompile.

> [!IMPORTANT]
> The reserved balance is only useful if others can't reduce it, so it MUST only decrease due to transactions sent by the account itself. If `withdrawAll()` accepted delegated calls, any transaction could empty a reserve, which ACP-194 worst-case validation cannot detect.

### Transaction verification

For the purposes of verification, an account is treated as _delegated_ if either:
- It is delegated in the settled state; or
- An authorization signed by the account appears in any transaction in the pending execution queue, or earlier in the current block, regardless of whether that authorization delegates or revokes delegation.

A queued revocation is ignored because it is not guaranteed to be applied: the transaction containing it may be invalidated at execution, or the authorization's nonce may be stale. This is analogous to queued value transfers not increasing the receiver's worst-case balance under ACP-194. Always treating the account as delegated trades being overly conservative for a fixed period of $\tau$ seconds, in return for increased safety.

During transaction verification, the following rules are applied:
- If the sender is not treated as delegated, the rules of ACP-194 are unchanged; the transaction is confirmed to be able to pay for its worst case transaction fee and transferred balance by looking at the sender account's regular balance and accounting for prior transactions it has sent that are still in the pending execution queue.
- Otherwise, the transaction is valid only if the sender's worst-case reserved balance is sufficient to pay its worst-case penalty.

The worst-case reserved balance of an account is calculated by starting from its settled reserved balance and then, for each of its transactions in the pending execution queue or earlier in the current block that was verified as delegated, in order:
- Deducting the worst-case penalty, $g_P$ multiplied by the effective gas price at the worst-case base fee; and
- If the transaction is a call to `withdrawAll()` setting the worst-case reserved balance to zero. Such calls can be determined without execution.

Deposits into reserved balances made by transactions in the pending execution queue are not included until settled.

## Backwards Compatibility

The introduction of EIP-7702 transactions will require a network upgrade to be scheduled.

Upon activation, the following invariants will be broken:

- (From EIP-7702) `tx.origin == msg.sender` can only be true in the topmost frame of execution.
    - Once an account has been delegated, it can invoke multiple calls per transaction.
- (From EIP-7702) An EOA nonce may not increase after transaction execution has begun.
    - Once an account has been delegated, the account may call a create operation during execution, causing the nonce to increase. Any of the account's transactions in the queue whose nonce is consumed by such an operation will be invalidated.
- A transaction included in an accepted block is executed.
    - Transactions from delegated accounts may be invalidated, in which case they are charged a penalty without being executed.

Additionally, at all points after an account is treated as delegated, it MUST have sufficient reserved balance to cover the worst-case potential penalties of all of its transactions in the pending execution queue. Notably, this means that:

- If a delegated account has insufficient reserved balance, it will be unable to send any further transactions until it, or any other account, deposits reserved balance via the `ReservedBalanceManager` precompile, and that deposit is settled.
    - In order to send transactions immediately after delegating (i.e. before ACP-194 settlement), an account MUST deposit reserved balance via the `ReservedBalanceManager` precompile prior to sending a set code transaction.
- Reserved balance can only be returned to the regular balance with `withdrawAll()`, after which the account MUST deposit again before sending further transactions while delegated.

The `eth_getBalance` RPC and the `BALANCE` opcode return only the regular balance, as this is the only balance that can be spent. The `eth_call` RPC method allows for querying the reserved balance of a given account.

## Reference Implementation

A reference implementation is not yet available and must be provided for this ACP to be considered implementable.

## Security Considerations

All of the [security considerations from the EIP-7702 specification](https://github.com/ethereum/EIPs/blob/e17d216b4e8b359703ddfbc84499d592d65281fb/EIPS/eip-7702.md#security-considerations) apply here as well, except for the considerations regarding "sponsored transaction relayers" and "transaction propagation". Those two considerations do not apply here, as they are accounted for by the penalty charged to invalidated transactions and the reserved balances that guarantee it.

The penalty gas $g_P$ is bounded from below by the intrinsic gas to ensure that invalidated transactions pay for the block space they consume, and by $\frac{g_L}{\lambda}$ to ensure that they pay for the queue space they consume in the same way as any other transaction under ACP-194.

## Rationale

### Invalidation instead of modified execution

An earlier version of this ACP executed transactions even if their nonce or balance had been invalidated, by decoupling execution nonces from transaction nonces and requiring the reserved balance to cover each transaction's worst case fee _and_ transferred value. This had three drawbacks:

- A reserved balance large enough to cover transferred value could be very large; e.g. an account wishing to send a large amount of AVAX would first have to move it into its reserved balance.
- The address of a contract created by a transaction could no longer be derived from the sender's address and the transaction's nonce.
- Implementation required modifying the upstream `geth` implementation instead of being able to inspect and act on errors from the "outside".

Not executing invalidated transactions avoids all, and the remaining DoS vector against the queue is handled by the penalty, which mirrors the existing ACP-194 charge for reserving, but not using, queue space.

### Single withdrawal method

Only a `withdrawAll()` method is provided. A withdrawal of a fixed amount would require block verification to retain a worst-case penalty for the withdrawal transaction itself, which would, in general, leave "dust" in the reserved balance. Both block verification and execution set the reserved balance to zero under `withdrawAll()`, avoiding this. A partial withdrawal can be achieved by withdrawing everything and depositing the remainder.

### Size of reserved balances

As a rule of thumb, a delegated account needs to keep a reserved balance of at least half of the gas that it places in the queue, at the queue's worst-case gas price. More precisely, the reserved balance $B$ needed to cover a total queued gas limit of $G$ is:

$$
B := M \cdot K \cdot \left( \exp\left( \frac{G}{2 \cdot K} \right) - 1 \right)
$$

Consider the account's queued transactions in execution order, the $i$-th having gas limit $g_{L,i}$, where $g_{L,i} > \lambda g_{0,i}$ so its penalty is $g_{P,i} = \lceil g_{L,i}/\lambda \rceil$. Let $x_i$ be the worst-case gas excess immediately before the $i$-th transaction, starting from $x_0 = 0$. Each transaction pays its penalty at the worst-case gas price $M \cdot \exp\left( \frac{x_i}{K} \right)$, so the account needs a reserved balance of:

$$
\sum_i g_{P,i} \cdot M \cdot \exp\left( \frac{x_i}{K} \right)
$$

Under ACP-194, the worst-case excess grows by each transaction's gas limit, scaled by $\frac{R - T}{R}$. With $R = 2 \cdot T$ and $\lambda = 2$, both the penalty and the excess increase are half the gas limit:

$$
x_{i+1} - x_i = g_{L,i} \cdot \frac{R - T}{R} = \frac{g_{L,i}}{2} = g_{P,i}
$$

ignoring rounding. The required balance is therefore a left Riemann sum of the increasing function $M \cdot \exp\left( \frac{x}{K} \right)$, with steps of width $g_{P,i}$ from $x = 0$ to $x = \frac{G}{2}$. It is bounded above by the integral:

$$
\sum_i g_{P,i} \cdot M \cdot \exp\left( \frac{x_i}{K} \right) \le \int_0^{G/2} M \cdot \exp\left( \frac{x}{K} \right) dx = M \cdot K \cdot \left( \exp\left( \frac{G}{2 \cdot K} \right) - 1 \right) = B
$$

The bound ignores the decrease in excess as time passes, which makes it conservative. It also assumes no priority fees, each of which would add $g_{P,i} \cdot \text{tip}_i$ to the requirement.

If instead every transaction has $g_{L,i} = g_{0,i}$ (e.g. 21,000-gas transfers), then $g_{P,i} = g_{L,i} = 2 \cdot (x_{i+1} - x_i)$, so the bound doubles to $2 \cdot B$. Since $g_P \le g_L$ for all transactions, this is the maximum for any combination of transactions.

For the C-Chain, $K = 87 \cdot T$ and $\omega_Q = 2 \cdot \omega_B = 2 \cdot R \cdot \tau \cdot \lambda = 40 \cdot T$, so:

$$
\begin{align}
G = \omega_Q = 40 \cdot T &\implies B = 87 \cdot T \cdot M \cdot \left( \exp\left( \frac{20}{87} \right) - 1 \right) \approx 22.49 \cdot T \cdot M \\
G = \omega_Q + \omega_B = 60 \cdot T &\implies B = 87 \cdot T \cdot M \cdot \left( \exp\left( \frac{30}{87} \right) - 1 \right) \approx 35.82 \cdot T \cdot M
\end{align}
$$

With $T = 4{,}000{,}000$ gas/s and $M = 5$ nAVAX, $T \cdot M = 0.02$ AVAX, giving:

| $G$ (million gas) | $g_L > \lambda g_0$ | $g_L = g_0$ |
| - | - | - |
| $\omega_Q = 160$ | $B \approx 0.45$ | $B \approx 0.90$ |
| $\omega_Q + \omega_B = 240$ | $B \approx 0.72$ | $B \approx 1.43$ |


## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
