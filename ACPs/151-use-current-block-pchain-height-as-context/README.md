| ACP           | 151                                                                |
| :------------ | :----------------------------------------------------------------- |
| **Title**     | Use current block P-Chain height as context for state verification |
| **Author(s)** | Ian Suvak ([@iansuvak](https://github.com/iansuvak))               |
| **Status**    | Proposed                                                           |
| **Track**     | Standards                                                          |

## Abstract

Proposes that the ProposerVM passes inner VMs the P-Chain block height of the current block being built rather than the P-Chain block height of the parent block. Inner VMs use this P-Chain height for verifying aggregated signatures of Avalanche Interchain Messages (ICM). This will allow for a more reliable way to determine which validators should participate in signing the message, and remove unnecessary waiting periods.

## Motivation

Currently the ProposerVM passes the P-Chain height of the parent block to inner VMs, which use the value to verify ICM messages in the current block. Using the parent block's P-Chain height is necessary for verifying the proposer and reaching consensus on the current block, but it is not necessary for verifying ICM messages within the block.

Using the P-Chain height of the current block being built would make operations using ICM messages to modify the validator set, such as ones specified in [ACP-77](../77-reinventing-subnets/README.md) be verifiable sooner and more reliably. Currently at least two new P-Chain blocks need to be produced after the relevant state change for it to be reflected for purposes of ICM aggregate signature verification.

## Specification

The [block context](https://github.com/ava-labs/avalanchego/blob/d2e9d12ed2a1b6581b8fd414cbfb89a6cfa64551/snow/engine/snowman/block/block_context_vm.go#L14) contains a `PChainHeight` field that is passed from the ProposerVM to the inner VMs building the block. It is later used by the inner VMs to fetch the canonical validator set for verification of ICM aggregated signatures.

The `PChainHeight` currently passed in by the ProposerVM is the P-Chain height of the parent block. The proposed change is to instead have the ProposerVM pass in the P-Chain height of the current block.

## Backwards Compatibility

This change requires an upgrade to make sure that all validators verifying the validity of the ICM messages use the same P-Chain height and therefore the same validator set. Prior to activation nodes should continue to use P-Chain height of the parent block.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable". 

## Security Considerations

ProposerVM needs to use the parent block's P-Chain height to verify proposers for security reasons but we don't have such restrictions for verifying ICM message validity in the current block being built. Therefore, this should be a safe change.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
