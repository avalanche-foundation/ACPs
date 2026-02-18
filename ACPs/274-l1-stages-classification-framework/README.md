| ACP | 274 |
|:--------------|:------------------------------------------------------------|
| **Title** | L1 Stages Classification Framework |
| **Author(s)** | Giacomo Barbieri ([@ijaack94](https://x.com/ijaack94)) |
| **Status** | Proposed  ([Discussion]()) |
| **Track** | Best Practices |

## Abstract

The L1 Stages Classification Framework is a standardized, neutral methodology for describing the trust characteristics of Avalanche Layer 1 (L1) blockchains. As the Avalanche L1 ecosystem expands, participants require clear, consistent, and transparent information about validator control, operator diversity, bridged asset security, and governance transparency.

This framework introduces four independent dimensions:

1. Validator Control  
2. Validator Operator Diversity  
3. Bridged Asset Security  
4. Governance Transparency  

Each dimension is scored using a qualitative, reproducible scale: **Single-Party (SP), Multi-Party (MP), Community Governance (CG), or Permissionless (P)**.

The framework does **not** aggregate dimensions into a single score or rank L1s. Instead, each L1 is characterized by its four-dimensional profile (e.g., `Control: MP, Diversity: CG, Bridged Security: MP, Governance: SP`), allowing different stakeholders to interpret these attributes according to their own needs and risk preferences.

Scoring and methodology evolution are coordinated by an **L1 Stages Classification Board**, composed of stakeholders with diverse and explicitly disclosed interests. All members have voting rights. Conflicts of interest are allowed and expected; transparency and plurality of perspectives are the core safeguards, not the absence of stake.

---

## Motivation

The Avalanche L1 ecosystem has grown from a handful of experimental chains to dozens of production L1s with diverse governance models, validator structures, and security assumptions. Today, it is difficult for validators, builders, users, and integrators to compare these assumptions in a consistent way.

This information gap creates several problems:

### 1. Inconsistent Communication

Different analytics platforms and dashboards describe L1 trust characteristics using inconsistent language and heuristics. One platform may call a chain "decentralized" based on validator count; another may focus on who actually decides validator membership. This lack of shared vocabulary makes cross-L1 comparisons noisy and subjective.

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

L1s often evolve over time: starting with highly centralized validator and governance models, then moving toward more distributed control as they mature. There is currently no common framework for:

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
2. Validator Operator Diversity  
3. Bridged Asset Security  
4. Governance Transparency  

Each dimension is scored using one of four labels:

- **SP** – Single-Party  
- **MP** – Multi-Party  
- **CG** – Community Governance  
- **P** – Permissionless  

These labels are shorthand for specific, documented criteria. The framework does **not** compute a single combined score or "stage". It only provides a structured way to express and compare trust assumptions.

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

- **Community Governance (CG)**  
  Validator membership is governed by community decision mechanisms (e.g., governance token voting, validator voting), with no single small group able to unilaterally decide.  
  Characteristics:
  - Changes are proposed and decided through on-chain or off-chain community processes.
  - The community, rather than a predefined small group, has final say.

- **Permissionless (P)**  
  Validators can join and exit based on open, algorithmic criteria (e.g., stake, technical requirements) without any approval process.  
  Characteristics:
  - No explicit "add/remove" authority at the social layer.
  - Entry/exit is determined by protocol rules (e.g., minimum stake, slashing conditions).

**Notes:**

- This dimension is about **who can alter the validator set**, not how many validators there are.
- For example, a chain with many validators but a single admin key that can remove any of them is **SP**.

---

### 3. Dimension 2: Validator Operator Diversity

**Question:** How concentrated is validator operation among independent entities?

**Definition:** This dimension addresses how many distinct operators are involved and how much weight each controls, including geographic and infrastructure distribution.

**Scoring:**

- **Single-Party (SP)**  
  A single operator runs all or almost all of the validator weight, or a dominant share creating a single point of failure.  
  Examples:
  - One company operates ≥ 75% of validator weight.
  - All validators are effectively controlled by the same legal entity.

- **Multi-Party (MP)**  
  A small number of entities (e.g., 2–5) operate the majority of validator weight.  
  Characteristics:
  - One or more operators may control > 20–40% of weight.
  - Failures or compromises of a small coalition can significantly impact liveness and security.

- **Community Governance (CG)**  
  A larger set of independent operators. Indicative (not rigid) criteria:  
  - At least ~6 independent operators.  
  - No single operator controls ≥ 20% of weight.  
  - Some geographic and infrastructure diversification is present.

- **Permissionless (P)**  
  Highly diversified operator set. Indicative criteria:  
  - 10+ independent operators.  
  - High Nakamoto coefficient (e.g., > 40).  
  - Largest operator controls < 15% of weight.  
  - No single data center or cloud provider hosts ≥ 20% of weight.  
  - Operator composition evolves over time without being locked into a small set.

**Notes:**

- "Independent operator" refers to distinct legal or organizational entities, not just distinct node IDs.
- Exact numeric thresholds are guidelines; the Board documents and updates concrete thresholds over time.

---

### 4. Dimension 3: Bridged Asset Security

**Question:** How robust is the L1's security posture with respect to bridged assets, given ICM's 67% BLS threshold and other bridging assumptions?

**Definition:** This dimension focuses on the **risk to assets bridged to/from the L1**, especially via ICM. It expresses how operator distribution, validator management, and related practices affect the potential for loss or unauthorized movement of bridged assets.

While many implementation details matter, this dimension is rooted in a few critical questions:

- Could a single operator (or very small group) effectively control ≥ 67% of the validator weight (or equivalent for the bridging mechanism)?
- Are there minimums and limits on operator and infrastructure concentration?
- Are these conditions monitored and documented?

**Scoring:**

- **Single-Party (SP)**  
  Conditions such as:
  - A single operator can effectively steer ≥ 67% of validator weight (or BLS signing power).
  - No meaningful limits on operator concentration.
  - No public monitoring or documentation of these risks.  
  In this setup, compromise or collusion of a single operator could plausibly lead to total loss of bridged assets.

- **Multi-Party (MP)**  
  Improvements over SP, but still material concentration:  
  - No single operator controls ≥ 67% of weight, but one or more operators control > 20%.  
  - Operator set is small (e.g., 2–5), and a small coalition could reach 67%.  
  - Some monitoring of concentration may exist, but policies for responding to it are limited.

- **Community Governance (CG)**  
  Incorporates the **critical security requirements** in practice:  
  Typically:
  - No single operator controls ≥ 20% of validator weight.  
  - L1 has at least 5 validators, and preferably significantly more.  
  - No single data center hosts ≥ 20% of weight.  
  - No single cloud provider hosts ≥ 20% of weight.  
  - Changes in concentration are observed and can be acted upon by governance or operators.  
  Under these constraints, the risk that a single operator compromise leads to catastrophic loss of bridged assets is significantly reduced.

- **Permissionless (P)**  
  Strongest expression of bridged asset security posture:  
  - Meets or exceeds CG conditions.  
  - Has ongoing, public monitoring of operator weight, infrastructure distribution, and changes over time.  
  - Bridged-asset risk assumptions and mitigation strategies are clearly documented.  
  - If multiple bridging paths exist, at least one operates with these guarantees in mind (e.g., bridging paths that rely on the most decentralized subset of operators).

**Notes:**

- This dimension is **Avalanche-specific** in that it explicitly accounts for ICM's reliance on 67% of validator weight / BLS signatures and the corresponding risk to bridged assets.
- It is intentionally overlapping with Operator Diversity, but focuses narrowly on **what those distributions mean for bridged asset risk**, not liveness or censorship in general.

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
  - Upgrades require multi-party agreement (e.g., multi-sig, council).  
  - Short but non-zero timelocks (e.g., 1–7 days).  
  - Governance processes are documented but still primarily rely on a small group.

- **Community Governance (CG)**  
  - Upgrades require some form of community decision (e.g., token-holder voting, validator voting).  
  - Notice and timelocks are longer (e.g., 7–14 days).  
  - Governance rules and history are publicly documented and generally predictable.

- **Permissionless (P)**  
  - Long timelocks (e.g., ≥ 14 days) or partially immutable core components.  
  - Governance changes are transparent, on-chain, and widely visible.  
  - Users, validators, and integrators have sufficient time to react (upgrade, exit, or adapt) before changes become active.

**Notes:**

- This dimension is not about **who** has power (that is Dimension 1), but **how visible and predictable** governance actions are.
- For example, an L1 can have SP Validator Control but CG Governance Transparency if the single party commits to strong timelocks and full transparency.

---

### 6. Multi-Dimensional Profiles (No Ranking)

The framework **does not derive** any "Stage", "Level", or ordinal ranking from these dimensions.

Each L1 is simply characterized by a 4-tuple:

- `Validator Control: SP/MP/CG/P`  
- `Validator Operator Diversity: SP/MP/CG/P`  
- `Bridged Asset Security: SP/MP/CG/P`  
- `Governance Transparency: SP/MP/CG/P`  

Example:

```text
L1 X:
  Validator Control:           MP
  Validator Operator Diversity: CG
  Bridged Asset Security:      MP
  Governance Transparency:     CG
```

It is up to validators, integrators, users, and L1 teams to interpret this profile based on their use cases and risk appetite. The framework is **descriptive, not prescriptive**.

No composite "score", "grade", or "tier" is produced by this ACP.

---

## L1 Stages Classification Board

### 1. Purpose

The L1 Stages Classification Board coordinates:

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

### 4. Selection and Term

The community process (e.g., GitHub nominations + Snapshot voting) chooses Board members. A possible pattern:

- 1–3 seats reserved for L1 builders.  
- 1–2 seats reserved for validators / infra.  
- 1–2 seats for independent researchers / analytics.  
- 1 seat optionally reserved for a Foundation-aligned steward.

Terms, rotations, and selection mechanics are left flexible to the community, but recommended:

- Terms of 1 year, with the option to be re‑elected.  
- Staggered rotation so that not all seats change at once.

### 5. Scoring Process

The Board uses a public, documented process for classifying each L1:

1. **Information Gathering**  
   - Collect on-chain data, documentation, and statements from the L1 team.  
   - Optionally, ask the L1 to submit a self-assessment with evidence.

2. **Individual Draft Scoring**  
   - Each member independently proposes SP/MP/CG/P for each dimension, with short rationale.

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
   - The Board finalizes the four dimension scores.  
   - Votes for each member (per dimension) are published.  
   - A short justification and links to evidence are published.  
   - Any optional recusals are noted.

6. **Updates Over Time**  
   - When an L1 makes a meaningful change (e.g., new validator policy, new governance model, large shift in operator distribution), it or the community may request re-assessment.
   - At least once a year, all active L1s are re‑reviewed at a high level.

### 6. Decisions and Voting Thresholds

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
  - The four dimension scores (SP/MP/CG/P).  
  - A short explanation of what each dimension means.  
  - A link to the Board's reasoning and evidence for that L1.

- Avoid:
  - Turning the four-dimensional profile into a single number, letter grade, or ranking.
  - Presenting one pattern as inherently superior ("this is the best stage").

- May provide:
  - Comparison tools that show profiles side by side.
  - Filters (e.g., "show L1s with CG or P in Bridged Asset Security").
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

This ACP does not change core protocol behavior. Its security relevance is indirect, via improved understanding and communication.

Key points:

- **Bridged Asset Security dimension** makes visible the risk that a small operator set could compromise bridged assets, especially under ICM's 67% threshold.  
- **Operator Diversity** and **Validator Control** dimensions highlight where liveness and censorship risks are concentrated.  
- **Governance Transparency** dimension helps integrators understand how much advance notice they get before assumption changes.

Because the Board is explicitly allowed to include members with stakes and roles, the primary security mechanism is **transparency**: all votes, rationales, and roles are public. The community can then adopt or ignore classifications based on trust in the process and people.

---

## Reference Implementation

There is no single "reference implementation" requirement. Any platform can:

- Parse the ACP's dimension definitions.
- Apply them manually or via community-driven scoring.
- Display the resulting profiles.

The L1 Stages Classification Board's outputs can act as a canonical reference if the community adopts them.

---

## Backwards Compatibility

This ACP is informational and does **not**:

- Change consensus rules.  
- Modify ICM semantics.  
- Require any on-chain upgrade.

Existing L1s may ignore or adopt the framework. No protocol-level compatibility constraints exist.

---

## Open Questions

1. Are the four dimensions sufficient, or should additional dimensions (e.g., censorship resistance, validator exclusivity) be considered in future revisions?  
2. Are the qualitative cutoffs for SP/MP/CG/P clear enough, or should more concrete numeric examples be standardized up front?  
3. Should there be optional "viewpoints" (e.g., "validator view", "user view") that weigh dimensions differently, while still not collapsing them into a single score?  
4. How frequently should the Board revisit the rubrics, and should changes require wider community ratification (e.g., Snapshot votes)?  
5. What is the best process to keep the Bridged Asset Security dimension in sync with evolving ICM and bridging designs?

---

## License

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
