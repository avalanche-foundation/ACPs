| ACP | XXX |
| :- | :- |
| **Title** | X-Chain Removal |
| **Author(s)** | Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

Proposes the deprecation and end-of-life of the X-Chain. All AVAX held on the X-Chain would be made available to its existing owners as UTXOs on the P-Chain.

## Motivation

The X-Chain was initially created at the inception of Avalanche to demonstrate the speed and flexibility of the Avalanche consensus mechanism. It runs the Avalanche Virtual Machine (AVM), which was the first implementation of a custom virtual machine for Avalanche. The AVM does not support programmability, but it does support specific actions such the creation of new fungible and non-fungible tokens (ANTs), and operations on them such as mints and transfers. 

At the time of the launch of Avalanche Mainnet, the original vision was for these ANTs on the X-Chain to be a key piece of the broader Avalanche ecosystem by allowing them to be traded on DEXes, moved to the P-Chain to be used to validate new Avlanache Subnets (now L1s), and more. In the 5+ years since that time, the Avalanche ecosystem has been largely successful in realizing the effects of that vision. There are many tokens and applications launched on Avalanche in novel ways, and Avalanche L1s can define their own validation mechanisms, including the staking of arbitrary new tokens. However, the path to realize this vision ultimately did not involve any ANT assets created on the X-Chain.

Today, the X-Chain is barely used at all. In the past year, there have been in total less than XXX transactions on the X-Chain, and XXX% of them were `BaseTx`s performing basic AVAX transfers, which is an operation possible on both the C-Chain and the P-Chain as well. Despite this lack of usage, significant development effort would need to be put into the X-Chain in order to support items such as state sync and dynamic fees. Given that the it is serving little purpose or benefit to the Avalanche ecosystem, it will be best to focus all future efforts on continuing to enable and push novel use case development on Avalanche, rather than maintaining the X-Chain.

The X-Chain can be safely halted and removed from the Primary Network without any permanent loss of AVAX. Any AVAX remaining on the X-Chain can be made available to their existing owners on the P-Chain in a future network upgrade, as specified below.

## Specification

This ACP would need to be activated in steps across two required network upgrades, referred to as upgrade $A$ and upgrade $B$. Upgrade $B$ must come after after upgrade $A$, but they do not necessarily need to be consecutive network upgrades.

As part of the process, support for ANTs created on the X-Chain will be discontinued. However once upgrade $B$ is completed, **all AVAX held on the X-Chain at the time of its termination will be made available to its existing owners on the P-Chain**.

### X-Chain Deprecation and EOL Process

#### 1. Depreaction Notice (prior to upgrade $A$)

Should this ACP have the necessary support to move forward, a planned X-Chain deprecation notice will be published and communicated to broader Avalanche ecosystem. This will provide the entire ecosystem an extended window of time to move any AVAX they hold on the X-Chain to either the P- or C-Chains prior to the X-Chain's termination.

#### 2. Upgrade $A$

Upon activation of network upgrade $A$, the final block on the X-Chain will be accepted. Specifically, any X-Chain block whose parent timestamp is at or past the activation timestamp of upgrade $A$ will be considered invalid. This ensures that no further blocks will be accepted on the chain.

Additionally, as of the activation timestamp of upgrade $A$ `ExportTx`s on the P- and C-Chains will be considered invalid if their destination blockchain ID is the X-Chain blockchain ID. This prevents any accidental attempts to move funds to the X-Chain from locking funds in the future.

As of the activation timestamp of upgrade $A$, support for ANTs is officially removed.

Additionally, any AVAX that is either
1. Held in UTXOs on the X-Chain
1. Exported from the P- or C-Chain's to be moved to the X-Chain

will be temporarily frozen for the duration of time between the activation of upgrade $A$ and upgrade $B$.

#### 3. Upgrade $B$

After the X-Chain is halted and `ExportTx`s to the X-Chain discontinued upon the activation of upgrade $A$, it is possible to determine:
- The full set of [SECP256K1 transfer outputs](https://build.avax.network/docs/rpcs/x-chain/txn-format#secp256k1-transfer-output) that hold AVAX on the X-Chain.
- The full set of [SECP256K1 transfer outputs](https://build.avax.network/docs/rpcs/x-chain/txn-format#secp256k1-transfer-output) from [`ExportTx`s](https://build.avax.network/docs/rpcs/x-chain/txn-format#unsigned-exporttx) on the P- and C-Chains that hold AVAX and have the X-Chain blockchain ID as their `destination_chain`.

The above represents the full set of AVAX UTXOs that will permanently frozen after activation of network upgrade $A$. These sets can be computed by anyone by replaying the X-, P-, and C-Chain transactions up through the activation of upgrade $A$, and tracking the repspective UTXO sets present in the current state. 

Upon activation of network upgrade $B$, identical versions of these UTXOs will be created on the P-Chain, which already supports the same exact [SECP256K1 transfer output](https://build.avax.network/docs/rpcs/p-chain/txn-format#secp256k1-transfer-output) format. This means that the UTXOs will have the same `type_id`, `amount`, `locktime`, `threshold`, and `addresses` as they did on the X-Chain. These UTXOs can be explicitly specified on the P-Chain with the resulting sets determined from re-executing the chains as described above.

## Backwards Compatibility

This ACP would be activated as a part of 2 required network upgrades.

After the first network upgrade, all support for ANTs minted on the X-Chain would be removed, and all AVAX held on the X-Chain would be temporarily frozen. Anyone with X-Chain integrations would need to move their AVAX to the P- or C-Chain's prior to the first network upgrade to prevent any distruption of service.

All AVAX frozen on the X-Chain would be again be made available to the identical holders on the P-Chain as of the activation of the second network upgrade.

## Reference Implementation

A reference implementation is not yet available and must be provided for this ACP to be considered implementable.

## Security Considerations

Removal of the X-Chain from the Primary Network should serve as an overall security improvement to the Avalanche Ecosystem as it significantly reduces the possible attack surface and the amount of code needed to be properly maintained for the network to remain secure and available.

That being said, it should also be considered that:
1. The calculation of the full set of AVAX UTXOs to be made available on the P-Chain as part of the second network upgrade must be done with extreme care, and made publicly verifiable by anyone before activation.
1. There will be a temporary loss of AVAX liquidity between the first and second network upgrades. This should be reduced as much as possible with a well communicated deprecation notice as far in advance as possible of the first network upgrade, allowing all Avalanche ecosystem partcipants a long window of time to move their AVAX off of the X-Chain in preparation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
