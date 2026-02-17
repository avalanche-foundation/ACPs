| ACP | 273 |
| :--- | :--- |
| **Title** | Reduce Minimum Validator Staking Period to 24 Hours |
| **Author(s)** | Meaghan FitzGerald ([@meaghanfitzgerald](https://github.com/meaghanfitzgerald)) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This proposal reduces the minimum validator staking period on Avalanche's Primary Network from 2 weeks (336 hours) to 1 day (24 hours). The change lowers barriers to validator participation while maintaining network security, enabling more flexible staking strategies and improving capital efficiency across the ecosystem.

## Motivation

### Current State

The Avalanche Primary Network currently requires validators and delegators to stake for a minimum of 2 weeks (336 hours) to be eligible for staking rewards. While this ensures validator commitment to network security, it creates friction for participants who require more flexible liquidity access.

### Benefits of a Shorter Minimum Period

1. Increased Validator Participation: A 24-hour minimum removes a significant barrier to entry. Validators who cannot commit capital for 2 weeks can now participate, increasing overall network stake and decentralization.

2. Improved Capital Efficiency: Stakers gain reliable access to liquidity after 24 hours rather than 14 days. This enables more dynamic capital allocation strategies while maintaining commitment to network security during the staking period.

### Reward Impact Analysis

Using Avalanche's reward formula at the time of writing, the APY difference between 24-hour and 2-week staking periods is approximately 0.04 percentage points (6.08% vs 6.12%). This minimal difference preserves reward incentives while enabling liquidity flexibility.

## Specification

### Technical Changes

Update the minimum staking duration by modifying `MinStakeDuration` in the genesis configuration:

Current:
```go
MinStakeDuration = 336 * time.Hour
```

Proposed:
```go
MinStakeDuration = 24 * time.Hour
```

### Implementation Details

The staking mechanism remains unchanged. Validators must still meet all existing requirements:

- Minimum stake: 2,000 AVAX for Primary Network validators
- Uptime requirement: 90% (per ACP-267)
- Hardware requirements: As specified in ACP-256

Only the minimum duration parameter changes, with all other validation rules and reward calculations preserved.

## Backwards Compatibility

This is a non-backwards compatible change to the P-Chain validation and execution requirements. Therefore, it requires a network upgrade in order to be implemented.

This change affects only new staking periods initiated after activation. Existing validators with active staking periods remain unaffected and will complete their original durations under the previous rules.

The change is non-breaking: all existing validation infrastructure, reward mechanisms, and consensus operations continue functioning without modification.

## Security Considerations

Shorter staking periods increase the likelihood of significant validator churn over a short period of time. This may impact Primary Network consensus safety and the reliability of Interchain Message delivery, which is required for L1 validator operations. This risk may impact the eventual implementation details of this ACP.

Additionally, if the incentive to stake for longer than 24 hours is insufficient, the vast majority of stake may opt for the minimum 24-hour duration (particularly likely with the addition of [ACP-236](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/236-auto-renewed-staking/README.md), which introduces auto-renewed staking). In this scenario, essentially all entities securing the network could change completely within 24 hours. The entire validator set could disappear, or a low-stake-weight validator could become a high-stake-weight validator within a single day (i.e. a validator could go from 1% network stake to 20% network stake in just a few hours). This instability poses a real cost in terms of network security and should be weighed against the benefits of shorter staking periods.

Validators remain subject to the same accountability standards during the 24-hour period. Network consensus sampling assumes the same validator availability model. Historical data demonstrates that validator uptime patterns remain consistent regardless of staking duration length, as infrastructure quality and operational commitment drive uptime, not duration requirements alone.

## Open Questions

1. If the minimum validation period is shortened, should the minimum delegation period also be reduced?

At the time of writing, the ratio of `AddPermissionlessDelegatorTx`s to `AddPermissionlessValidatorTx`s over the past 365 days was 100:1, with delegations accounting for 46% of all P-Chain transactions. Assuming all delegations are currently set to the minimum duration (14 days), reducing the minimum to 24 hours would increase the rate of state growth from delegation operations on the P-Chain by approximately 14x.

Given this potential impact, if the minimum delegation period is reduced, should there be an additional requirement that short-term delegations (e.g., 24 hours) must stake at least 1,000 AVAX?

2. Based on the scenario posed in Security Considerations, should the rewards rate differ for shorter vs longer validation periods to incentivize network stability?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
