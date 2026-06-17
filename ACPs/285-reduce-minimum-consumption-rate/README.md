| ACP | 285 |
| :--- | :--- |
| **Title** | Reduce Minimum Consumption Rate |
| **Author(s)** | Matias Antonio ([@crypto-virgil](https://github.com/crypto-virgil)), Eric Lu ([@ericlu-avax](https://github.com/ericlu-avax)), Martin Eckardt ([@martineckardt](https://github.com/martineckardt)), Meaghan FitzGerald ([@meaghanfitzgerald](https://github.com/meaghanfitzgerald)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/288)) |
| **Track** | Standards |

## Abstract

This proposal reduces the `MinConsumptionRate` parameter governing Primary Network staking rewards from 10% to 7.5%. `MinConsumptionRate` sets the annualized reward rate (ARR) for a validator whose stake duration approaches the minimum staking duration. `MaxConsumptionRate` is not modified, leaving the maximum reward available to long-term validators unchanged. The change is intended to improve the long-term alignment between staking duration and network security and to redistribute rewards toward longer-duration participants. It is one of a series of ACPs intended to improve network security and validator economics.

## Motivation

The proposed change has three goals: reducing the AVAX inflation rate, extending the network's security budget, and incentivizing longer staking durations.

First, lowering the reward-rate floor cuts the rewards given to short-duration stakes. This is projected to reduce the AVAX inflation rate by 0.5 to 1 percentage point per year, reflecting a lower reward rate partially offset by longer average staking durations. A reduction in inflation would benefit stakers and holders alike by structurally reducing the rate at which the protocol token is diluted.

Second, lowering the floor slows the rate at which the network spends its finite security budget, leaving more issuance available to sustain rewards in the future.

Third, lowering the floor steepens the ARR gradient between minimum- and maximum-duration staking, widening the gap from approximately 1.02 percentage points to 2.30 percentage points. This directly encourages new and existing validators to commit their stake for longer. Economic modeling estimates that this change would extend the stake-weighted average staking duration by approximately two months.

This proposal is designed to work alongside two related ACPs: [ACP-236](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/236-auto-renewed-staking/README.md) enables auto-renewal of staking positions, and [ACP-273](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/273-reduce-minimum-staking-duration/README.md) reduces the minimum staking duration to 48 hours. Together, those two proposals make a compounding 48-hour stake operationally identical to a long-tenor stake while still getting near-maximum rewards. By widening the staking duration difference to 2.30 percentage points, this ACP restores and improves the incentive for longer-term staking.

### Background

Primary Network staking rewards follow a linear interpolation between `MinConsumptionRate` and `MaxConsumptionRate` based on a validator's stake duration relative to the `MintingPeriod` (365 days). Annualized reward rate (ARR) denotes the rate produced by this formula:

```
ARR(d) = (RemainingSupply / CurrentSupply) × (MinRate + (MaxRate − MinRate) × d / MintingPeriod)
```

Where:
- `d` = validator stake duration in days
- `MinRate` = `MinConsumptionRate / PercentDenominator` = 10% (current)
- `MaxRate` = `MaxConsumptionRate / PercentDenominator` = 12% (unchanged)
- `MintingPeriod` = 365 days
- `RemainingSupply` = `SupplyCap − CurrentSupply`

This formula produces an ARR that increases linearly with duration. `MinConsumptionRate` sets the floor: the reward a validator receives if it stakes for the shortest possible period. `MaxConsumptionRate` sets the ceiling: the reward for a full 365-day commitment.

### Goal 1: Reduce AVAX Inflation

As noted in a [recent public discussion](https://x.com/frostLedger/status/2039683238285713557) by the Avalanche Foundation: *"Inflation should be used surgically. To reward specific behaviors: uptime, performance, ecosystem contribution. Not as a blanket payment for passively existing on the network."*

Once a validator selects a staking duration, `MinConsumptionRate` is a key determinant of the staking rewards received, and therefore of the AVAX created under the protocol as rewards. In aggregate, it is a key protocol parameter determining the AVAX inflation rate, alongside the aggregate stake durations chosen by validators. Given the current network stake, staking behavior, and protocol parameter configurations, the trailing one-year inflation rate is approximately 5.5%. That figure uses circulating supply (AVAX issued minus burned minus staked) as the monetary base, which is the closest analogue to M1 (cash and cash equivalents) in monetary economics.

The current 10% `MinConsumptionRate` functions as untargeted issuance at the floor of the reward formula. It is a code-level parameter, not the realized ARR. The formula multiplies it by the remaining-supply ratio, so the effective minimum-duration ARR today sits near 5.4% and keeps declining as the supply budget is drawn down. Because of that multiplier, lowering the parameter from 10% to 7.5% reduces the realized minimum-duration ARR by roughly 1.3 percentage points, not the full 2.5 that the raw numbers suggest. Separately, because `MaxConsumptionRate` is unchanged, the same cut more than doubles the gap between the minimum-duration and maximum-duration rates, widening the duration premium across the curve.

The mechanical impact of the change can be isolated by holding staking behavior constant, with the aggregate staked amount, staking durations, and transaction fee levels (which affect net emissions) all held at recently observed values. Under those assumptions, inflation is projected to be approximately 0.5 percentage point lower than under the current `MinConsumptionRate`.

![Projected net inflation](<Projected net inflation.png>)

The level effect reduces total annual AVAX issuance for as long as the supply budget remains, and the gradient effect roughly doubles the gap between minimum-duration and 365-day reward rates. Today a validator committing for 14 days receives roughly 84% of the rate given for a 365-day commitment, so duration accounts for only about 16% of that variation. Lowering the floor to 7.5% reduces that flat subsidy without touching the ceiling long-duration stakers already receive.

As discussed in more detail in Goal 3, staking behavior, and the choice of staking duration in particular, is also likely to respond to the change. A structural model of staking preferences was estimated from historical staking data under the current protocol parameter configurations, then used to run counterfactual simulations at the proposed `MinConsumptionRate`, accounting for the staking duration changes. The simulations suggest that the annual inflation rate would fall by 0.5 to 1 percentage point from its current level. The full model specification and estimation are available [here](https://x.com/eric_lu_sc/status/2049126828527251907?s=20).

### Goal 2: Extend the Network's Security Budget

Lower emissions for short-duration stakers slow the rate at which new AVAX enters circulation, even with no change in staking behavior.

If `MinConsumptionRate` were left at 10%, the protocol would continue spending its security budget faster than necessary. Primary Network rewards are funded from the finite 720 million AVAX cap, and every reward issued is a permanent draw on that budget. The slower it is spent, the more issuance remains available as a policy tool for future needs, and the longer the network sustains rewards before inflation-funded emissions taper toward zero.

A faster issuance schedule also brings more newly minted AVAX into circulating supply over any given period. To the extent that some portion of new issuance is sold rather than restaked, a higher emission rate adds more AVAX to circulating supply per unit time than a lower one. Reducing the floor to 7.5% slows that pace.

Holding staking behavior constant, the mechanical impact on the token supply path can be estimated in the same way. Both total supply and circulating supply grow more slowly under the lower `MinConsumptionRate` than at the current level, leaving more of the security budget available for the future.

![Projected AVAX supply](<Projected AVAX supply.png>)

### Goal 3: Encourage Validators to Stake Longer

Under the proposed change, the gap between the 14-day and 365-day ARR widens from 1.02 to 2.30 percentage points, more than doubling the duration premium. A wider premium should encourage validators to choose longer staking periods.

This matters because the relative incentive for longer staking has been shrinking as AVAX supply grows. ACP-236 will erode it further by largely eliminating the extra operational cost of renewing short-duration stakes. The proposed change compensates for that erosion and strengthens the incentive for long-term staking.

![AVAX Staking ARR vs Staking Duration over time](<AVAX Staking ARR vs Staking Duration over time.jpeg>)

![ARR Spread Between Long (365-day) and Short (14-day) Staking Duration](<ARR Spread.png>)

This is an expected secondary effect, not a guaranteed outcome. The same structural model behind Goal 1's inflation estimate also projects that the stake-weighted average staking duration would rise by roughly two months. Empirical evidence is mixed on whether the wider premium will flip large validators' duration choices. Conversations with node operators and large-position stakers suggest that liquidity preference may dominate even a 2.30 percentage point premium for some cohorts.

If validators do shift to longer durations, the network benefits further. Longer restaking cycles mean validators rotate less often, which strengthens stability and security. If validators do not change their behavior, or if they opt for shorter durations, the goals in the previous sections can still be realized.

### Proposed Fix: Lower the Floor, Keep the Ceiling

Reducing `MinConsumptionRate` from 10% to 7.5% produces the following reward schedule:

| Duration | Current ARR (MinRate = 10%) | ARR (MinRate = 7.5%) | Compounding ARR (MinRate = 7.5%) |
| -------- | --------------------------- | -------------------- | -------------------------------- |
| 2 days   | -                           | 4.00%                | 4.08%                            |
| 14 days  | 5.36%                       | 4.08%                | 4.16%                            |
| 90 days  | 5.58%                       | 4.58%                | 4.65%                            |
| 182 days | 5.85%                       | 5.18%                | 5.25%                            |
| 365 days | 6.38%                       | 6.38%                | 6.38%                            |

> All values calculated using the live reward formula with current circulating supply of 470,134,316 AVAX and supply cap of 720,000,000 AVAX. Compounding ARR assumes the user utilizes ACP-236 auto-renewal for 365 days, and all rewards received are added to the principal and restaked.

The maximum reward for a 365-day validator is unchanged. The incentive gradient more than doubles, going from a spread of 1.02 percentage points to 2.30 percentage points, an increase of over 125%.

## Specification

The Primary Network reward rate increases linearly with stake duration, between a floor $r$ (`MinConsumptionRate`) and a ceiling (`MaxConsumptionRate`). This ACP lowers the floor from $10\%$ to $7.5\%$, applied as a linear ramp over 90 days rather than a single step. The ceiling, minting period, and supply cap are unchanged.

### Mechanism

Let $r_0$ be the initial floor rate, $\Delta$ the total reduction, $P$ the ramp duration, and $t_0$ the activation timestamp.

Prior to activation, the floor rate is unchanged. For a stake with start time $t < t_0$:

$$r(t) = r_0$$

At and after activation, for a stake with start time $t \ge t_0$:

$$r(t) = r_0 - \Delta \cdot \frac{\min(t - t_0,\ P)}{P}$$

This decreases linearly from $r_0$ at $t_0$ to $r_0 - \Delta$ at $t_0 + P$, and holds at $r_0 - \Delta$ thereafter.

$r(t)$ is evaluated once per stake, using the stake's start time, and is fixed for that stake's full duration. A stake beginning during the ramp locks in the rate at its start, so a stake that starts on day 45 uses $8.75\%$ for its entire term. Stakes already active at $t_0$ are unaffected, since their reward is determined at their own start time.

Rates are stored in units of `PercentDenominator` $= 10^6$, so $r_0 = 100{,}000$ and $\Delta = 25{,}000$. Because $r(t)$ is an integer, it decreases by one unit, $0.0001$ percentage points, approximately every $5.184$ minutes, which is the $129{,}600$-minute window divided by $25{,}000$ units.

The parameters at activation are:

| Parameter | P-Chain Configuration |
| --------- | --------------------- |
| $r_0$ - initial `MinConsumptionRate` | 10% (100,000) |
| $\Delta$ - total reduction | 2.5% (25,000) |
| $P$ - reduction period | 90 days |
| $t_0$ - activation time | Helicon |

### A Note on the Linear Ramp

The reduction is phased so that the floor rate is continuous in a stake's start time. No single moment should carry a disproportionate incentive to enter just before or just after it.

A single-step reduction breaks that property. The floor would drop discontinuously at $t_0$. A stake beginning just before $t_0$ locks in $10\%$ for its full term, and a stake beginning just after locks in $7.5\%$. An arbitrarily small difference in start time produces the entire $2.5$-point change, so the marginal value of staking one moment earlier becomes unbounded at the boundary. That concentrates a one-time incentive to front-run activation, producing a spike of staking immediately before $t_0$ followed by a lull.

The linear ramp removes the discontinuity. Just after $t_0$ the floor is still approximately $10\%$, and it declines by only $0.0001$ percentage points about every $5.184$ minutes. Staking one day earlier during the window changes the locked-in rate by roughly $0.028$ percentage points rather than the full $2.5$. The incentive to enter at a higher rate is spread smoothly across the 90 days instead of concentrated at a single block, and integrators such as liquid staking protocols have the full window to adjust rather than repricing instantaneously.

### Unchanged Parameters

The following staking parameters are explicitly **not modified** by this proposal:

| Parameter | Value |
|-----------|-------|
| `MaxConsumptionRate` | 12% |
| `MinStakeDuration` | Governed separately by [ACP-273](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/273-reduce-minimum-staking-duration/README.md) |
| `MaxStakeDuration` | 365 days |
| Minimum validator stake | 2,000 AVAX |
| Minimum delegator stake | 25 AVAX |
| Uptime requirement | 90% (per [ACP-267](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/267-uptime-requirement-increase/README.md)) |
| Reward formula structure | Unchanged |

## Backwards Compatibility

This is a non-backwards-compatible change to P-Chain reward calculation. It requires a network upgrade to activate.

The change affects only new staking periods initiated after activation. Validators and delegators with active staking positions at the time of activation continue to receive rewards under the original parameters for the remainder of their existing stake periods. No migration or re-registration is required.

## Reference Implementation

The genesis `MinConsumptionRate` stays at 10%. Two new values drive the transition, a total reduction of 2.5% and a reduction period of 90 days. They are applied in the `GetRewardsCalculator` function (`vms/platformvm/txs/executor/state_changes.go`), which reads the current chain timestamp and decides the rate. Before activation it returns the static 10% calculator. For 90 days after the Helicon upgrade, the P-Chain builds the calculator from a copy of the reward config with `MinConsumptionRate` reduced by the interpolated amount. After the 90-day window, it builds the calculator with the full 2.5% reduction applied, landing exactly on 7.5%.

The reward formula in `vms/platformvm/reward/calculator.go` is unchanged and simply receives the reduced rate. Activation is gated on the Helicon upgrade timestamp in the upgrade configuration. The implementation does not introduce new state transitions or P-Chain transaction types.

## Security Considerations

### Interaction With ACP-273 and ACP-236

This proposal is designed to work in conjunction with [ACP-273](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/273-reduce-minimum-staking-duration/README.md) and [ACP-236](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/236-auto-renewed-staking/README.md). ACP-273 reduces the minimum staking duration to 48 hours. ACP-236 enables zero-friction auto-renewal. Without a change to `MinConsumptionRate`, these two proposals together make minimum-duration compounded stakes economically near-equivalent to long-duration commitments (cost of choosing 2-day rolling vs. 90-day: ~0.50 pp). This proposal provides the complementary reward gradient that ACP-273's own Security Considerations section identified as potentially necessary.

If ACP-273 activates without this proposal, the combination of 48-hour minimum duration and auto-renewal represents a meaningful security regression. This proposal should be considered a companion to ACP-273.

## Open Questions

1. Is 7.5% the right value, or should `MinConsumptionRate` be reduced further?

  The 7.5% floor was selected to produce a meaningful but not extreme shift in the ARR gradient. The expected impact from a wider range of `MinConsumptionRate` changes was analyzed, and 7.5% was found to be a good balance between the goals and potential risks. A lower floor (e.g., 5%) might reduce minimum-duration validator participation to levels that negatively affect decentralization, or lead to staking duration increases that are too large, resulting in higher emissions. It is worth noting that these analyses extrapolate beyond what has been observed historically, so a more conservative change is also a precaution against such model misspecification risks. In addition, the risk to certain ecosystem applications (e.g., LSTs and relevant strategies) that are beyond the scope of the structural model for staking preferences was considered and investigated, and the impact on the key variables was assessed. A moderate change to 7.5% and the phased implementation of the change were both chosen to mitigate the potential unmodeled risk.

2. Should `MaxConsumptionRate` change simultaneously?

  Reducing `MinConsumptionRate` without touching `MaxConsumptionRate` preserves the full reward for 365-day stakers. An alternative framing would reduce `MaxConsumptionRate` as well to slow total issuance, accepting lower returns for long-term stakers in exchange for a slower approach to the supply cap. This proposal does not pursue that direction, but it is worth discussing.

3. Should this proposal activate concurrently with ACP-273?

  Given the Security Considerations above, simultaneous activation with ACP-273 is strongly preferred. The two proposals address the same underlying tension of short staking duration versus network stability, from complementary angles.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
