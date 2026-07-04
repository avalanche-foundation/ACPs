| ACP | TBD |
| :--- | :--- |
| **Title** | C-Chain-First EVM Contract Code Size Increase with Avalanche L1 Customizability |
| **Author(s)** | Giacomo Barbieri [(@ijaack94)](https://x.com/ijaack94) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This ACP proposes a C-Chain-first increase to the maximum deployed EVM contract code size, while also making the same limit configurable for Avalanche L1s that run Subnet-EVM-compatible execution environments. Today, both environments inherit Ethereum's fixed runtime bytecode limit introduced by EIP-170 (24,576 bytes), as well as the corresponding initcode ceiling introduced by EIP-3860. Those defaults preserve Ethereum compatibility, but they can unnecessarily constrain Avalanche builders who prefer larger contract artifacts in exchange for simpler application architecture, fewer proxy splits, and less deployment fragmentation.

Under this ACP, the Avalanche C-Chain default `maxCodeSize` is increased to `49,152` bytes, with `maxInitCodeSize` increased proportionally to `98,304` bytes. Avalanche L1s MUST support configurable `maxCodeSize` and `maxInitCodeSize` consensus parameters, allowing each network to preserve Ethereum defaults, match the C-Chain, or choose another supported value. This ACP does not change opcode semantics or core gas accounting. Instead, it makes code-size policy an explicit Avalanche design surface: C-Chain moves first, and Avalanche L1s retain customizability.

## Motivation

Avalanche should not treat Ethereum's bytecode-size limit as immutable when the constraint is materially affecting real builder behavior. The EIP-170 contract size limit is a policy choice optimized for Ethereum's historical tradeoffs, not a universal optimum for all EVM environments.

This proposal takes the position that Avalanche should lead on the C-Chain first.

Several application teams want to deploy larger contracts for reasons that are operationally rational:

- complex applications may prefer fewer contract boundaries and fewer proxy patterns;
- security reviews are sometimes easier on a single coherent artifact than on a heavily fragmented deployment;
- bytecode fragmentation can add engineering and auditing overhead without improving application logic;
- Avalanche is well positioned to compete for builders who want a more permissive, but still well-guarded, EVM environment.

Recent market examples, including Robinhood's reported support for materially larger contracts than Ethereum, show that code-size policy is increasingly a competitive chain feature rather than just a low-level implementation detail.

Making the C-Chain the first mover does two things:

1. it gives Avalanche's flagship EVM environment a stronger builder proposition immediately; and
2. it establishes a clean precedent for Avalanche L1s to customize the same parameter according to their own validator and application requirements.

Avalanche L1 customizability still matters. Some L1s may want to retain Ethereum defaults for strict compatibility, while others may want to match or exceed the C-Chain limit. This ACP therefore pairs a C-Chain-first default increase with an explicit L1 configuration surface rather than forcing a single network-wide policy everywhere.

## Specification

### 1. Scope

This ACP applies to:

- the Avalanche C-Chain; and
- Avalanche L1s that run Subnet-EVM-compatible execution environments.

On the C-Chain, this ACP increases the default contract size and initcode limits.

On Avalanche L1s, this ACP requires support for configurable contract size and initcode limits through consensus parameters.

### 2. New Consensus Parameters

Subnet-EVM-compatible Avalanche L1s MUST support the following consensus parameters:

- `maxCodeSize`: maximum number of bytes permitted for deployed runtime bytecode.
- `maxInitCodeSize`: maximum number of bytes permitted for initcode during contract creation.

For the Avalanche C-Chain, the post-activation defaults are:

- `maxCodeSize = 49,152` bytes
- `maxInitCodeSize = 98,304` bytes

For Avalanche L1s, if these parameters are not explicitly configured, implementations MUST preserve Ethereum-compatible defaults:

- `maxCodeSize = 24,576` bytes
- `maxInitCodeSize = 49,152` bytes

These values correspond to the existing EIP-170 and EIP-3860 defaults.

### 3. Configuration Rules

`maxCodeSize` and `maxInitCodeSize` are consensus parameters and therefore MUST be identical across all validating nodes for a given Avalanche L1.

The Avalanche C-Chain MUST activate the following values at the upgrade timestamp defined by the implementation:

- `maxCodeSize = 49,152`
- `maxInitCodeSize = 98,304`

An Avalanche L1 MAY set its own parameters:

- in genesis at chain launch; or
- in a coordinated network upgrade activated at a specific timestamp.

If an Avalanche L1 explicitly sets `maxCodeSize` and omits `maxInitCodeSize`, `maxInitCodeSize` MUST default to `2 * maxCodeSize`.

Implementations MAY reject Avalanche L1 configurations where:

- `maxCodeSize < 24,576` bytes;
- `maxInitCodeSize < maxCodeSize`; or
- `maxInitCodeSize != 2 * maxCodeSize`, if the implementation chooses to enforce a fixed ratio for simplicity.

To maximize interoperability across Avalanche L1 tooling, reference implementations should initially enforce:

- `24,576 <= maxCodeSize <= 98,304`
- `maxInitCodeSize = 2 * maxCodeSize`

This lets Avalanche L1s preserve Ethereum compatibility, match the C-Chain, or increase further up to 4x Ethereum's deployed-code limit while keeping the parameter surface narrow and predictable.

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

This ACP is not backwards compatible with the pre-upgrade C-Chain deployment limit, because the C-Chain default contract size and initcode limits increase at activation.

On the C-Chain, the compatibility impact is limited to contract creation validity after activation: contracts that would previously fail due to EIP-170/EIP-3860-sized limits may deploy successfully after the upgrade. Ordinary execution semantics for already-deployed contracts remain unchanged.

For Avalanche L1s, this ACP is backwards compatible by default because chains that do not opt in retain Ethereum-compatible defaults.

Potential compatibility considerations across both environments include:

- deployment tools may assume Ethereum's 24,576-byte runtime limit;
- explorers, static analyzers, and indexers may have hardcoded assumptions around EIP-170-sized artifacts;
- bridges, wallets, and SDKs that market themselves as “Ethereum-compatible” may need to clarify that compatibility does not imply Ethereum-identical code-size policy.

## Reference Implementation

The reference implementation for this ACP is split across two codebases:

- **Coreth (C-Chain):** a network upgrade that activates `maxCodeSize = 49,152` and `maxInitCodeSize = 98,304` on the C-Chain, and threads those activated values through the existing EIP-170/EIP-3860 enforcement points in `core/state_transition.go`, `core/txpool/validation.go`, and `sync/client/client.go`, with matching coverage in `core/state_processor_test.go`, `sync/client/client_test.go`, and `params/protocol_params_test.go`.
- **Subnet-EVM (Avalanche L1s):** explicit `maxCodeSize` and `maxInitCodeSize` consensus parameters in chain configuration and upgrade parsing, validation at genesis and upgrade time, `maxInitCodeSize = 2 * maxCodeSize` defaulting when omitted, and enforcement through the same contract creation, txpool, and sync code paths. The implementation naturally touches `params/extras/config.go`, `params/config_extra.go`, `core/genesis.go`, `core/state_transition.go`, `core/txpool/validation.go`, and `sync/client/client.go`, with corresponding tests.

Together, these changes provide the full reference implementation: Coreth demonstrates the C-Chain-first activation path, while Subnet-EVM demonstrates the Avalanche L1 customizability surface.

A compliant implementation should:

1. increase the C-Chain `maxCodeSize` to `49,152` and `maxInitCodeSize` to `98,304` at a defined upgrade point;
2. add `maxCodeSize` and `maxInitCodeSize` as explicit consensus parameters in Subnet-EVM-compatible Avalanche L1 configuration;
3. validate those parameters during genesis parsing and network-upgrade activation;
4. apply the configured values in contract creation checks for `CREATE` and `CREATE2`;
5. expose the configured values through operator-facing or tooling-facing interfaces; and
6. include local-network telemetry comparing contract deployment latency, block-building behavior, and state growth at multiple size thresholds.

## Security Considerations

Larger deployable contracts increase worst-case resource usage during contract creation, code dissemination, and downstream tooling analysis. This does not automatically make larger limits unsafe, but it does make the tradeoff explicit.

Relevant risks include:

- **Block construction and verification cost:** larger contract artifacts may increase block processing and propagation time.
- **State growth:** larger bytecode blobs increase long-term state footprint.
- **Tooling fragility:** off-chain systems may silently assume Ethereum-sized artifacts.
- **DoS surface:** chains that raise limits too aggressively may increase the cost asymmetry between deployers and validators if other guardrails are not sufficient.

These risks are why this ACP proposes:

- a bounded C-Chain increase to 2x Ethereum's default rather than an unbounded jump;
- explicit Avalanche L1 configurability rather than forcing the C-Chain value everywhere;
- a narrow initial recommended Avalanche L1 range capped at 4x Ethereum's default; and
- explicit visibility of active limits for operators and tooling.

The C-Chain upgrade should be benchmarked before activation, and any Avalanche L1 adopting a higher limit should benchmark deployment latency, block propagation, and archival/storage implications before activation.

## Open Questions

- Is 2x Ethereum's default the right C-Chain starting point, or should the first C-Chain increase be smaller or larger?
- Should Avalanche L1 implementations be permitted to exceed 4x Ethereum's default from day one, or should the interoperable range remain tighter initially?
- Should tooling-facing RPC standardization be part of this ACP, or left to implementation-specific documentation?
- Should future work pair larger code-size limits with additional deployment gas or other anti-DoS guardrails?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
