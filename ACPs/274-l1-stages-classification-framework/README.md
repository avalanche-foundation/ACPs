| ACP | 274 |
|:--------------|:------------------------------------------------------------|
| **Title** | L1 Trust Profile Framework |
| **Author(s)** | Giacomo Barbieri ([@ijaack94](https://x.com/ijaack94)) |
| **Status** | Proposed ([Discussion]()) |
| **Track** | Best Practices |

<!-- Track note: This ACP is filed as Best Practices to reflect its non-protocol-changing, informational nature. However, given that it defines a shared methodology, a Board, and a classification process intended for broad ecosystem adoption, the community may wish to consider whether Standards Track is more appropriate. -->

## Abstract

The L1 Trust Profile Framework is a standardised, neutral methodology for describing the trust characteristics of Avalanche Layer 1 (L1) blockchains. As the Avalanche L1 ecosystem expands, participants require clear, consistent, and transparent information about validator control, operator distribution, censorship resistance, and governance transparency.

This framework introduces four independent dimensions:

1. Validator Control
2. Validator Distribution & Bridged Asset Security
3. Censorship Resistance
4. Governance Transparency

Each dimension is scored using a qualitative, reproducible scale with four labels — **Single-Party (SP), Multi-Party (MP), Multi-Stakeholder (MS), or Open (OP)** — all of which describe the nature and concentration of the controlling party along a consistent axis.

The framework does **not** aggregate dimensions into a single score or rank L1s. Instead, each L1 is characterised by its four-dimensional profile (e.g., `Control: MP, Distribution: MS, Censorship: MP, Governance: SP`), allowing different stakeholders to interpret these attributes according to their own needs and risk preferences.

Scoring and methodology evolution are coordinated by an **L1 Trust Profile Board**, composed of stakeholders with diverse and explicitly disclosed interests. All members have voting rights. Conflicts of interest are allowed and expected; transparency and plurality of perspectives are the core safeguards, not the absence of stake.

---

## Motivation

The Avalanche L1 ecosystem has grown from a handful of experimental chains to dozens of production L1s with diverse governance models, validator structures, and security assumptions. Today, it is difficult for validators, builders, users, and integrators to compare these assumptions in a consistent way.

This information gap creates several problems:

### 1. Inconsistent Communication

Different analytics platforms and dashboards describe L1 trust characteristics using inconsistent language and heuristics. One platform may call a chain "decentralised" based on validator count; another may focus on who actually decides validator membership. This lack of shared vocabulary makes cross-L1 comparisons noisy and subjective.

### 2. Validator and Integrator Friction

Validators and integrators who consider participating in or integrating with an L1 must manually inspect:

- Validator management contracts
- Operator concentration
- Bridged asset assumptions
- Governance and upgrade processes

There is no standard way to compare validator trust assumptions across L1s, making it difficult to make informed decisions about validator economics and security assumptions.

### 3. Bridged Asset Risk Visibility

In Avalanche, L1s typically use Interchain Messaging (ICM) and bridges to connect with other chains (e.g., C-Chain). The security of bridged assets depends critically on validator/operator distribution and the way bridging is wired into consensus.

However, these assumptions are rarely surfaced in a structured way. An L1 with concentrated operator weight can, in practice, expose bridged assets to catastrophic risk, but this risk is not visible in current dashboards.

### 4. Governance Evolution Tracking

L1s often evolve over time: starting with highly centralised validator and governance models, then moving toward more distributed control as they mature. There is currently no common framework for:

- Describing where an L1 is "today"
- Showing how its trust assumptions change over time
- Communicating those changes to users and validators

### 5. Need for Neutral, Non-Ranking Information

Different L1s are built for different purposes:

- Enterprise or application-specific chains may prefer strong central control.
- Community/public chains may aim for broad participation and resilience.

The ecosystem needs a way to describe trust assumptions **without** implicitly ranking one approach as "better" or "worse". The goal is *description*, not prescription.

---

## Specification

### 1. Overview

Each Avalanche L1 is described by **four independent dimensions**:

1. Validator Control
2. Validator Distribution & Bridged Asset Security
3. Censorship Resistance
4. Governance Transparency

Each dimension is scored using one of four labels:

| Label | Full Name | Controlling Party |
|---|---|---|
| **SP** | Single-Party | One entity |
| **MP** | Multi-Party | Small known group (2–5) |
| **MS** | Multi-Stakeholder | Broad community / many parties |
| **OP** | Open | No human gating; protocol-determined |

All four labels describe the same axis: **who (if anyone) holds effective authority over this dimension**, from most concentrated (SP) to fully open (OP). This consistency is intentional — each dimension is assessed against the same spectrum.

The framework does **not** compute a single combined score or "stage". It only provides a structured way to express and compare trust assumptions.

---

### 2. Dimension 1: Validator Control

**Question:** Who decides validator entry and exit?

**Definition:** This dimension captures how validator set membership is controlled: by a single actor, a small group, a broader community, or algorithmic rules.

**Scoring:**

- **Single-Party (SP)**
  One entity (person, company, core team, or tightly controlled multisig) has unilateral or near-unilateral authority to add or remove validators.
  Examples:
  - A single EOA or 1-of-1 multisig can change the validator set.
  - A small team with shared keys makes changes without external consent.

- **Multi-Party (MP)**
  Validator set changes require agreement among multiple known entities (e.g., 2–5 parties).
  Examples:
  - 2-of-3 or 3-of-5 multisig controlling validator set management.
  - Council of known entities that must approve changes.

- **Multi-Stakeholder (MS)**
  Validator membership is governed by community decision mechanisms (e.g., governance token voting, validator voting), with no single small group able to unilaterally decide.
  Characteristics:
  - Changes are proposed and decided through on-chain or off-chain community processes.
  - The community, rather than a predefined small group, has final say.

- **Open (OP)**
  Validators can join and exit based on open, algorithmic criteria (e.g., stake, technical requirements) without any approval process.
  Characteristics:
  - No explicit "add/remove" authority at the social layer.
  - Entry/exit is determined by protocol rules (e.g., minimum stake, slashing conditions).

**Notes:**

- This dimension is about **who can alter the validator set**, not how many validators there are.
- For example, a chain with many validators but a single admin key that can remove any of them is **SP**.

---

### 3. Dimension 2: Validator Distribution & Bridged Asset Security

**Question:** How concentrated is validator operation among independent entities, and what does that concentration mean for the security of bridged assets?

**Definition:** This dimension addresses two closely related concerns in a single assessment: (a) how many distinct operators are involved and how much weight each controls, including geographic and infrastructure distribution; and (b) what those distributions mean for the security of assets bridged to/from the L1, especially under ICM's 67% BLS signing threshold.

These two concerns are assessed together because operator concentration *is* the primary driver of bridged asset risk on Avalanche. Separating them would produce largely redundant scores. Where bridging assumptions diverge materially from operator distribution (e.g., due to custom bridging paths), that divergence is noted in the Board's published rationale.

**Scoring:**

- **Single-Party (SP)**
  A single operator runs all or almost all of the validator weight, creating a single point of failure for both liveness and bridged assets.
  Examples:
  - One company operates ≥ 75% of validator weight.
  - A single operator can effectively steer ≥ 67% of BLS signing power.
  - No meaningful limits on operator concentration; no monitoring of bridged asset risk.
  In this setup, compromise or collusion of a single operator could plausibly lead to total loss of bridged assets.

- **Multi-Party (MP)**
  A small number of entities (e.g., 2–5) operate the majority of validator weight. Material concentration remains.
  Characteristics:
  - No single operator controls ≥ 67% of weight, but one or more operators control > 20%.
  - A small coalition could reach the 67% ICM threshold.
  - Some monitoring of concentration may exist, but policies for responding to it are limited.

- **Multi-Stakeholder (MS)**
  A larger, more distributed operator set. Indicative (not rigid) criteria:
  - At least ~6 independent operators.
  - No single operator controls ≥ 20% of validator weight.
  - No single data centre hosts ≥ 20% of weight.
  - No single cloud provider hosts ≥ 20% of weight.
  - Changes in concentration are observed and can be acted upon.
  Under these constraints, the risk that a single operator compromise leads to catastrophic loss of bridged assets is significantly reduced.

- **Open (OP)**
  Highly diversified operator set with active monitoring. Indicative criteria:
  - 10+ independent operators.
  - High Nakamoto coefficient (e.g., > 40).
  - Largest operator controls < 15% of weight.
  - No single data centre or cloud provider hosts ≥ 20% of weight.
  - Operator composition evolves over time without being locked into a small set.
  - Ongoing, public monitoring of operator weight and infrastructure distribution.
  - Bridged-asset risk assumptions and mitigation strategies are clearly documented.

**Notes:**

- "Independent operator" refers to distinct legal or organisational entities, not just distinct node IDs.
- Exact numeric thresholds are guidelines; the Board documents and updates concrete thresholds over time.
- This dimension is **Avalanche-specific** in that it explicitly accounts for ICM's reliance on 67% of validator weight / BLS signatures and the corresponding risk to bridged assets.

---

### 4. Dimension 3: Censorship Resistance

**Question:** Can a controlling party selectively exclude or delay transactions?

**Definition:** This dimension captures how difficult it is for a single entity or small coalition to systematically prevent specific transactions or users from being included in blocks. It is distinct from Validator Control (which describes who manages the validator set) and focuses instead on the practical ability to censor at the transaction level.

**Scoring:**

- **Single-Party (SP)**
  A single validator or controlling entity can effectively censor transactions.
  Examples:
  - One operator controls enough weight to unilaterally exclude transactions.
  - A single entity can instruct all validators to filter specific addresses or transaction types without consequence.

- **Multi-Party (MP)**
  A small, known coalition (2–5 parties) could coordinate to censor transactions.
  Characteristics:
  - No single party alone can censor, but collusion among a small set is practically feasible.
  - Validator set is small enough that coordinated filtering is difficult to detect and counter in real time.

- **Multi-Stakeholder (MS)**
  Censorship requires coordination across a large, diverse group of independent operators.
  Characteristics:
  - No small coalition can reach the threshold required to exclude transactions.
  - Coordination at the scale required would be visible and contestable.
  - Users or L1 teams have practical recourse if censorship is attempted.

- **Open (OP)**
  The validator set is sufficiently open and distributed that targeted, sustained censorship is impractical.
  Characteristics:
  - Permissionless or near-permissionless validator entry makes it difficult to maintain a censoring coalition over time.
  - Any censored party can seek alternative validators or routes.
  - No single legal or organisational entity controls enough of the validator set to impose censorship without observable defection.

**Notes:**

- This dimension is particularly relevant for L1s used by DeFi applications, where transaction-level censorship can have direct financial consequences.
- Censorship resistance is related to, but distinct from, operator diversity: an L1 may have many operators but still be censorship-vulnerable if those operators share a common legal jurisdiction or infrastructure provider that could coerce filtering.

---

### 5. Dimension 4: Governance Transparency

**Question:** How transparent and predictable is the process for upgrading the L1's critical parameters and contracts?

**Definition:** This dimension expresses the visibility and predictability of governance decisions that affect security assumptions (e.g., changes to validator management, bridging logic, protocol parameters).

**Scoring:**

- **Single-Party (SP)**
  - Upgrades can be made unilaterally by a single entity.
  - No formal timelock or public notice required.
  - Governance is largely opaque or ad hoc.

- **Multi-Party (MP)**
  - Upgrades require multi-party agreement (e.g., multisig, council).
  - Short but non-zero timelocks (e.g., 1–7 days).
  - Governance processes are documented but still primarily rely on a small group.

- **Multi-Stakeholder (MS)**
  - Upgrades require some form of community decision (e.g., token-holder voting, validator voting).
  - Notice and timelocks are longer (e.g., 7–14 days).
  - Governance rules and history are publicly documented and generally predictable.

- **Open (OP)**
  - Long timelocks (e.g., ≥ 14 days) or partially immutable core components.
  - Governance changes are transparent, on-chain, and widely visible.
  - Users, validators, and integrators have sufficient time to react (upgrade, exit, or adapt) before changes become active.

**Notes:**

- This dimension is not about **who** has power (that is Dimension 1), but **how visible and predictable** governance actions are.
- For example, an L1 can have SP Validator Control but MS Governance Transparency if the single party commits to strong timelocks and full public disclosure.

---

### 6. Multi-Dimensional Profiles (No Ranking)

The framework **does not derive** any "Stage", "Level", or ordinal ranking from these dimensions.

Each L1 is characterised by a 4-tuple:

- `Validator Control: SP/MP/MS/OP`
- `Validator Distribution & Bridged Asset Security: SP/MP/MS/OP`
- `Censorship Resistance: SP/MP/MS/OP`
- `Governance Transparency: SP/MP/MS/OP`

Example:

```text
L1 X:
  Validator Control:                          MP
  Validator Distribution & Bridged Security:  MS
  Censorship Resistance:                      MP
  Governance Transparency:                    MS
```

It is up to validators, integrators, users, and L1 teams to interpret this profile based on their use cases and risk appetite. The framework is **descriptive, not prescriptive**.

No composite "score", "grade", or "tier" is produced by this ACP.

---

## L1 Trust Profile Board

### 1. Purpose

The L1 Trust Profile Board coordinates:

- Maintenance of dimension definitions and rubrics.
- Application of the framework to concrete L1s.
- Publication of scores and reasoning.
- Iterative refinement of the framework based on community feedback.

The Board's role is **procedural and methodological**, not regulatory: it provides a consistent process for classification, not enforcement.

### 2. Composition and Voting

- The Board consists of **5–7 members**.
- All members have **equal voting rights**.
- Members are expected to represent **diverse stakeholder perspectives**, e.g.:
  - L1 builders
  - Validators or infra providers
  - Analytics / research contributors
  - End-user or community representatives
  - Avalanche Foundation (or ecosystem steward) representatives

**Conflicts of interest are allowed and expected.** Members are chosen precisely because they bring real stakes, constraints, and goals into the discussion.

The primary safeguard is **transparency and plurality**, not artificial neutrality.

### 3. Interest Disclosure

Rather than prohibiting conflicts, this ACP requires disclosure:

- Each Board member publishes:
  - L1s they are directly involved with (builder, employee, advisor, validator, major tokenholder, etc.).
  - Projects or platforms they maintain (e.g., analytics tools).
- For each L1 classification, the record notes which members have material involvement with that L1.

Members **may** voluntarily recuse themselves from voting on a given L1 if they feel it would increase confidence in the process, but recusal is not mandatory.

### 4. Initial Board Selection (Bootstrap)

To address the governance cold-start problem, the following bootstrap process applies:

1. Within **30 days** of this ACP's adoption, the Avalanche Foundation nominates an initial slate of 5–7 Board members following the seat composition guidelines above.
2. The community ratifies (or requests revisions to) this slate via a Snapshot vote within **60 days** of adoption.
3. If the community does not ratify within 60 days, the Foundation may seat the Board on an interim basis while a second ratification round is organised.
4. This bootstrap Board serves for one full term (1 year), after which normal community nomination and election procedures apply.

The bootstrap process is a pragmatic starting point, not a permanent feature. The intent is to ensure the Board is operational while community governance processes are being established.

### 5. Ongoing Selection and Term

After the initial term, Board members are selected via community process (e.g., GitHub nominations + Snapshot voting):

- 1–3 seats reserved for L1 builders.
- 1–2 seats reserved for validators / infra.
- 1–2 seats for independent researchers / analytics.
- 1 seat optionally reserved for a Foundation-aligned steward.

Recommended terms:

- 1 year, with the option to be re-elected.
- Staggered rotation so that not all seats change at once.

### 6. Scoring Process

The Board uses a public, documented process for classifying each L1:

1. **Information Gathering**
   - Collect on-chain data, documentation, and statements from the L1 team.
   - Optionally, ask the L1 to submit a self-assessment with evidence.

2. **Individual Draft Scoring**
   - Each member independently proposes SP/MP/MS/OP for each dimension, with short rationale.

3. **Discussion and Convergence**
   - The Board meets (publicly or with published minutes) to:
     - Compare rationales.
     - Resolve misunderstandings and edge cases.
     - Move toward a converged score for each dimension.

4. **L1 Review**
   - The draft profile is sent to the L1 team for comment (e.g., 1–2 weeks window).
   - The L1 can:
     - Point out factual errors.
     - Provide missing evidence.
     - Offer clarifications.
   - The L1 cannot "veto" a score, but its input must be taken into account and documented.

5. **Finalization and Publication**
   - The Board finalises the four dimension scores.
   - Votes for each member (per dimension) are published.
   - A short justification and links to evidence are published.
   - Any optional recusals are noted.

6. **Updates Over Time**

   **Full review:** At least once a year, all active L1s are re-assessed at a high level.

   **Change-triggered review:** When an L1 makes a material change to validator policy, operator composition, bridging architecture, or governance, it may request (or the community may petition for) an expedited re-assessment. The Board commits to initiating change-triggered reviews within **30 days** of a valid request. Material changes include but are not limited to:
   - Changes to the validator management contract or its signers.
   - Shifts in operator weight concentration exceeding 10 percentage points for any single operator.
   - Modifications to bridging architecture or ICM integration.
   - Changes to governance timelocks or upgrade mechanisms.

### 7. Decisions and Voting Thresholds

- For adopting or changing **rubrics / definitions**:
  - A supermajority (e.g., ≥ 2/3) of Board members is recommended.

- For **classifying an L1** on the four dimensions:
  - A simple majority is sufficient, but minority positions may be recorded as "alternate views" if they differ materially.

- All votes are recorded and published, including dissenting opinions.

---

## Implementation Guidance

### 1. For Analytics Platforms

Platforms that wish to implement this framework should:

- Display, for each L1:
  - The four dimension scores (SP/MP/MS/OP).
  - A short explanation of what each dimension means.
  - A link to the Board's reasoning and evidence for that L1.

- Avoid:
  - Turning the four-dimensional profile into a single number, letter grade, or ranking.
  - Presenting one pattern as inherently superior.

- May provide:
  - Comparison tools that show profiles side by side.
  - Filters (e.g., "show L1s with MS or OP in Validator Distribution").
  - Historical views (how an L1's profile changed over time).

### 2. For L1 Projects

L1 teams can use the framework as:

- A **self-assessment tool**, even before Board classification.
- A **roadmap aid**, when planning changes to validator policies, bridged asset assumptions, or governance.
- A **communication tool**, to explain to users and integrators:
  - "Here is where we are."
  - "Here is where we plan to go, and why."

The framework does not demand that all L1s aim for the same pattern. Different goals justify different profiles.

---

## Security Considerations

This ACP does not change core protocol behaviour. Its security relevance is indirect, via improved understanding and communication.

Key points:

- **Validator Distribution & Bridged Asset Security** makes visible the risk that a small operator set could compromise bridged assets, especially under ICM's 67% threshold.
- **Validator Control** and **Censorship Resistance** dimensions highlight where liveness, exclusion, and censorship risks are concentrated.
- **Governance Transparency** helps integrators understand how much advance notice they receive before assumption changes.

Because the Board is explicitly allowed to include members with stakes and roles, the primary security mechanism is **transparency**: all votes, rationales, and roles are public. The community can then adopt or ignore classifications based on trust in the process and people.

---

## Reference Implementation

There is no single "reference implementation" requirement. Any platform can:

- Parse the ACP's dimension definitions.
- Apply them manually or via community-driven scoring.
- Display the resulting profiles.

The L1 Trust Profile Board's outputs can act as a canonical reference if the community adopts them.

---

## Backwards Compatibility

This ACP is informational and does **not**:

- Change consensus rules.
- Modify ICM semantics.
- Require any on-chain upgrade.

Existing L1s may ignore or adopt the framework. No protocol-level compatibility constraints exist.

---

## Open Questions

1. Are the qualitative cutoffs for SP/MP/MS/OP clear enough, or should more concrete numeric examples be standardised up front?
2. Should there be optional "viewpoints" (e.g., "validator view", "DeFi user view") that weigh dimensions differently, while still not collapsing them into a single score?
3. How should the Censorship Resistance dimension account for legal/jurisdictional coercion, which may not be visible on-chain?
4. Should there be a formal mechanism for third parties (e.g., security researchers) to trigger a change-triggered review, beyond L1 teams and the community?
5. How should the framework evolve to handle L1s that use novel validator/bridging designs not yet contemplated in the current rubrics?

---

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
