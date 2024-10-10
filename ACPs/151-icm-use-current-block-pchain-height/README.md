| ACP           | 151                                                             |
| :------------ | :-------------------------------------------------------------- |
| **Title**     | Use current block P-Chain height for ICM signature verification |
| **Author(s)** | Ian Suvak ([@iansuvak](https://github.com/iansuvak))            |
| **Status**    | Proposed                                                        |
| **Track**     | Standards                                                       |

## Abstract

Proposes that VMs use the P-Chain height of the current block being built instead of the parent block when verifying aggregated signatures of Avalanche Interchain Messages (ICM). This will allow for a more reliable way to determine which validators should participate in signing the message, and remove unnecessary waiting periods.

## Motivation

Currently ICM messages use P-Chain height of the parent block when verifying messages for inclusion in the current block. Using the parent's P-Chain height is necessary for safe block building purposes but not for ICM message verification. 

Using the P-Chain height of the current block being built would make operations using ICM messages to modify the validator set, such as ones specified in [ACP-77](../77-reinventing-subnets/README.md) be verifiable sooner and more reliably. Currently at least two new P-Chain blocks need to be produced after the relevant state change for it to be reflected for purposes of ICM aggregate signature verification.

## Specification

[Block Context](https://github.com/ava-labs/avalanchego/blob/d2e9d12ed2a1b6581b8fd414cbfb89a6cfa64551/snow/engine/snowman/block/block_context_vm.go#L14) containing `PChainHeight` field is being passed to the VMs building the inner block, and is later being used by the VMs to fetch the canonical validator set for verification of ICM aggregated signatures. 

The height currently passed in is the P-Chain height of the parent block which is the same height used by the ProposerVM to verify the proposer. This is necessary for proposer verification but not for verifying the state of this block so should be populated with the P-Chain height of the block being currently built instead. 

## Backwards Compatibility

This change requires an upgrade to make sure that all validators verifying the validity of the ICM messages use the same P-Chain height and therefore the same validator set. Prior to activation nodes should continue to use P-Chain height of the parent block.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable". 

## Security Considerations

ProposerVM needs to use the parent block's P-Chain height to verify proposers for security reasons but we don't have such restrictions for verifying ICM message validity in the current block being built, therefore this should be a safe change.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
