```text
ACP: 113
Title: Provable Virtual Machine Randomness
Author(s): Tsachi Herman <http://github.com/tsachiherman>
Discussions-To: <GitHub Discussion URL (POPULATED BY MAINTAINER, DO NOT SET)>
Status: Proposed
Track: Standards
Replaces (*optional): n/a
Superseded-By (*optional): n/a
```

## Abstract

Avalanche offers developers flexibility through subnets and EVM-compatible smart contracts. However, the platform's deterministic block execution limits the use of traditional random number generators within these contracts.

To address this, a mechanism is proposed to generate verifiable, non-cryptographic random number seeds on the Avalanche platform. This method ensures uniformity while allowing developers to build more versatile applications.


## Motivation

Reliable randomness is essential for building exciting applications on Avalanche. Games, participant selection, dynamic content, supply chain management, and decentralized services all rely on unpredictable outcomes to function fairly. Randomness also fuels functionalities like unique identifiers and simulations. Without a secure way to generate random numbers within smart contracts, Avalanche applications become limited.

Avalanche's traditional reliance on external oracles for randomness creates complexity and bottlenecks. These oracles inflate costs, hinder transaction speed, and are cumbersome to integrate. As Avalanche scales to more Subnets, this dependence on external systems becomes increasingly unsustainable.

A solution for verifiable random number generation within Avalanche solves these problems. It provides fair randomness functionality across the chains, at no additional cost. This paves the way for a more efficient Avalanche ecosystem.

## Specification

### Changes Summary

Existing avalanche protocol breaks the block building into two parts : external and internal. The external block is the SnowMan++ block, whereas the internal block is the actual virtual machine block.

To support randomness, a BLS based VRF implementation is used, that would be recursively signing its own signatures as its message. Since the BLS signatures are deterministic, they provide a great way to construct a reliable VRF.

For proposers that do not have a BLS key associated with their node, the hash of the signature from the previous round.

In order to bootstrap the signatures chain, a missing signature would be replaced with a byte slice that is the hash product of a verifiable and trustable seed.

The changes proposed here would affect the way a block is being validated. Therefore, when this change gets implemented, it needs to be deployed as a mandatory upgrade.

```
		+-----------------------+           +-----------------------+
		|  Block n              | <-------- |  Block n+1            |
		+-----------------------+           +-----------------------+
		|  VRF-Sig(n)           |           |  VRF-Sig(n+1)         |
		|  ...                  |           |  ...                  |
		+-----------------------+           +-----------------------+

		+-----------------------+           +-----------------------+
		|  VM n                 |           |  VM n+1               |
		+-----------------------+           +-----------------------+
		|  VRF-Out(n)           |           |  VRF-Out(n+1)         |
		+-----------------------+           +-----------------------+

		VRF-Sig(n+1) = Sign(VRF-Sig(n), Block n+1 proposer's BLS key)
		VRF-Out(n) = Hash(VRF-Sig(n))
```

### Changes Details

#### Step 1. Adding BLS signature to proposed blocks

```golang
type statelessUnsignedBlock struct {
	…
	vrfSig    []byte `serialize:”true”`
}
```

#### Step 2. Populate signature

When a block proposer attempts to build a new block, it would need to use the parent block as a reference.

The `vrfSig` field within each block is going to be daisy-chained to the `vrfSig` field from it's parent block.

Populating the `vrfSig` would following this logic:

1. The current proposer has a BLS key
   
	a. If the parent block has an empty `vrfSig` signature, that signature would be signed using the proposer’s BLS key. This is the base case.

	b. If the parent block does not have an empty `vrfSig` signature, the proposer would sign the bootStrappingBlockSignature with its BLS key. See the bootStrappingBlockSignature details below.

2. The current proposer does not have a BLS key
   
   a. If the parent block has a non empty `vrfSig` signature, the proposer would set the proposed block `vrfSig` to the 32 byte hash result of the following preimage:
	```
	+-------------------------+----------+------------+
	|  prefix :               | [8]byte  | "rng-derv" |
	+-------------------------+----------+------------+
	|  vrfSig :               | [96]byte |  96 bytes  |
	+-------------------------+----------+------------+
	```

	b. If the parent block has an empty `vrfSig` signature, the proposer would leave the field empty.

The bootStrappingBlockSignature that would be used above is the hash of the following preimage:

```
+-----------------------+----------+------------+
|  prefix :             | [8]byte  | "rng-root" |
+-----------------------+----------+------------+
|  chainID :            | [32]byte |  32 bytes  |
+-----------------------+----------+------------+
|  networkID:           | uint32   |  4 bytes   |
+-----------------------+----------+------------+
```

#### Step 3. Signature Verification

This signature verification would perform the exact opposite of what was done in step 2, and would verify the cryptographic correctness of the operation.

Validating the `vrfSig` would following this logic:
1. The proposer has a BLS key

	a. If the parent block had a non empty `vrfSig`, then a BLS signature verification of the proposed block  `vrfSig` would be made against the proposer’s BLS public key and the parent's block  `vrfSig` as the message.
	b. If the parent block does has an empty `vrfSig`, then a BLS signature verification of the proposed block `vrfSig` against the proposer’s BLS public key and bootStrappingBlockSignature would take place.

2. The proposer does not have a BLS key

	a. If the parent block had a non empty `vrfSig`, then the hash of the preimage ( as described above ) would be compared against the proposed `vrfSig`.

	b. If the parent block has an empty `vrfSig` then the proposer's `vrfSig` would be validated to be empty.

#### Step 4. Extract the VRF Out and pass to block builders

Calculating the VRF Out would be done by hashing the preimage of the following struct:

```
+-----------------------+----------+------------+
|  prefix :             | [8]byte  | "vrfout  " |
+-----------------------+----------+------------+
|  vrfout:              | [96]byte |  96 bytes  |
+-----------------------+----------+------------+
```

Before calculating the VRF Out, the method needs to explicitly check the case where the `vrfSig` is empty. In that case, the output of the VRF Out needs to be empty as well.

## Backwards Compatibility

The above design has taken backward compatibility considerations. The chain would keep working as before, and at some point, would have the newly added `vrfSig` populated.

From usage perspective, each VM would need to make its own decision on whether it should use the newly provided random seed. Initially, this random seed would be all zeros - and would get populated once the feature rolled out to a sufficient number of nodes.

## Security Considerations

Virtual machine random seeds, while appearing to offer a source of randomness within smart contracts, fall short when it comes to cryptographic security. Here's a breakdown of the critical issues:

- Limited Permutation Space: The number of possible random values is derived from the number of validators. While no validator, nor a validator set, would be able to manipulate the randomness into any single value, a nefarious actor(s) might be able to exclude specific numbers.
- Predictability Window: The seed value might be accessible to other parties before the smart contract can benefit from its uniqueness. This predictability window creates a vulnerability. An attacker could potentially observe the seed generation process and predict the sequence of "random" numbers it will produce, compromising the entire cryptographic foundation of your smart contract.

Despite these limitations appearing severe, attackers face significant hurdles to exploit them. First, the attacker can't control the random number, limiting the attack's effectiveness to how that number is used. Second, a substantial amount of AVAX is needed. And last, such an attack would likely decrease AVAX's value, hurting the attacker financially.

One potential attack vector involves collusion among multiple proposers to manipulate the random number selection. These attackers could strategically choose to propose or abstain from proposing blocks, effectively introducing a bias into the system. By working together, they could potentially increase their chances of generating a random number favorable to their goals.

However, the effectiveness of this attack is significantly limited for the following reasons:
- Limited options: While colluding attackers expand their potential random number choices, the overall pool remains immense (2^32 possibilities). This drastically reduces their ability to target a specific value.
- Protocol's countermeasure: The protocol automatically eliminates any bias introduced by previous proposals once an honest proposer submits their block.

In essence, while this attack is theoretically possible, its practical impact is negligible due to the vast number of potential outcomes and the protocol's inherent safeguards.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

