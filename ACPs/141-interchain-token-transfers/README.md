| ACP           | 141                                                                      |
| :------------ | :----------------------------------------------------------------------- |
| **Title**     | Interchain Token Transfer                                                |
| **Author(s)** | Matthew Lam ([@minghinmatthewlam](https://github.com/minghinmatthewlam)) |
| **Status**    | Implementable                                                            |
| **Track**     | Best Practices Track                                                     |

## Abstract

Proposes a best practices standard for token transfers between Avalanche L1s using interchain messaing (ICM) and [Teleporter](https://github.com/ava-labs/teleporter).

## Motivation

Token transfer between Avalanche L1s is a common use case that dApps building on Avalanche have been looking for. Now that Teleporter, built on top of [Avalanche Warp Messaging](https://docs.avax.network/build/cross-chain/awm/overview), supports interchain messaging between Avalanche L1s, Teleporter can be used to faciliate token transfers between Avalanche L1s. However, token bridging is complex and has many edge cases to consider, especially since you're dealing with user's funds. This ACP aims to cover a standard for token bridging between Avalanche L1s using Teleporter, and the different considerations that need to be taken into account.

## Specification

### Token Transferrer Interface

We propose that smart contracts performing token transfers between different Avalanche L1s implement the following considerations into their interface.

The token transferrer contract can be used to transfer multiple source tokens to multiple destination tokens, or can be specified to handle only transfers for one source token. Both approaches have their trade offs. With the approach to transfer multiple source tokens, on destination, the token transferrer contract would need to deploy a new contract for each new source token, likely in the form a wrapped token. The token transferrer can keep track of existing wrapped tokens for specific destination chains, as well as the balances transferred to each. With the wrapped token approach though, the destination tokens wouldn't have custom implementations and features, limiting the functionality of the destination token representation. Security wise with this approach, if you allow any arbitrary ERC20 token to be bridged through the same token transferring contract, it'd be difficult and more prone to vulnerability to have a contract that works with any ERC20 token. With the approach to transfer only one source token in the token transferrer contract, the token bridged to each destination chain can have custom implementations and features, and also can tailor its security to that specific token. However, with the custom destination token representation, the token transferrer contract would need to deploy the contract on destination chain before bridging the token from source. This ACP focuses and advocates for the latter approach, where the token transferrer contract is specified to handle only transfers for one source token, prioritizing security and custom implementations for destination tokens.

Another high level consideration is whether the token transferrer contracts on different chains will have shared security. This means if one token transferrer contract on one chain is compromised, with numerous posible reasons like the custom implementation contract having a vulnerability or the chain being compromised, then it can affect the security of the token transferrer contracts on other chains.

For example there are token transferrer contracts on chains A, B, and C, that can directly transfer tokens between each other. If the token transferrer contract on chain A gets compromised, it can mint an extra amount of tokens on chain A to an attacker, and then be used to drain the corresponding tokens on chain B and C. To prevent this, there needs to be some tracking of the balance of tokens on each chain, and each token transfer needs to be checked against the balance that chain holds. This ACP explores the implementation of a "home" token transferrer contract, which will track the balances of tokens on each chain, and all other chains would be referred to as "remote" token transferrer contracts. In order to avoid the shared security across token transffer contracts on different chains, every token transfer needs to be checked against the balance on the home token transferrer. This means that transfers between two remote token transferrers need to be done in two steps, first transferring the tokens from the source remote token transferrer to the home token transferrer, and then transferring the tokens from the home token transferrer to the destination remote token transferrer. "Multi-hop" will be used to describe the two step process of transferring tokens between two remote token transferrers. "Single hop" will be used to describe the direct transfer of tokens between a remote token transferrer and the home token transferrer.

At a minimum, when sending tokens cross chain to a different L1, the token transferrer contract will at least need the following inputs from the user.

- Token address that is being transferred on source chain.
- Sender address that is transferring the tokens.
- Destination blockchain ID that the tokens are being transferred to.
- Token transferrer contract address being sent to.
- Destination address that the tokens are being transferred to.
- Amount of tokens being transferred.
- Fee amount for relaying the transaction.
- Fee token address
- Required gas limit for executing the transfer on destination chain, part of Teleporter's message input. See [here](https://github.com/ava-labs/teleporter/tree/main/contracts/teleporter#message-delivery-and-execution)

The token address will be set in the `constructor` for the token transferrer contract, and the sender address will be set by `msg.sender` when calling for a token transfer. Amount is specified differently for ERC20 and native tokens. For native tokens, the token transfer call will be `payable` and the `msg.value` specifies the transfer amount. For ERC20 tokens, a separate `amount` input will be needed. Rest of the inputs can be defined in a struct:

```solidity
struct SendTokensInput {
    bytes32 destinationBlockchainID;
    address destinationTokenTransferrerAddress;
    address feeTokenAddress;
    uint256 feeAmount;
    address recipient;
    uint256 requiredGasLimit;
}
```

### Methods

A method needed for the token transferrer contract is `send`, which users can call to send tokens to a different L1. The method will take in the `SendTokensInput` struct as input, and will also specify an amount, either directly as an input parameter or through `msg.value` if it's a native asset transfer. Another method needed is receiving from Teleporter, and the contract has to implement [ITeleporterReceiver](https://github.com/ava-labs/teleporter/blob/main/contracts/teleporter/ITeleporterReceiver.sol) to receive messages from Teleporter. The ERC20 and native token transferrer contract interface will look like:

```solidity
interface IERC20TokenTransferrer is ITeleporterReceiver {
    /**
     * @notice Sends ERC20 tokens to the specified destination.
     * @param input Specifies information for delivery of the tokens
     * @param amount Amount of tokens to send
     */
    function send(SendTokensInput calldata input, uint256 amount) external;
}
```

```solidity
interface INativeTokenTransferrer is ITeleporterReceiver {
    /**
     * @notice Sends native tokens to the specified destination.
     * @param input Specifies information for delivery of the tokens
     */
    function send(SendTokensInput calldata input) external payable;
}
```

#### Send and Call

A primary reason to transfer tokens is to access different dApps built on different L1s. With simple token transfers this would mean a few steps for a user:

1. Transfer tokens to the remote token transferrer contract.
2. Switch wallet to the remote chain.
3. Submit the tx on the remote chain to the dApp of interest.

To simplify this process, the token transferrer contract can implement a `sendAndCall` method, which will take the bridged tokens on remote chain and call a contract and function specified by the user. This way, the user can directly interact dApps on different chains all from one chain and tx. To support `sendAndCall`, tokens transferred over from `sendAndCall` will be approved to spend for the contract address the user wants to call, and will make a call to that contract address with a message payload that the user specifies. The required inputs include:

- recipient contract address to call
- recipient payload to call (inclues function signature and parameters)
- recipient gas limit

```solidity
struct SendAndCallInput {
    bytes32 destinationBlockchainID;
    address destinationTokenTransferrerAddress;
    address recipientContract;
    bytes recipientPayload;
    uint256 requiredGasLimit;
    uint256 recipientGasLimit;
    address feeTokenAddress;
    uint256 feeAmount;
}
```

In order to receive tokens from a `sendAndCall` method, there will be a separate interface the recipient contract needs to implement, because not having a standard interface for the recipient contract can increase the scope for possible vulnerabilities, and also that recipient contract might not be able to receive the tokens properly. The recipient contract will need to implement a `receiveTokens` method:

```solidity
interface IERC20SendAndCallReceiver {
    /**
     * @notice Called to receive the amount of the given token
     * @param sourceBlockchainID Blockchain ID that the transfer originated from
     * @param originTokenTransferrerAddress Address of the token transferrer that initiated the Teleporter message
     * @param originSenderAddress Address of the sender that sent the transfer. This value
     * should only be trusted if {originTokenTransferrerAddress} is verified and known.
     * @param token Address of the token to be received
     * @param amount Amount of the token to be received
     * @param payload Arbitrary data provided by the caller
     */
    function receiveTokens(
        bytes32 sourceBlockchainID,
        address originTokenTransferrerAddress,
        address originSenderAddress,
        address token,
        uint256 amount,
        bytes calldata payload
    ) external;
}
```

```solidity
interface INativeSendAndCallReceiver {
    /**
     * @notice Called to receive the amount of the native token. Implementations
     * must properly handle the msg.value of the call in order to ensure it doesn't
     * become improperly made inaccessible.
     * @param sourceBlockchainID Blockchain ID that the transfer originated from
     * @param originTokenTransferrerAddress Address of the token transferrer that initiated the Teleporter message
     * @param originSenderAddress Address of the sender that sent the transfer. This value
     * should only be trusted if {originTokenTransferrerAddress} is verified and known.
     * @param payload Arbitrary data provided by the caller
     */
    function receiveTokens(
        bytes32 sourceBlockchainID,
        address originTokenTransferrerAddress,
        address originSenderAddress,
        bytes calldata payload
    ) external payable;
}
```

### Multi-hop

Any time a remote token transferrer is called to send, it'll check whether the tokens are being transferred to another remote token transferrer, or back to the home. If it's to another remote token transferrer, it'll perform a transfer to the home contract first, and the home contract will check the balance before forwarding to the final destination remote token transferrer. To support this, home token transferrer contracts will need to keep a balance of the tokens transferred to each remote token transferrer in state.

To receive and correctly handle each of these use cases, the contract can denote transfer message types.

```solidity
enum TransferrerMessageType {
    SINGLE_HOP_SEND,
    SINGLE_HOP_CALL,
    MULTI_HOP_SEND,
    MULTI_HOP_CALL
}

struct TransferrerMessage {
    TransferrerMessageType messageType;
    bytes payload;
}
```

`payload` includes the encoded bytes of one of the above transferrer message types, for example single hop:

```solidity
struct SingleHopSendMessage {
    address recipient;
    uint256 amount;
}
```

## Reference Implementation

See reference implementation on [Github here](https://github.com/ava-labs/avalanche-interchain-token-transfer).

In addition to implementing the interface and design described above, the reference implementation includes audited contracts demonstrating any combination of erc20 and native token transfers.

## Security Considerations

With cross-chain token transfers there are different error cases in the transfer process to consider:

- If the multi-hop transfer fails to forward the token transfer to the final destination, the user's tokens would already be locked to the token transferrer. The ICTT reference implementation handles this with a `multiHopFallback` address that receives the tokens if the forwarding transfer fails.
- Message delivery guarantees are not given when users initiate the token transfer. The ICTT reference implementation uses Teleporter, which allows any sender to relay their own messages if needed. For example, if their message has no associated fees, and other relayers decide not to relay it, the sender can still relay the message themselves.
- `sendAndCall` can take in arbitrary call data and make lower level calls that can fail. The ICTT reference implementation takes a `fallbackRecipient` address, and if the `sendAndCall` fails, the tokens are sent to the fallback recipient.

## Acknowledgements

Thanks to @michaelkaplan13 @cam-schultz @geoff-vball @feuGeneA @bernard-avalabs for contribution and feedback on ICTT reference implementation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
