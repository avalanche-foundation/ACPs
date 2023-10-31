```text
ACP: <TODO>
Title: Subnet-Only Validators
Author(s): Patrick O'Grady <patrickogrady.xyz>
Discussions-To: <TODO>
Status: Proposed
Track: Standards
```

## Abstract

Remove the requirement that Subnet Validators must validate the Primary Network and instead require bond...

Validators without restricting their ability to send or verify Avalanche Warp Messages (AWM). Preview a future transition to Pay-As-You-Go Subnet Validation and $AVAX-Augmented Subnet Security.

## Motivation

Each node operator must stake at least 2000 $AVAX (~$20k at the time of writing) to first become a Primary Network validator before they qualify to become a Subnet Validator. All Subnet Validators, to satisfy their role as Primary Network Validators, must also allocate 8 AWS vCPU, 16 GB RAM, and 1 TB storage to sync the entire Primary Network (X-Chain, P-Chain, and C-Chain) and participate in its consensus, in addition to whatever resources are required for each Subnet they are validating.

Although the fee paid to the Primary Network to operate a Subnet does not go up with the amount of activity on the Subnet, the fixed, upfront cost of setting up a Subnet Validator on the Primary Network deters new projects that prefer smaller, even variable, costs until demand is observed. _Unlike L2s that pay some increasing fee (usually denominated in units per transaction byte) to an external chain for data availability and security as activity scales, Subnets provide their own security/data availability and the only cost operators must pay from processing more activity is the hardware cost of supporting additional load._

Regulated entities that are prohibited from validating permissionless, smart contract-enabled blockchains (like the C-Chain) can’t launch a Subnet because they can’t opt-out of Primary Network Validation. This deployment blocker prevents a large cohort of Real World Asset (RWA) issuers from bringing unique, valuable tokens to the Avalanche Ecosystem (that could move between C-Chain <> Subnets using AWM/Teleporter).

TODO: fault isolation

## Specification

* Remove the requirement for a Subnet Validator to also be a Primary Network Validator but do not prevent it (current behavior not deprecated)
* Introduce a new transaction type on the P-Chain for Subnet Validators that only want to validate a Subnet (`AddSubnetOnlyValidatorTx`)
* Require Subnet-Only Validators to bond X \$AVAX per validation per Subnet (to account for P-Chain Load)

Without the requirement to validate the Primary Network, the need for Subnet Validators to instantiate and sync the C-Chain and X-Chain can be relaxed. Subnet Validators will only be required to sync the P-chain to track any validator set changes in their Subnet and to support Cross-Subnet communication via AWM (see “Primary Network Partial Sync” mode introduced in [Cortina 8](https://github.com/ava-labs/avalanchego/releases/tag/v1.10.8)). The lower resource requirement in this "minimal mode" will provide Subnets with greater flexibility of validation hardware requirements as operators are not required to reserve any resources for C-Chain/X-Chain operation.

_The value of the required "bond" (X) is open for debate. To avoid impacting network stability, I think it should be at least 250-750 \$AVAX. To set this "bond" lower, I think the PlatformVM should be futher optimized (assumes that lower fees lead to a corresponding increase in Subnets)._

### Future Directions
#### Improve PlatformVM Efficiency + Pay-As-You-Go Subnet Validation Fees

* Replace Proposal Block-based Subnet reward voting with BLS Multi-Signature votes from Subnet Validators
* Transition Subnet Validation fees to a dynamically priced, continuously charged mechanism that doesn't require locking/staking large amounts of \$AVAX upfront _(this could be broken out as a separate step but makes sense to include in the voting code refactor, so kept here)_

To increase the P-Chain capacity for processing Subnet reward distributions (which should allow fees to be parameterized more aggressively), we first should replace Proposal Block-based voting (1 Subnet reward votes per block) with BLS Multi-Signature voting (N Subnet reward votes per block). _In a future state, this mechanism may allow Subnets to implement arbitrary reward mechanisms by adding an "amount" to this signed message instead of relying on the \$AVAX reward curve that is currently used by all Elastic Subnets. @stephenbuttolph has a number of ideas about this approach 👀._

Once this vote processing change is implemented, it would be possible to just transition to a lower required "bond" amount, but many (myself included) think that it would be more competitve to transition to a dynamically priced, continuous payment mechanism. This new mechanism would target some $Y nAVAX fee that would be paid by each Subnet Validator per Subnet per second (pulling from a "Subnet Validator's Account") instead of requiring a large upfront lockup of \$AVAX.

_The rate of nAVAX/second should be set by the demand for validating Subnets on Avalanche compared to some usage target per Subnet and across all Subnets. This rate should be locked for each Subnet Validation period to ensure operators are not subject to suprise costs if demand rose significantly over time. All fees would still be paid to the network and burned, like all other P-Chain, X-Chain, and C-Chain transactions. The optimization work outlined here should allow the min rate to be set as low as ~512-4096 nAVAX/second (or 1.3-10.6 \$AVAX/month)._

#### $AVAX-Augmented Subnet Security

* Enable unstaked \$AVAX to be locked to a Subnet Validator on an Elastic Subnet and slashed if said Subnet Validator commits an attributable fault (i.e. proposes/signs conflicting blocks/AWM payloads)
* Reward locked \$AVAX associated with Subnet Validators that were not slashed with Elastic Subnet staking rewards

Currently, the only way to secure an Elastic Subnet is to stake its staking token. Many have requested the option to use \$AVAX for this token, however, this could easily allow an adversary to takeover small Elastic Subnets (where the amount of \$AVAX staked may be much less than the circulating supply).

\$AVAX-Augmented Subnet Security would allow anyone holding \$AVAX to lock it to specific Subnet Validators and earn Elastic Subnet reward tokens for supporting honest participants. Recall, all stake management on the Avalanche Network (even for Subnets) occurs on the P-Chain. Thus, staked tokens (\$AVAX and/or custom staking tokens used in Elastic Subnets) and stake weights (used for AWM verification) are secured by the full \\$AVAX stake of the Primary Network. \$AVAX-Augmented Subnet Security, like staking, would be implemented on the P-Chain and enjoy the full security of the Primary Network. This approach means locking \$AVAX occurs on the Primary Network (no need to transfer \$AVAX to a Subnet, which may not be secured by meaningful value yet) and proofs of malicious behavior are processed on the Primary Network (a colluding Subnet could otherwise chose not to process a proof that would lead to their "lockers" being slashed).

_This approach is comparable to the idea of using \$ETH to secure DA on [EigenLayer](https://www.eigenlayer.xyz/) (without reusing stake) or \$BTC to secure Cosmos Zones on [Babylon](https://babylonchain.io/) (but not using an external ecosystem)._


## Backwards Compatibility

* All existing Subnet Validators can continue validating both the Primary Network and whatever Subnets they are validating. This change would just provide a new option for Subnet Validators that allows them to sacrifice their staking rewards for a smaller upfront $AVAX commitment and lower infrastructure costs.
* Support for Phase 1 would require adding a new transaction type to the P-Chain. This new transaction would not be compatible with Avalanche Cortina and require a mandatory Avalanche Network upgrade.

## Reference Implementation

`TODO` (once considered "Implementable")

## Security Considerations

* Any Subnet Validator running in "Partial Sync Mode" will not be able to verify Atomic Imports on the P-Chain and will rely entirely on Primary Network consensus to only accept valid P-Chain blocks.
* High-throughput Subnets will be better isolated from the Primary Network and should improve its resilience (i.e. surges of traffic on some Subnet cannot destabilize a Primary Network Validator).

## Open Questions

* Bond amount?

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

