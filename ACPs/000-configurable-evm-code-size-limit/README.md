| ACP | TBD |
| :--- | :--- |
| **Title** | Configurable EVM Contract Code Size Limit for Avalanche L1s |
| **Author(s)** | Giacomo Barbieri [(@ijaack94)](https://x.com/ijaack94) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This ACP proposes making the maximum deployed EVM contract code size configurable for Avalanche L1s that run Subnet-EVM-compatible execution environments. Today, Avalanche L1s inherit Ethereum's fixed runtime bytecode limit introduced by EIP-170 (24,576 bytes), as well as the corresponding initcode ceiling introduced by EIP-3860. While these defaults preserve Ethereum compatibility, they unnecessarily constrain Avalanche L1s whose validator sets are willing to accept larger contract artifacts in exchange for simpler application architecture, fewer proxy splits, and less deployment fragmentation.

Under this ACP, Avalanche L1s MAY configure a higher consensus-level `maxCodeSize`, while preserving the Ethereum default when no override is specified. The associated initcode limit MUST scale accordingly so deployments remain internally consistent. This ACP does not change opcode semantics, gas accounting, or C-Chain behavior by default. Instead, it gives Avalanche L1s an explicit, opt-in configurability surface for bytecode size limits, allowing each network to choose its own balance between compatibility, performance, and developer ergonomics.

## Motivation

Avalanche's value proposition is not strict uniformity with Ethereum at all layers; it is configurable execution environments with credible validator-enforced rules. The EIP-170 contract size limit is a reasonable default for generalized EVM networks, but it is still a policy choice rather than a law of nature.

Several application teams want to deploy larger contracts for reasons that are operationally rational:

- complex applications may prefer fewer contract boundaries and fewer proxy patterns;
- security reviews are sometimes easier on a single coherent artifact than on a heavily fragmented deployment;
- some chains optimize for app-specific throughput or controlled validator sets rather than maximal bytecode portability;
- a strict Ethereum-sized cap can force unnecessary engineering workarounds on Avalanche L1s even when their validators are comfortable accepting the tradeoff.

Recent market examples, including Robinhood's reported support for materially larger contracts than Ethereum, show that EVM builders increasingly view code-size policy as a chain-level design parameter. Avalanche should support the same flexibility where it fits the chain operator's goals.

At the same time, a blanket increase on the Avalanche C-Chain would be harder to justify because it changes network-wide expectations for the most Ethereum-like Avalanche environment. This ACP therefore focuses on the cleaner and more Avalanche-native step first: make the limit configurable for Avalanche L1s, keep the default unchanged, and let validators opt in deliberately.

## Specification

### 1. Scope

This ACP applies to Avalanche L1s that run Subnet-EVM-compatible execution environments.

This ACP does **not** change the C-Chain default contract size limit or initcode limit. A future ACP MAY propose enabling a different limit on the C-Chain, but that is outside the scope of this proposal.

### 2. New Consensus Parameters

Subnet-EVM-compatible Avalanche L1s MUST support the following consensus parameters:

- `maxCodeSize`: maximum number of bytes permitted for deployed runtime bytecode.
- `maxInitCodeSize`: maximum number of bytes permitted for initcode during contract creation.

If these parameters are not explicitly configured, implementations MUST preserve Ethereum-compatible defaults:

- `maxCodeSize = 24,576` bytes
- `maxInitCodeSize = 49,152` bytes

These values correspond to the existing EIP-170 and EIP-3860 defaults.

### 3. Configuration Rules

`maxCodeSize` and `maxInitCodeSize` are consensus parameters and therefore MUST be identical across all validating nodes for a given Avalanche L1.

An Avalanche L1 MAY set these parameters:

- in genesis at chain launch; or
- in a coordinated network upgrade activated at a specific timestamp.

If `maxCodeSize` is explicitly set and `maxInitCodeSize` is omitted, `maxInitCodeSize` MUST default to `2 * maxCodeSize`.

Implementations MAY reject configurations where:

- `maxCodeSize < 24,576` bytes;
- `maxInitCodeSize < maxCodeSize`; or
- `maxInitCodeSize != 2 * maxCodeSize`, if the implementation chooses to enforce a fixed ratio for simplicity.

To maximize interoperability across Avalanche L1 tooling, reference implementations should initially enforce:

- `24,576 <= maxCodeSize <= 98,304`
- `maxInitCodeSize = 2 * maxCodeSize`

This gives Avalanche L1s up to 4x Ethereum's deployed-code limit while keeping the parameter surface narrow and predictable.

### 4. Contract Creation Semantics

For `CREATE` and `CREATE2`:

- if the resulting runtime bytecode exceeds `maxCodeSize`, contract creation MUST fail;
- if the supplied initcode exceeds `maxInitCodeSize`, contract creation MUST fail.

No other EVM execution semantics are changed by this ACP.

In particular:

- opcode behavior is unchanged;
- code deposit gas remains unchanged;
- existing gas metering for initcode analysis remains unchanged unless separately modified by a future ACP.

### 5. RPC and Tooling Exposure

Implementations SHOULD expose the active `maxCodeSize` and `maxInitCodeSize` values through chain configuration inspection endpoints, client configuration output, or both, so that explorers, SDKs, deployment tooling, and auditors can determine the active deployment constraints of a given Avalanche L1.

### 6. Activation Requirements

Any network upgrade that changes `maxCodeSize` or `maxInitCodeSize` MUST clearly specify:

- activation timestamp;
- old and new values;
- whether previously failing deployments are expected to become valid after activation.

Previously deployed contracts are unaffected. Only contract creation validity after activation changes.

## Backwards Compatibility

This ACP is backwards compatible by default because chains that do not opt in retain Ethereum-compatible defaults.

For Avalanche L1s that do opt in, the main compatibility impact is positive but chain-specific: contracts that would previously fail deployment due to EIP-170/EIP-3860-sized limits may deploy successfully after activation.

Potential compatibility considerations include:

- deployment tools may assume Ethereum's 24,576-byte runtime limit;
- explorers, static analyzers, and indexers may have hardcoded assumptions around EIP-170-sized artifacts;
- bridges, wallets, and SDKs that market themselves as “Ethereum-compatible” may need to clarify that compatibility does not imply Ethereum-identical code-size policy.

Because this ACP only relaxes deployment constraints for opt-in Avalanche L1s, ordinary transaction execution and already-deployed contracts remain unaffected.

## Reference Implementation

A reference implementation is not yet provided.

A compliant implementation should:

1. add `maxCodeSize` and `maxInitCodeSize` as explicit consensus parameters in Subnet-EVM-compatible configuration;
2. validate those parameters during genesis parsing and network-upgrade activation;
3. apply the configured values in contract creation checks for `CREATE` and `CREATE2`;
4. expose the configured values through operator-facing or tooling-facing interfaces; and
5. include local-network telemetry comparing contract deployment latency, block-building behavior, and state growth at multiple size thresholds.

## Security Considerations

Larger deployable contracts increase worst-case resource usage during contract creation, code dissemination, and downstream tooling analysis. This does not automatically make larger limits unsafe, but it does make the tradeoff explicit.

Relevant risks include:

- **Block construction and verification cost:** larger contract artifacts may increase block processing and propagation time.
- **State growth:** larger bytecode blobs increase long-term state footprint.
- **Tooling fragility:** off-chain systems may silently assume Ethereum-sized artifacts.
- **DoS surface:** chains that raise limits too aggressively may increase the cost asymmetry between deployers and validators if other guardrails are not sufficient.

These risks are why this ACP proposes:

- opt-in L1 configurability rather than a blanket network-wide increase;
- unchanged Ethereum-compatible defaults;
- a narrow initial recommended range capped at 4x Ethereum's default; and
- explicit visibility of active limits for operators and tooling.

Any Avalanche L1 adopting a higher limit should benchmark deployment latency, block propagation, and archival/storage implications before activation.

## Open Questions

- Should the initial interoperable range be capped at 4x Ethereum's default, or should implementations permit a wider range from day one?
- Should C-Chain eventually expose the same configurability, or should this remain an Avalanche L1-only feature?
- Should tooling-facing RPC standardization be part of this ACP, or left to implementation-specific documentation?
- Should future work pair larger code-size limits with additional deployment gas or other anti-DoS guardrails?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
