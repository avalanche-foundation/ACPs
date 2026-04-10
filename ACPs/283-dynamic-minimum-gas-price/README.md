| ACP | 283 |
| :- | :- |
| **Title** | Dynamic Minimum Gas Price |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Martin Eckardt ([@martineckardt](https://github.com/martineckardt)) |
| **Status** | Proposed [Discussion](https://github.com/avalanche-foundation/ACPs/discussions/284) |
| **Track** | Standards |

## Abstract

Proposes making the minimum gas price parameter on the C-Chain dynamically adjustable via validator voting, using the same mechanism established by [ACP-176](../176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) (gas target) and [ACP-226](../226-dynamic-minimum-block-times/README.md) (minimum block delay).

Currently, ACP-176 sets the minimum gas price to 1 wei, allowing spam transactions to flood the chain at near-zero cost during low-activity periods. This has been exploited by protocols such as XEN, which performs proof-of-work mining on-chain when gas prices are negligible, causing unnecessary state growth and validator resource consumption. The current defense relies on wallets artificially inflating gas prices via failing transactions — a stopgap measure costing the community several AVAX per day.

This ACP applies the proven validator voting pattern from ACP-176 and ACP-226 to the minimum gas price. Validators express their preferred minimum gas price via node configuration, and the network converges on the median preference weighted by stake. The dynamic minimum gas price acts as a true floor: during low-activity periods the floor binds, while during congestion the existing ACP-176 fee mechanism dominates unchanged. This requires no future hard forks for parameter adjustments and allows the network to organically respond to changing spam conditions.

## Motivation

[ACP-176](../176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) introduced a dynamic fee mechanism for the C-Chain, setting the minimum gas price $M$ to 1 wei — the smallest possible denomination of the native EVM asset. The rationale was that the dynamic fee mechanism would raise prices under load, making a near-zero floor acceptable for price discovery.

In practice, this creates a significant vulnerability during low-activity periods. When the excess gas consumption $x = 0$ (no recent demand above the target), the base fee drops to 1 wei. At 1 wei per gas, a simple transfer consuming 21,000 gas costs 21,000 wei ($\approx 0.000000000000021$ AVAX) — effectively free. This creates zero economic barrier for spam during quiet periods.

The validator voting mechanism for adjusting network parameters has already been proven twice on the Avalanche network:

1. **ACP-176**: Validators dynamically adjust the target gas consumption rate $T$
2. **ACP-226**: Validators dynamically adjust the minimum block delay time

This ACP applies the same proven pattern to the minimum gas price. The benefits are:

- **No hard forks required**: Future adjustments to the minimum gas price do not require coordinated network upgrades
- **Stake-weighted convergence**: The effective value converges on the median preference weighted by validator stake
- **Decentralized governance**: Each validator independently sets their preference based on local assessment of network conditions

Pre-ACP-176, the C-Chain gas price was approximately 25 nAVAX. Even a minimum gas price floor of 0.01 nAVAX — 2,500x below this baseline — would be entirely invisible to normal users while making spam economically infeasible.

## Specification

This ACP introduces a dynamic minimum gas price using the same exponential adjustment and validator voting mechanism described in [ACP-176](../176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) (for the gas target $T$) and [ACP-226](../226-dynamic-minimum-block-times/README.md) (for the minimum block delay). 

## Backwards Compatibility

The changes proposed in this ACP require a network upgrade in order to take effect. Prior to its activation, the current minimum gas price of 1 wei continues to apply. Its activation should have minimal compatibility effects:

- **Transaction formats**: Unchanged. Wallets and transaction signing are not impacted.
- **User fees**: The initial minimum gas price of approximately 0.01 nAVAX is far below the gas prices users experience during any meaningful network activity (~25 nAVAX historically). The change is invisible to legitimate users.
- **Tooling**: The new header field is RLP-optional (following the same pattern as `minimumBlockDelayExcess` from ACP-226). Block explorers and indexers that do not parse the new field will continue to work.
- **Gas price estimation APIs**: `eth_gasPrice` and related APIs should respect the new dynamic minimum gas price floor.

## Security Considerations

This ACP changes the minimum gas price of the C-Chain from a static constant to a dynamically-adjusted parameter governed by validator voting. Several security aspects should be considered:

**Validators setting the minimum gas price too high**: The exponential mechanism bounds how quickly the minimum gas price can change per block, giving the community time to detect and respond to concerning changes. Additionally, a hard upper bound at $q_{max}$ (100 nAVAX) provides an absolute ceiling — even if all validators vote for the maximum, the minimum gas price cannot exceed this cap. Validators are also economically incentivized not to price out users, as doing so reduces network utility and the value of their staked AVAX.

**Validators setting the minimum gas price too low**: The same bounded adjustment speed applies in the downward direction, providing significant time for detection and response.

**Comparison to existing approaches**: The minimum gas price adjustment mechanism has the same structure as the proven gas target adjustment (ACP-176) and minimum block delay adjustment (ACP-226), providing confidence in its security properties. Compared to Base's approach of administrator-set static floors, the validator voting mechanism achieves the same spam deterrence in a decentralized manner.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
