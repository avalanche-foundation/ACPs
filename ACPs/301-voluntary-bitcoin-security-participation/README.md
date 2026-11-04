| ACP | 301 |
| :--- | :--- |
| **Title** | Voluntary Bitcoin Security Participation Reward Routing |
| **Author(s)** | Relic ([@0x12371C](https://github.com/0x12371C), [agent@thesecretlab.app](mailto:agent@thesecretlab.app)) |
| **Status** | Proposed |
| **Track** | Standards |

## Abstract

This ACP adds an optional, opt-in third bucket to Primary Network validator reward settlement so a validator may route a configured share of **settled rewards** to a Bitcoin Security Participation (BSP) sink.

It extends the cycle-based settlement model in [ACP-236](../236-auto-renewed-staking/README.md). At the end of an auto-renewed staking cycle, reward AVAX is split among:

1. restake (`AutoCompoundRewardShares`),
2. withdraw to the existing rewards owner, and
3. transfer to a validator-configured `BspRewardsOwner`.

**Staked principal is never an input, output, or collateral of this mechanism.** Default `BspRewardShares` is `0`, so validators that do not opt in behave exactly as ACP-236. Consensus, uptime, and stake unlocking do not depend on Bitcoin, oracles, agents, treasuries, or any off-chain BSP facility.

Off-chain BSP policy (treasury operations, bAVAX, hashpower markets) is **out of scope** for Avalanche Network Clients. This ACP only defines how a validator may earmark already-earned reward AVAX at cycle settlement.

## Motivation

Avalanche already has a large set of persistent validator identities, a recurring reward stream, and (via ACP-236) native cycle settlement plus auto-compounding on Fuji. Bridged BTC on Avalanche is **asset interoperability**. It does not finance SHA-256 work, add hashpower demand, or give Avalanche a position in Bitcoin's security economy.

Bitcoin Security Participation is **security interoperability**: a voluntary path from settled validator rewards to a sink that an off-chain facility may use to fund real Bitcoin-directed work. The Avalanche Foundation has described ecosystem value capture and validator economics as dedicated research streams. This ACP supplies the smallest P-Chain primitive that research needs: a deterministic, opt-in routing of **rewards only**.

Without a native settlement hook, any BSP program must scrape withdrawals off-chain, which is unauditable, easy to fake, and easy to confuse with a claim on stake. Putting the split at cycle settlement makes the commitment observable, opt-in, and impossible to satisfy by touching principal.

This ACP is intentionally small. It does not require validators to participate. It does not put Bitcoin, AI, credit, or infrastructure inside consensus. Those belong to a voluntary treasury that consumes BSP outputs, if and only if this routing primitive exists.

Source discussion: [thesecretlab.app/upgradeavax](https://thesecretlab.app/upgradeavax) (*The Long Winter*, Relic, August 2026).

## Specification

### 1. Scope

- Applies only to **Primary Network** auto-renewed validators as defined in ACP-236.
- Does **not** apply to L1 validators, legacy subnet validators, or delegators.
- Does **not** change how principal is locked, unlocked, or weighted.
- Does **not** introduce Bitcoin headers, oracles, or hashpower verification into AvalancheGo.

L1s MAY ignore this ACP. Under [ACP-77](../77-reinventing-subnets/README.md), an L1 MAY later define its own analogue; that is a separate proposal.

### 2. Invariants (MUST)

Implementations MUST enforce all of the following:

1. **Principal isolation.** Stake UTXOs (`StakeOuts`) MUST NOT be inputs to BSP routing. BSP MUST NOT gain a lien, liquidation right, or rehypothecation right over staked AVAX.
2. **Rewards only.** Only AVAX that ACP-236 would already classify as cycle rewards (validation rewards and, if applicable, the validator's commission on delegation rewards) MAY be split toward BSP.
3. **Opt-in default.** If `BspRewardShares` is unset, it MUST be treated as `0`.
4. **Share bound.** `AutoCompoundRewardShares + BspRewardShares <= 1_000_000`.
5. **Fail-closed for consensus.** If the BSP sink cannot be paid (malformed owner, empty owner, or output construction failure), the cycle MUST still complete ACP-236 restake/withdraw/exit logic. BSP output MAY be skipped for that cycle; principal MUST still unlock or restake per ACP-236. Consensus MUST NOT stall on BSP.
6. **No recursive BSP credit in-protocol.** `BspRewardsOwner` outputs are ordinary transferable AVAX. This ACP does not mint a second token or increase stake weight by itself.
7. **No Bitcoin in verification.** Block verification MUST NOT query Bitcoin, hashprice, agents, or any off-chain API.

### 3. New auto-renew config fields

ACP-236's auto-renew config is extended.

`BspRewardShares` uses the same units as `AutoCompoundRewardShares`: millionths of cycle rewards (`percentage * 10_000`). Range `[0, 1_000_000]`.

`BspRewardsOwner` is an `fx.Owner` that receives the BSP share at cycle settlement.

```
AutoCompoundRewardShares + BspRewardShares + WithdrawShares = 1_000_000
```

where `WithdrawShares` is implied (not stored):

```
WithdrawShares = 1_000_000 - AutoCompoundRewardShares - BspRewardShares
```

Examples:

- `(compound=300_000, bsp=0)` — ACP-236 today: restake 30%, withdraw 70%.
- `(compound=300_000, bsp=100_000)` — restake 30%, BSP 10%, withdraw 60%.
- `(compound=0, bsp=0)` — withdraw 100% of rewards; principal-only restake if the validator continues.

### 4. Transaction types

This ACP extends the ACP-236 transaction types. New fields are **appended** so pre-upgrade payloads remain prefix-decodable until the activation timestamp, after which the new codec version is required.

#### 4.1 `AddAutoRenewedValidatorTx` (extended)

Existing ACP-236 fields unchanged, plus:

```golang
type AddAutoRenewedValidatorTx struct {
  // ... ACP-236 fields including AutoCompoundRewardShares and Period ...

  // Percentage of cycle rewards routed to BspRewardsOwner, in millionths.
  // Range [0..1_000_000]. MUST satisfy
  // AutoCompoundRewardShares + BspRewardShares <= 1_000_000.
  BspRewardShares uint32 `serialize:"true" json:"bspRewardShares"`

  // Owner that receives the BSP share of cycle rewards.
  // Ignored when BspRewardShares == 0. MUST be a valid fx.Owner when
  // BspRewardShares > 0.
  BspRewardsOwner fx.Owner `serialize:"true" json:"bspRewardsOwner"`
}
```

Semantic checks at issuance:

- `BspRewardShares <= 1_000_000`
- `AutoCompoundRewardShares + BspRewardShares <= 1_000_000`
- if `BspRewardShares > 0`, `BspRewardsOwner` MUST verify as a non-empty owner
- if `BspRewardShares == 0`, `BspRewardsOwner` MAY be empty

#### 4.2 `SetAutoRenewedValidatorConfigTx` (extended)

Authorized by existing `ValidatorAuthority` / `Auth` as in ACP-236. Adds the same two fields. Updates take effect **only at the end of the current cycle**, matching ACP-236 config semantics.

Setting `BspRewardShares` to `0` stops BSP routing at the next cycle boundary without affecting restake/stop behavior.

#### 4.3 `RewardAutoRenewedValidatorTx`

No new fields. Settlement behavior changes as specified in §5.

### 5. Cycle settlement

At cycle end, `RewardAutoRenewedValidatorTx` runs ACP-236 uptime and reward computation unchanged. Let `R` be the AVAX amount that ACP-236 would restake-or-withdraw as rewards for this cycle (excluding principal).

```
R_compound = floor(R * AutoCompoundRewardShares / 1_000_000)
R_bsp      = floor(R * BspRewardShares / 1_000_000)
R_withdraw = R - R_compound - R_bsp
```

Then:

1. Apply ACP-236 restake of principal (if continuing) plus `R_compound`, including the existing `MaxStakeLimit` cap. Any excess that ACP-236 would withdraw remains a withdrawal, not BSP.
2. If `R_bsp > 0`, create a transferable output of `R_bsp` AVAX to `BspRewardsOwner`.
3. Send `R_withdraw` to `ValidatorRewardsOwner` / `DelegatorRewardsOwner` as ACP-236 already does for the withdrawn remainder.

If the validator is **not** reward-eligible, ACP-236 forced exit applies: principal unlocks, accrued rewards withdraw to the existing rewards owners, and **no BSP output is created**. BSP is a share of earned rewards, not a claim through a failed cycle.

If `Stop requested` (next `Period == 0`), principal unlocks per ACP-236. `R_bsp` is still paid from that cycle's earned rewards if the validator was reward-eligible. Principal is not delayed or withheld for BSP.

### 6. Delegators

Delegators are unchanged from ACP-236: no auto-renewed delegation, delegation must fit inside the validator cycle. This ACP does **not** add a delegator BSP share. Only the validator's own reward slice (validation rewards + commission) is eligible. Delegator rewards continue to `DelegatorRewardsOwner`.

### 7. What this ACP does not specify

The following are **explicitly not** Avalanche Network Client requirements. They MAY be implemented off-chain by a BSP treasury that spends `BspRewardsOwner` outputs:

- purchase of hashpower, miner working capital, or energy contracts
- issuance or auction of bAVAX or any second token
- reserve-coverage ratios, haircuts, or credit/LTV
- autonomous agents or allocation models
- infrastructure or compute-hardware investment

Those policies MUST NOT be encoded in P-Chain verification. If a treasury fails, validators who opted in simply stop receiving a useful sink; they set `BspRewardShares = 0`. Consensus is unaffected.

### 8. Activation

- **Fuji first.** Mainnet MUST NOT activate until Fuji has produced at least one full auto-renewed cycle with both `BspRewardShares = 0` and `BspRewardShares > 0` test validators, and telemetry for those cycles is published.
- Activation is a network upgrade timestamp after ACP-236 is active on that network.
- Pre-upgrade auto-renewed validators are treated as `BspRewardShares = 0` until they issue `SetAutoRenewedValidatorConfigTx` (or a replacement add-tx after re-entry).

### 9. Telemetry

Clients SHOULD log, per `RewardAutoRenewedValidatorTx`:

- `txID`, cycle timestamp
- `R`, `R_compound`, `R_bsp`, `R_withdraw`
- whether BSP output was created or skipped (and skip reason)

This is for operators and researchers. It is not a consensus object.

## Backwards Compatibility

This ACP is backwards compatible for validators that leave `BspRewardShares = 0`.

It is **not** codec-compatible with pre-upgrade ACP-236 transactions once the upgrade activates: ANCs must roll together. Nodes that have not upgraded cannot verify post-upgrade `AddAutoRenewedValidatorTx` / `SetAutoRenewedValidatorConfigTx` bytes.

Impact on the community:

- No forced change to staking APY, uptime rules, or minimum stake.
- No change to delegator UX.
- Wallets and explorers that decode auto-renewed validator config MUST learn two new fields; ignoring them would mis-display reward splits.
- Off-chain BSP programs that today scrape reward withdrawals remain valid; they are not required to use the new sink.

## Reference Implementation

None in AvalancheGo at submission. This ACP is `Proposed`.

A compliant implementation SHOULD:

1. extend ACP-236 config and the two write txs as in §4;
2. implement §5 in `RewardAutoRenewedValidatorTx` execution;
3. add unit tests for share bounds, `BspRewardShares = 0`, skip-on-sink-failure, and forced-exit (no BSP on ineligible cycles);
4. run a local network with two validators (opt-in and opt-out) for at least two cycles and attach telemetry as auxiliary files in this directory.

Off-chain treasury software is not a reference implementation of this ACP.

## Security Considerations

**Consensus isolation.** The hard requirement is §2.5: BSP must not be able to halt cycle settlement. A broken sink, empty owner, or downstream treasury exploit can burn or freeze **only** the AVAX that validators chose to send as rewards. Stake weight and liveness must be invariant to that failure.

**Opt-in social engineering.** A malicious `BspRewardsOwner` can receive opted-in rewards. This is the same class of risk as setting a wrong `ValidatorRewardsOwner` today. Authorization remains `ValidatorAuthority`.

**Share-sum overflow.** Implementations MUST reject configs where the two stored shares sum above `1_000_000` at issuance, not at settlement, so a cycle cannot mint more rewards than `R`.

**No leverage in-protocol.** Allowing BSP outputs to feed back into stake in the same tx would couple treasury policy to validator weight. This ACP forbids that: BSP outputs are withdrawn AVAX, not restaked AVAX. Restake remains solely `R_compound` under ACP-236 caps.

**Bitcoin and oracle risk.** Because verification does not touch Bitcoin, a hashprice crash, pool failure, or agent bug cannot slash or stall Primary Network validation.

**Mandatory BSP.** This ACP MUST NOT be interpreted as requiring non-zero `BspRewardShares`. A later ACP that mandated a floor would be a different, backwards-incompatible proposal and is not authorized here.

## Open Questions

- Should `BspRewardsOwner` be restricted to a secp256k1 / `fx.Owner` that wallets already support, or allowed to be a P-Chain atomic-output consumed by a C-Chain contract via a follow-up ACP?
- Is commission-on-delegation in `R` the right default, or should BSP apply only to self-stake validation rewards?
- After Fuji, is a global per-cycle cap on `R_bsp` (network-wide) useful as a circuit breaker, or is per-validator opt-in sufficient?
- How should explorers display the three-way split without implying that stake is bonded to Bitcoin?

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
