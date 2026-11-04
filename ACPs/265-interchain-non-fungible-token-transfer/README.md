| ACP           | 265                                                                                                                   |
| :------------ | :-------------------------------------------------------------------------------------------------------------------- |
| **Title**     | Interchain Non-Fungible Token Transfer (ICNFTT)                                                                       |
| **Author(s)** | Simon ([midnight-commit](https://github.com/midnight-commit)), YakMan ([snow-farmer](https://github.com/snow-farmer)) |
| **Status**    | Proposed                                                                                                              |
| **Track**     | Best Practices Track                                                                                                  |

## Abstract

This specification defines ICNFTT (Interchain Non-Fungible Token Transfer), a best practices standard for NFT (ERC721) transfers between Avalanche L1s using Interchain Messaging (ICM) and [Teleporter](https://github.com/ava-labs/icm-services/tree/main/icm-contracts/contracts/teleporter).

ICNFTT employs a **Hub-and-Spoke architecture** where Home contracts on canonical NFT chains manage token escrow and Remote contracts on destination chains mint representation tokens. The standard supports batch transfers for gas efficiency, composable `sendAndCall` operations for cross-chain interactions, and metadata preservation across chains.

Unlike fungible token bridges, ICNFTT addresses NFT-specific challenges including unique asset identity preservation, single-hop-only transfers (due to inability to pay relayer fees in non-fungible assets), and token location tracking to prevent duplication. The design ensures a single source of truth for token ownership while enabling seamless cross-chain transfers.

The specification provides complete interface definitions for implementers and a [reference implementation](https://github.com/tesseract-protocol/icnftt) for ERC721 tokens, enabling seamless NFT movement across the growing Avalanche L1 ecosystem while maintaining security and preventing double-spending.

## Motivation

The ability to transfer NFTs between Avalanche L1 networks is increasingly important as the ecosystem grows and specialized chains emerge for gaming, digital collectibles, metaverse applications, and other NFT-centric use cases. While [ICTT (Interchain Token Transfer)](https://github.com/ava-labs/icm-services/tree/main/icm-contracts/contracts/ictt) provides a robust standard for fungible token transfers, NFTs have unique requirements that necessitate a dedicated standard:

1. **Unique Asset Identity**: Unlike fungible tokens where any unit is interchangeable, each NFT has a unique identity that must be preserved across chains.

2. **Metadata Preservation**: NFTs carry metadata (token URIs, attributes, etc.) that must be transferred alongside the token to maintain its utility and value on destination chains.

3. **No Native Multi-Hop Fee Mechanism**: ICTT supports multi-hop transfers by paying fees in the transferred token itself. Since NFTs are non-fungible, there's no equivalent mechanism to pay relayer fees for intermediate hops.

4. **State Synchronization Complexity**: NFTs often have complex on-chain state (royalties, dynamic attributes, game state) that may need different synchronization strategies across chains.

This ACP establishes a standard for NFT bridging between Avalanche L1s, addressing these unique challenges while maintaining security and enabling seamless user experiences.

## Specification

### Architecture Overview

ICNFTT employs a **Hub-and-Spoke architecture** with Home and Remote contracts:

- **Home Contract**: Deployed on the chain where the canonical NFT collection exists. It acts as an adapter for existing ERC721 tokens, managing cross-chain transfers by holding tokens in escrow when they are sent to remote chains.

- **Remote Contract**: Deployed on destination chains where users want to access the NFTs. It mints representation tokens when NFTs arrive from the Home chain and burns them when tokens return to Home.

This architecture ensures:

- A single source of truth for token ownership
- Prevention of double-spending across chains
- Tracking of each token's current location

### Token Transferrer Interface

We propose that smart contracts performing NFT transfers between different Avalanche L1s implement the following interface.

The token transferrer uses a **single-hop only** model, meaning transfers can only occur between Home and Remote contracts directly. This design choice is made because:

1. NFTs cannot be used to pay for relayer fees on intermediate chains
2. Single-hop transfers reduce complexity and potential failure points
3. Token location tracking is simplified with direct Home-Remote relationships

#### Input Structures

When sending NFTs cross-chain, users must provide the following inputs:

```solidity
struct SendTokenInput {
    bytes32 destinationBlockchainID;
    address destinationTokenTransferrerAddress;
    address recipient;
    address primaryFeeTokenAddress;
    uint256 primaryFee;
    uint256 requiredGasLimit;
}
```

| Field                                | Description                                                               |
| ------------------------------------ | ------------------------------------------------------------------------- |
| `destinationBlockchainID`            | The blockchain ID of the destination chain                                |
| `destinationTokenTransferrerAddress` | The address of the token contract on the destination chain                |
| `recipient`                          | The address that will receive the token(s) on the destination chain       |
| `primaryFeeTokenAddress`             | The address of the ERC20 token used to pay for the Teleporter message fee |
| `primaryFee`                         | The amount of fee tokens to pay for relaying                              |
| `requiredGasLimit`                   | The gas limit required for executing the message on the destination chain |

### Methods

The core interface for cross-chain NFT transfers:

```solidity
interface IERC721Transferrer is ITeleporterReceiver {
    /**
     * @notice Sends ERC721 tokens to the specified destination.
     * @param input Specifies information for delivery of the tokens
     * @param tokenIds Array of token IDs to send
     */
    function send(SendTokenInput calldata input, uint256[] calldata tokenIds) external;

    /**
     * @notice Sends ERC721 tokens to another chain and triggers a contract call.
     * @param input Specifies information for delivery and the contract call
     * @param tokenIds Array of token IDs to send
     */
    function sendAndCall(SendAndCallInput calldata input, uint256[] calldata tokenIds) external;

    /**
     * @notice Returns the blockchain ID that the transferrer is deployed on.
     * @return The blockchain ID
     */
    function getBlockchainID() external view returns (bytes32);
}
```

### Batch Transfers

Unlike fungible tokens where amounts can be aggregated, NFTs must be transferred individually by their token IDs. ICNFTT supports **batch transfers** of multiple NFTs in a single transaction by accepting an array of token IDs. This reduces gas costs and provides a better user experience when transferring multiple items.

### Send and Call

To enable composable cross-chain interactions, ICNFTT supports a `sendAndCall` method that transfers NFTs and executes a contract call on the destination chain in a single operation. This enables use cases such as:

- Listing NFTs on a remote chain marketplace directly from the home chain
- Staking NFTs in a remote chain protocol
- Using NFTs as collateral in remote chain DeFi applications

```solidity
struct SendAndCallInput {
    bytes32 destinationBlockchainID;
    address destinationTokenTransferrerAddress;
    address recipientContract;
    address fallbackRecipient;
    bytes recipientPayload;
    uint256 recipientGasLimit;
    address primaryFeeTokenAddress;
    uint256 primaryFee;
    uint256 requiredGasLimit;
}
```

The recipient contract must implement:

```solidity
interface IERC721SendAndCallReceiver {
    /**
     * @notice Called by the token contract when tokens are sent via sendAndCall.
     * @dev To take ownership of tokens, the recipient MUST call transferFrom
     * to move each token from the calling contract to itself. Tokens not
     * transferred will be sent to the fallback recipient.
     *
     * @param sourceBlockchainID The blockchain ID the tokens were sent from
     * @param originTokenTransferrerAddress The token transferrer address on source chain
     * @param originSenderAddress The sender address on the source chain
     * @param tokenAddress The ERC721 token contract address on this chain
     * @param tokenIds Array of token IDs being sent
     * @param payload Additional data provided by the caller
     */
    function receiveTokens(
        bytes32 sourceBlockchainID,
        address originTokenTransferrerAddress,
        address originSenderAddress,
        address tokenAddress,
        uint256[] calldata tokenIds,
        bytes calldata payload
    ) external;
}
```

### Message Types

The protocol uses typed messages to handle different operations:

```solidity
enum TransferrerMessageType {
    REGISTER_REMOTE,
    SINGLE_HOP_SEND,
    SINGLE_HOP_CALL
}

struct TransferrerMessage {
    TransferrerMessageType messageType;
    bytes payload;
}
```

For basic transfers:

```solidity
struct SendTokenMessage {
    address recipient;
    uint256[] tokenIds;
    bytes[] tokenMetadata;
}
```

For send-and-call operations:

```solidity
struct SendAndCallMessage {
    address originSenderAddress;
    address recipientContract;
    uint256[] tokenIds;
    bytes recipientPayload;
    uint256 recipientGasLimit;
    address fallbackRecipient;
    bytes[] tokenMetadata;
}
```

### Remote Registration

Before tokens can be transferred, Remote contracts must register with their corresponding Home contract. This registration process:

1. Ensures both contracts are aware of each other
2. Establishes a verified communication channel
3. Allows the Home contract to validate incoming messages

The registration flow:

1. Home contract owner sets the expected Remote contract address for a chain
2. Remote contract calls `registerWithHome()` to send a registration message
3. Home contract verifies the sender matches the expected address
4. Upon successful verification, the Remote is registered and transfers are enabled

```solidity
// On Home contract
function setExpectedRemoteContract(
    bytes32 remoteBlockchainID,
    address expectedRemoteAddress
) external;

// On Remote contract
function registerWithHome(TeleporterFeeInfo calldata feeInfo) external;
```

### Token Location Tracking

The Home contract maintains a mapping of each token's current location:

```solidity
mapping(uint256 tokenId => bytes32 blockchainID) internal _tokenLocation;
```

- When a token is on the Home chain, its location is `bytes32(0)`
- When sent to a Remote chain, the location is updated to that chain's blockchain ID
- When returning from a Remote, the location is reset to `bytes32(0)`

This tracking prevents tokens from being "duplicated" across chains and ensures tokens can only return from the chain they were sent to.

### Metadata Transfer

NFT metadata (such as token URIs) can be transferred alongside tokens to ensure consistency across chains. The Home contract implements:

```solidity
function _prepareTokenMetadata(
    uint256 tokenId,
    TransferrerMessageType messageType
) internal view virtual returns (bytes memory);
```

The Remote contract processes received metadata:

```solidity
function _processTokenMetadata(
    uint256 tokenId,
    bytes memory metadata
) internal virtual;
```

This extensible design allows implementations to transfer any metadata relevant to their use case, from simple token URIs to complex attribute structures.

#### Metadata Transfer Considerations

Implementations should carefully consider their metadata strategy:

**One-Time vs. Synchronized Metadata**

Metadata is transferred at the time of the cross-chain transfer and is **not automatically synchronized** afterward. If metadata changes on the Home chain while a token is on a Remote chain:

- The Remote chain will retain the metadata that was transferred at send time
- The updated metadata will only be reflected on the Remote after a round-trip (Remote → Home → Remote)
- Implementations requiring real-time metadata synchronization should consider additional cross-chain messaging patterns

**Upgrade Considerations**

When designing metadata structures, implementations should consider forward compatibility. Remote contracts should handle missing or unrecognized metadata fields without reverting to ensure smooth upgrades.

### Home Contract Interface

```solidity
interface IERC721TokenHome is IERC721Transferrer {
    // Events
    event ERC721TokenHomeInitialized(address indexed token);
    event RemoteChainRegistered(bytes32 indexed blockchainID, address indexed remote);
    event RemoteChainExpectedContractSet(bytes32 indexed blockchainID, address indexed expectedRemote);
    event TokenLocationUpdated(uint256 indexed tokenId, bytes32 indexed destinationBlockchainID);

    // Functions
    function setExpectedRemoteContract(bytes32 remoteBlockchainID, address expectedRemoteAddress) external;
    function getToken() external view returns (address);
    function getRegisteredChains() external view returns (bytes32[] memory);
    function getRegisteredChain(uint256 index) external view returns (bytes32);
    function getRegisteredChainsLength() external view returns (uint256);
    function getRemoteContract(bytes32 remoteBlockchainID) external view returns (address);
    function getTokenLocation(uint256 tokenId) external view returns (bytes32);
}
```

### Remote Contract Interface

```solidity
interface IERC721TokenRemote is IERC721Transferrer {
    // Events
    event ERC721TokenRemoteInitialized(bytes32 indexed homeBlockchainID, address indexed homeContractAddress);
    event TokenMinted(uint256 indexed tokenId, address indexed owner);
    event TokenBurned(uint256 indexed tokenId, address indexed owner);
    event HomeChainRegistered(bytes32 indexed chainId, address indexed homeAddress);
    event RegisterWithHome(bytes32 indexed teleporterMessageID, bytes32 indexed destinationBlockchainID, address indexed remote);

    // Functions
    function registerWithHome(TeleporterFeeInfo calldata feeInfo) external;
    function getHomeBlockchainID() external view returns (bytes32);
    function getHomeTokenAddress() external view returns (address);
    function getIsRegistered() external view returns (bool);
}
```

## Usage Examples

This section provides concrete examples demonstrating how to use ICNFTT for common scenarios.

### Example 1: Basic Single Token Transfer

Transfer a single NFT from the Home chain to a Remote chain.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC721TokenHome} from "@tesseract/icnftt/interfaces/IERC721TokenHome.sol";
import {SendTokenInput} from "@tesseract/icnftt/interfaces/IERC721Transferrer.sol";

contract BasicTransferExample {
    IERC721TokenHome public homeContract;
    IERC721 public nftContract;
    IERC20 public feeToken;

    // Remote chain configuration
    bytes32 public constant REMOTE_BLOCKCHAIN_ID = 0x898b8aa8353f2b79ee1de07c36474fcee339003d90fa06ea3a90d9e88b7d7c33;
    address public remoteContractAddress;

    constructor(
        address _homeContract,
        address _nftContract,
        address _feeToken,
        address _remoteContract
    ) {
        homeContract = IERC721TokenHome(_homeContract);
        nftContract = IERC721(_nftContract);
        feeToken = IERC20(_feeToken);
        remoteContractAddress = _remoteContract;
    }

    /**
     * @notice Sends a single NFT to the Remote chain
     * @param tokenId The ID of the token to transfer
     * @param recipient The address to receive the token on the Remote chain
     * @param fee The amount of fee tokens to pay for relaying
     */
    function sendToRemote(
        uint256 tokenId,
        address recipient,
        uint256 fee
    ) external {
        // 1. Transfer NFT to this contract (or approve homeContract directly)
        nftContract.transferFrom(msg.sender, address(this), tokenId);

        // 2. Approve the Home contract to transfer the NFT
        nftContract.approve(address(homeContract), tokenId);

        // 3. Approve fee tokens if paying a fee
        if (fee > 0) {
            feeToken.transferFrom(msg.sender, address(this), fee);
            feeToken.approve(address(homeContract), fee);
        }

        // 4. Prepare the send input
        SendTokenInput memory input = SendTokenInput({
            destinationBlockchainID: REMOTE_BLOCKCHAIN_ID,
            destinationTokenTransferrerAddress: remoteContractAddress,
            recipient: recipient,
            primaryFeeTokenAddress: address(feeToken),
            primaryFee: fee,
            requiredGasLimit: 300_000 // Adjust based on Remote contract complexity
        });

        // 5. Create token IDs array and send
        uint256[] memory tokenIds = new uint256[](1);
        tokenIds[0] = tokenId;

        homeContract.send(input, tokenIds);
    }
}
```

### Example 2: Batch Transfer

Transfer multiple NFTs in a single transaction to reduce gas costs.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC721TokenHome} from "@tesseract/icnftt/interfaces/IERC721TokenHome.sol";
import {SendTokenInput} from "@tesseract/icnftt/interfaces/IERC721Transferrer.sol";

contract BatchTransferExample {
    IERC721TokenHome public homeContract;
    IERC721 public nftContract;
    IERC20 public feeToken;

    bytes32 public remoteBlockchainID;
    address public remoteContractAddress;

    constructor(
        address _homeContract,
        address _nftContract,
        address _feeToken,
        bytes32 _remoteBlockchainID,
        address _remoteContract
    ) {
        homeContract = IERC721TokenHome(_homeContract);
        nftContract = IERC721(_nftContract);
        feeToken = IERC20(_feeToken);
        remoteBlockchainID = _remoteBlockchainID;
        remoteContractAddress = _remoteContract;
    }

    /**
     * @notice Sends multiple NFTs to the Remote chain in a single transaction
     * @param tokenIds Array of token IDs to transfer
     * @param recipient The address to receive the tokens on the Remote chain
     * @param fee The amount of fee tokens to pay for relaying
     */
    function sendBatchToRemote(
        uint256[] calldata tokenIds,
        address recipient,
        uint256 fee
    ) external {
        // 1. Transfer all NFTs to this contract and approve
        for (uint256 i = 0; i < tokenIds.length; i++) {
            nftContract.transferFrom(msg.sender, address(this), tokenIds[i]);
            nftContract.approve(address(homeContract), tokenIds[i]);
        }

        // 2. Handle fee token approval
        if (fee > 0) {
            feeToken.transferFrom(msg.sender, address(this), fee);
            feeToken.approve(address(homeContract), fee);
        }

        // 3. Prepare and execute the batch send
        // Note: Gas limit should scale with batch size
        uint256 gasLimitPerToken = 150_000;
        uint256 baseGasLimit = 100_000;
        uint256 totalGasLimit = baseGasLimit + (gasLimitPerToken * tokenIds.length);

        SendTokenInput memory input = SendTokenInput({
            destinationBlockchainID: remoteBlockchainID,
            destinationTokenTransferrerAddress: remoteContractAddress,
            recipient: recipient,
            primaryFeeTokenAddress: address(feeToken),
            primaryFee: fee,
            requiredGasLimit: totalGasLimit
        });

        homeContract.send(input, tokenIds);
    }
}
```

### Example 3: Returning Tokens to Home Chain

Transfer NFTs back from a Remote chain to the Home chain.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {IERC721TokenRemote} from "@tesseract/icnftt/interfaces/IERC721TokenRemote.sol";
import {SendTokenInput} from "@tesseract/icnftt/interfaces/IERC721Transferrer.sol";

contract ReturnToHomeExample {
    IERC721TokenRemote public remoteContract;
    IERC20 public feeToken;

    constructor(address _remoteContract, address _feeToken) {
        remoteContract = IERC721TokenRemote(_remoteContract);
        feeToken = IERC20(_feeToken);
    }

    /**
     * @notice Returns NFTs from Remote chain back to Home chain
     * @param tokenIds Array of token IDs to return
     * @param recipient The address to receive tokens on the Home chain
     * @param fee The relayer fee
     */
    function returnToHome(
        uint256[] calldata tokenIds,
        address recipient,
        uint256 fee
    ) external {
        // 1. Transfer NFTs to this contract and approve
        for (uint256 i = 0; i < tokenIds.length; i++) {
            // The Remote contract IS the NFT contract on this chain
            IERC721(address(remoteContract)).transferFrom(
                msg.sender,
                address(this),
                tokenIds[i]
            );
            IERC721(address(remoteContract)).approve(
                address(remoteContract),
                tokenIds[i]
            );
        }

        // 2. Handle fee
        if (fee > 0) {
            feeToken.transferFrom(msg.sender, address(this), fee);
            feeToken.approve(address(remoteContract), fee);
        }

        // 3. Get Home chain info from the Remote contract
        bytes32 homeBlockchainID = remoteContract.getHomeBlockchainID();
        address homeTokenAddress = remoteContract.getHomeTokenAddress();

        // 4. Send back to Home
        SendTokenInput memory input = SendTokenInput({
            destinationBlockchainID: homeBlockchainID,
            destinationTokenTransferrerAddress: homeTokenAddress,
            recipient: recipient,
            primaryFeeTokenAddress: address(feeToken),
            primaryFee: fee,
            requiredGasLimit: 300_000
        });

        remoteContract.send(input, tokenIds);
        // Note: Tokens are burned on the Remote chain and released from
        // escrow on the Home chain
    }
}
```

## Migration Guide

This section provides guidance for teams looking to enable cross-chain transfers for existing NFT collections.

### Overview

ICNFTT is designed to work with existing ERC721 contracts without requiring modifications. The Home contract acts as an adapter that:

- Does not require modifications to the original ERC721 contract
- Uses standard `transferFrom` to move tokens
- Preserves all original token functionality on the Home chain

### Step-by-Step Migration Process

#### Step 1: Assess Your Collection

Before implementing ICNFTT, evaluate your collection's requirements:

| Consideration         | Questions to Ask                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------ |
| **Metadata Strategy** | Does your collection use on-chain or off-chain metadata? Are token URIs static or dynamic? |
| **State Complexity**  | Does your NFT have additional on-chain state (royalties, game attributes, etc.)?           |
| **Permissions**       | Who can mint? Are there admin functions that need to work cross-chain?                     |
| **Integrations**      | Which marketplaces, games, or protocols integrate with your collection?                    |

#### Step 2: Deploy Home Contract

Create a Home contract that wraps your existing ERC721:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {ERC721TokenHome} from "@tesseract/icnftt/ERC721TokenHome.sol";
import {IERC721} from "@openzeppelin/contracts/token/ERC721/IERC721.sol";

/**
 * @title MyCollectionHome
 * @notice Home contract adapter for an existing ERC721 collection
 */
contract MyCollectionHome is ERC721TokenHome {
    constructor(
        address teleporterRegistryAddress,
        address teleporterManager,
        address existingNFTContract
    ) ERC721TokenHome(
        teleporterRegistryAddress,
        teleporterManager,
        existingNFTContract
    ) {}

    /**
     * @notice Override to customize metadata transfer
     * @dev This example transfers the token URI as metadata
     */
    function _prepareTokenMetadata(
        uint256 tokenId,
        TransferrerMessageType /* messageType */
    ) internal view override returns (bytes memory) {
        // Get the token URI from the original contract
        // Assumes the original contract has a tokenURI function
        try IERC721Metadata(token).tokenURI(tokenId) returns (string memory uri) {
            return abi.encode(uri);
        } catch {
            return "";
        }
    }
}

interface IERC721Metadata {
    function tokenURI(uint256 tokenId) external view returns (string memory);
}
```

#### Step 3: Deploy Remote Contract(s)

For each destination chain, deploy a Remote contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

import {ERC721TokenRemote} from "@tesseract/icnftt/ERC721TokenRemote.sol";
import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";

/**
 * @title MyCollectionRemote
 * @notice Remote representation of the collection on a destination chain
 */
contract MyCollectionRemote is ERC721TokenRemote {
    // Store token URIs received from Home chain
    mapping(uint256 => string) private _tokenURIs;

    constructor(
        address teleporterRegistryAddress,
        address teleporterManager,
        bytes32 homeBlockchainID,
        address homeContractAddress,
        string memory name,
        string memory symbol
    ) ERC721TokenRemote(
        teleporterRegistryAddress,
        teleporterManager,
        homeBlockchainID,
        homeContractAddress,
        name,
        symbol
    ) {}

    /**
     * @notice Override to process metadata received from Home chain
     */
    function _processTokenMetadata(
        uint256 tokenId,
        bytes memory metadata
    ) internal override {
        if (metadata.length > 0) {
            string memory uri = abi.decode(metadata, (string));
            _tokenURIs[tokenId] = uri;
        }
    }

    /**
     * @notice Returns the token URI
     */
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "Token does not exist");
        return _tokenURIs[tokenId];
    }

    function _exists(uint256 tokenId) internal view returns (bool) {
        return _ownerOf(tokenId) != address(0);
    }
}
```

#### Step 4: Register Remote Contracts

After deployment, register Remote contracts with the Home:

```solidity
// On Home chain (called by admin)
homeContract.setExpectedRemoteContract(
    REMOTE_BLOCKCHAIN_ID,
    remoteContractAddress
);

// On Remote chain
remoteContract.registerWithHome(
    TeleporterFeeInfo({
        feeTokenAddress: FEE_TOKEN,
        amount: REGISTRATION_FEE
    })
);
```

## State Synchronization Considerations

NFTs often have more complex state requirements than fungible tokens. ICNFTT acknowledges several state synchronization approaches:

| Model                  | Description                                                           | Use Case                         |
| ---------------------- | --------------------------------------------------------------------- | -------------------------------- |
| **Autonomous Remote**  | Remote NFTs operate independently with minimal synchronization        | Approvals, local-only operations |
| **Hub-Push Async**     | Home pushes state changes to Remotes without requiring acknowledgment | Admin operations like pause      |
| **Hub-Centered Sync**  | All operations forward to Home for validation                         | Operations requiring consensus   |
| **Acknowledge Change** | Two-phase commit for critical operations                              | High-security state changes      |

Implementations may mix approaches depending on the specific operation. The reference implementation uses **Autonomous Remote** for most operations and recommends **Hub-Push Async** for administrative functions.

## Reference Implementation

See the reference implementation on [GitHub](https://github.com/tesseract-protocol/icnftt).

The reference implementation includes:

- `ERC721TokenHome`: Abstract contract for adapting existing ERC721 tokens for cross-chain transfers
- `ERC721TokenRemote`: Abstract contract for minting representation tokens on remote chains
- Support for both `send` and `sendAndCall` operations
- Token URI metadata preservation across chains
- Batch transfer support for multiple NFTs
- Registration and validation mechanisms

## Security Considerations

### Token Duplication Prevention

The Home contract tracks each token's location to prevent duplication:

- Tokens can only be received back from the chain they were sent to
- The `_validateReceiveToken` function ensures the source chain and sender match the expected values

### Send and Call Failure Handling

When `sendAndCall` operations fail:

- Tokens that are not transferred by the recipient contract are sent to the `fallbackRecipient`, preventing tokens from being permanently locked if the recipient contract reverts or fails to take ownership
- The `fallbackRecipient` address should be validated to be capable of receiving ERC721 tokens

### Registration Security

The two-step registration process prevents unauthorized Remote contracts from registering:

1. The Home contract owner must explicitly set the expected Remote contract address
2. The Remote contract's registration message must come from the expected address
3. Each Remote can only be registered once per chain

### Reentrancy Protection

All transfer operations use reentrancy guards to prevent reentrancy attacks during:

- Token transfers to the contract
- Cross-chain message sending
- `sendAndCall` recipient contract calls

### Message Delivery

Message delivery guarantees are provided by Teleporter:

- Users can relay their own messages if relayers do not process them
- Messages include replay protection
- Failed message delivery can be retried

### Multi-Hop Limitation

ICNFTT intentionally does not support multi-hop transfers (Remote to Remote). This is because:

- NFTs cannot be used to pay relayer fees for intermediate hops
- Direct Home-Remote transfers simplify security analysis
- Token location tracking becomes complex with multi-hop

For transfers between two Remote chains, users should:

1. Transfer from Remote A back to Home
2. Transfer from Home to Remote B

## Backwards Compatibility

ICNFTT is designed to work with existing ERC721 token contracts. The Home contract acts as an adapter that:

- Does not require modifications to the original ERC721 contract
- Uses standard `transferFrom` to move tokens
- Preserves all original token functionality on the Home chain

## ERC1155 Support

This ACP focuses specifically on ERC721 tokens. ERC1155 multi-token standard support is not included in this specification but could be addressed in a future ACP defining `ERC1155TokenHome` and `ERC1155TokenRemote` contracts following similar patterns, with modifications to handle semi-fungible tokens and batch transfers of multiple token types.

The core architecture and security model of ICNFTT are designed to be adaptable to ERC1155 with appropriate interface changes.

## Acknowledgements

Thanks to the [ICTT (Interchain Token Transfer)](https://github.com/ava-labs/icm-services/tree/main/icm-contracts/contracts/ictt) team for the foundational patterns that inspired ICNFTT.

Thanks to the Avalanche community for feedback and contributions.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
