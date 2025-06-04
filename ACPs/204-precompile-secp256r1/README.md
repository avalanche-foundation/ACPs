# ACP-204: Precompile for secp256r1 Curve Support

| ACP | 204 |
| :--- | :--- |
| **Title** | Precompile for secp256r1 Curve Support |
| **Author(s)** | [Santiago Cammi](https://github.com/scammi), [Arran Schlosberg](https://github.com/ARR4N)  |
| **Status** | Proposed ([Discussion]()) |
| **Track** | Standards |

## Abstract

This proposal introduces a precompiled contract that performs signature verifications for the secp256r1 elliptic curve on Avalanche's C-Chain. The precompile will be implemented at address `0x0200000000000000000000000000000000000006` and will enable native verification of P-256 signatures, significantly improving gas efficiency for biometric authentication systems, WebAuthn, and modern device-based signing mechanisms.

## Motivation

The secp256r1 (P-256) elliptic curve is the standard cryptographic curve used by modern device security systems, including Apple's Secure Enclave, Android Keystore, WebAuthn, and Passkeys. However, Avalanche currently only supports secp256k1 natively, forcing developers to use expensive Solidity-based verification that costs [200k-330k gas per signature verification](https://hackmd.io/@1ofB8klpQky-YoR5pmPXFQ/SJ0nuzD1T#Smart-Contract-Based-Verifiers).

This ACP proposes implementing EIP-7212's secp256r1 precompiled contract to unlock significant ecosystem benefits:

### Enterprise & Institutional Adoption

- Reduced onboarding friction: Enterprises can leverage existing biometric authentication infrastructure instead of managing seed phrases or hardware wallets
- Regulatory compliance: Institutions can utilize their approved device security standards and identity management systems
- Cost optimization: 100x gas reduction (from 200k-330k to 3,450 gas) makes enterprise-scale applications economically viable

The 100x gas cost reduction makes these use cases economically viable while maintaining the security properties institutions and users expect from their existing devices.

## Specification

This ACP implements [RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md) for secp256r1 signature verification on Avalanche. The specification follows RIP-7212 exactly, with the precompiled contract deployed at address `0x0200000000000000000000000000000000000006`.

### Core Functionality

- Input: 160 bytes (message hash + signature components r,s + public key coordinates x,y)
- Output: success: 32 bytes `0x...01`; failure: no data returned
- Gas Cost: 3,450 gas (based on EIP-7212 benchmarking)
- Validation: Full compliance with NIST FIPS 186-3 specification

### Activation

This precompile may be activated as part of Avalanche's next network upgrade. Individual Avalanche L1s and subnets could adopt this enhancement independently through their respective client software updates.

For complete technical specifications, validation requirements, and implementation details, refer to [RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md).

## Backwards Compatibility

This ACP introduces a new precompiled contract and does not modify existing functionality. No backwards compatibility issues are expected since:

1. The precompile uses a previously unused address
2. No existing opcodes or consensus rules are modified
3. The change is additive and opt-in for applications

Adoption requires a coordinated network upgrade for the C-Chain. Other EVM L1s can adopt this enhancement independently by upgrading their client software.

## Security Considerations

### Cryptographic Security

- The secp256r1 curve is standardized by NIST and widely vetted
- Security properties are comparable to secp256k1 (used by ECRECOVER)
- Implementation follows NIST FIPS 186-3 specification exactly

### Implementation Security
 
- Signature verification (vs public-key recovery) approach maximizes compatibility with existing P-256 ecosystem
- No malleability check included to match NIST specification, but wrapper libraries may choose to add this
- Input validation prevents invalid curve points and out-of-range signature components

### Network Security

- Gas cost prevents potential DoS attacks through expensive computation
- No consensus-level security implications beyond standard precompile considerations

## Reference Implementation

The implementation will build upon existing work:

1. EIP-7212 Reference: The [BOR implementation](https://github.com/maticnetwork/bor/pull/1069) of EIP-7212 provides the foundation
2. Coreth Implementation: Integration with Avalanche's C-Chain (Avalanche's fork of go-ethereum)
3. Cryptographic Library: Implementation utilizes Go's standard library `crypto/ecdsa` and `crypto/elliptic` packages, which implement NIST P-256 per FIPS 186-3 ([Go documentation](https://pkg.go.dev/crypto/elliptic#P256))


The implementation follows established patterns for precompile integration, adding the contract to the precompile registry and implementing the verification logic using established cryptographic libraries.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).