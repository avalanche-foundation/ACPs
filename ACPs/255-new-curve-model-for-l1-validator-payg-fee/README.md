# ACP-255: New Curve Model for L1 Validator PAYG Fee

| ACP | 255 |
|:-------|:-------------|
| **Title** | New Curve Model for L1 Validator PAYG Fee |
| **Author** | Giacomo Barbieri [(@ijaack94)](https://x.com/ijaack94)|
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

This ACP proposes a fundamental redesign of Avalanche's L1 validator fee economics through three coordinated modifications:

1. **Doubled Base Commission Rate:** Increase M from 512 to 1,024 nAVAX/sec (2.65 AVAX/month vs. current 1.33)
2. **L1-Size-Dependent Fee Multiplier:** Maximum premium of 50 AVAX/month for single-validator L1s, creating powerful decentralization incentives
3. **Gaussian Network Factor:** Elegant bell curve centered at 10,000 validators with 3.84x factor peak

**Economic Impact:**
- Current network (800 validators): **8.5x higher burn** (1,064 → 9,040 AVAX/month)
- At 2,000 validators: **9.9x higher burn** (2,660 → 26,360 AVAX/month)
- At 5,000 validators: **14.7x higher burn** (6,650 → 97,800 AVAX/month)
- **At 10,000 validators: 20.8x higher burn (PEAK)** (13,300 → 266,200 AVAX/month)
- At 20,000 validators: **8.2x higher burn** (26,600 → 218,000 AVAX/month)

## Motivation

The current ACP-77 flat-fee model provides insufficient burn (~1,064 AVAX/month) and creates no economic incentive for validator decentralization. ACP-255 implements a smooth Gaussian fee curve that:

- Funds protocol development during critical growth phases (800-10K validators)
- Incentivizes L1 validator set growth through economies of scale
- Reaches 20.8x burn peak at 10,000 validators vs. ACP-77 (transformative network scale)
- Gracefully declines to sustainable 1x baseline as network matures

## Specification

### 1: Base Commission Rate

**Change:** M = 512 → **1,024 nAVAX/sec** (2x increase)
- Current equivalent: 1.327 AVAX/month
- New equivalent: 2.654 AVAX/month

### 2: L1-Size-Dependent Fee Multiplier

This ACP proposes to introduce a new formula tied to the number of validators of a given L1.

**Formula:**
$$ \text{multiplier}(n) = 1.0 + 17.84 \times e^{-0.3 \times (n-1)} $$

**Fee Schedule:**

| L1 Validators | Multiplier | Fee/Validator | L1 Total |
|---|---|---|---|
| 50+ | 1.00x | 2.65 AVAX | Variable |
| 15 | 1.27x | 3.36 AVAX | 50.46 AVAX |
| **10** | **2.20x** | **5.84 AVAX** | **58.36 AVAX** |
| 5 | 6.37x | 16.91 AVAX | 84.57 AVAX |
| 1 | 18.84x | 50.00 AVAX | 50.00 AVAX |

**Key Feature:** Cost plateau at 10-15 validators makes decentralization economically irresistible.

### 3: Gaussian Network Factor

**Formula:**

$$ \text{networkFactor}(V) = 1.0 + 2.84 \times e^{-\left(\frac{V - 10,000}{7,500}\right)^2} $$

**Properties:**
- Peak at V = 10,000: factor = **3.84** (20.8x burn)
- Baseline (V = 0, ∞): factor = **1.0** (1x burn)
- Smooth, continuous, symmetric bell curve
- Width = 7,500 validators (controls spread)

**Final Fee Formula:**

$$ \text{feeRate}(V) = 2.65 \times \text{multiplier}(n) \times \text{networkFactor}(V) $$

**Network Factor Values:**

| Validators | Factor | Burn Multiplier |
|---|---|---|
| 800 | 1.63 | 8.8x |
| 1,000 | 1.68 | 9.1x |
| 2,000 | 1.90 | 10.3x |
| 5,000 | 2.82 | 15.3x |
| **10,000** | **3.84** | **20.8x (PEAK)** |
| 15,000 | 2.82 | 15.3x |
| 20,000 | 1.57 | 8.5x |

## Core Formulas

This section consolidates the mathematical formulas that define ACP-255's fee structure.

### L1 Multiplier Function

**Purpose:** Incentivize L1 decentralization by reducing per-validator costs as validator count increases.

**Formula:**
$$ \text{multiplier}(n) = 1.0 + 17.84 \times e^{-0.3 \times (n-1)} $$

**Parameters:**
- `n` = number of validators in the L1
- `17.84` = maximum premium coefficient (determines single-validator penalty)
- `0.3` = decay rate (controls how quickly costs decrease with additional validators)

**Behavior:**
- At `n=1`: multiplier = 18.84 (maximum penalty)
- At `n=5`: multiplier = 6.37 (significant reduction)
- At `n=10`: multiplier = 2.20 (approaching plateau)
- At `n≥50`: multiplier ≈ 1.0 (baseline)

### Network Factor Function (Gaussian)

**Purpose:** Create time-dependent fee dynamics that fund protocol development during critical growth phases.

**Formula:**
$$ \text{networkFactor}(V) = 1.0 + 2.84 \times e^{-\left(\frac{V - 10,000}{7,500}\right)^2} $$

**Parameters:**
- `V` = total number of L1 validators in the network
- `2.84` = peak amplitude (A)
- `10,000` = center of bell curve (μ)
- `7,500` = width parameter (σ)

**Behavior:**
- At `V=0`: factor ≈ 1.0 (baseline, far from peak)
- At `V=800`: factor = 1.63 (early growth phase)
- At `V=10,000`: factor = 3.84 (peak)
- At `V→∞`: factor → 1.0 (mature network baseline)

### Combined Fee Formula

**Final per-validator monthly fee:**
$$ \text{feeRate}(V, n) = M \times \text{multiplier}(n) \times \text{networkFactor}(V) $$

Where:
- `M` = base rate = 2.65 AVAX/month (1,024 nAVAX/sec × 2,592,000 sec/month / 10⁹)
- `n` = validators in the specific L1
- `V` = total network validators

**Example Calculation (8-validator L1 at 10K network validators):**
```
multiplier(8) = 1.0 + 17.84 × e^(-0.3 × 7) = 2.62
networkFactor(10,000) = 1.0 + 2.84 × e^0 = 3.84
feeRate = 2.65 × 2.62 × 3.84 = 26.66 AVAX/validator/month
Total L1 cost = 26.66 × 8 = 213.28 AVAX/month
```

## L1 Validator Growth Assumptions

This section explains the assumptions underlying ACP-255's economic model, addressing how L1 validator counts are expected to evolve over time.

### L1 Archetypes

Based on current Avalanche L1 deployments and ecosystem analysis, we identify two primary L1 categories:

#### Enterprise L1s (Estimated 60-70% of total L1s)

**Characteristics:**
- **Validator count:** 3-8 validators (typical: 5)
- **Growth trajectory:** Slow/stable (designed for control, not maximum decentralization)
- **Operator profile:** Internal infrastructure, trusted partners, professional node operators
- **Examples:** Financial institutions, gaming studios, supply chain platforms
- **Cost sensitivity:** Low (can absorb $100-500/month infrastructure costs as part of operational budget)
- **Security model:** Permissioned or semi-permissioned validator sets

**Typical lifecycle:**
- Launch: 1-3 validators (core team)
- 6 months: 3-5 validators (early partners)
- 12+ months: 5-8 validators (stable operating model)

#### Community L1s (Estimated 30-40% of total L1s)

**Characteristics:**
- **Validator count:** 10-50+ validators (typical plateau: 15-25)
- **Growth trajectory:** Rapid expansion driven by community/token incentives
- **Operator profile:** Decentralized community members, independent validators, delegators
- **Examples:** DeFi protocols, DAOs, open-source gaming ecosystems, NFT platforms
- **Cost sensitivity:** High (rely on volunteer validators or tokenomic incentives)
- **Security model:** Maximally decentralized, open validator sets

**Typical lifecycle:**
- Launch: 1-3 validators (core team bootstrap)
- 3 months: 5-10 validators (early community adoption)
- 6-12 months: 10-20 validators (active growth phase)
- 12+ months: 15-30 validators (mature, cost-optimal plateau)

### Expected Network Growth Trajectory

| Time Period | Total L1 Validators | Avg L1 Size | Rationale |
|-------------|---------------------|-------------|-----------|
| Current (2025) | ~800 | 5-8 | Early adoption, mostly enterprise pilots |
| 6 months | 1,500-2,000 | 6-10 | Initial community L1s launching |
| 12 months | 3,000-5,000 | 8-12 | Community L1s reaching maturity |
| 24 months | 8,000-10,000 | 10-15 | Peak growth phase, ACP-255 plateau incentives working |
| 36+ months | 15,000-20,000 | 12-20 | Sustainable mature network |

### Key Insight: Natural Convergence at 10-15 Validators

ACP-255's cost plateau at 10-15 validators **aligns with the natural maturity target for community L1s**. This creates three economic zones:

1. **Zone 1 (1-5 validators):** High cost pressure → strong incentive to add validators
2. **Zone 2 (5-15 validators):** Decreasing marginal cost → continued growth incentive
3. **Zone 3 (15+ validators):** Cost plateau → stable equilibrium, no penalty for additional decentralization

**Economic equilibrium:** The majority of L1s will naturally converge to 10-15 validators over 12-24 months, driven by cost optimization rather than mandates.

## Marginal Cost Comparison

This section addresses the critical question: **How does ACP-255 change the economics of adding validators to an L1?**

Under ACP-77, adding validators is purely a cost (flat +1.33 AVAX/month per validator). Under ACP-255, **adding validators can reduce total L1 costs** due to the L1 multiplier function.

### Marginal Cost Analysis (800 Network Validators)

**Definition:** Marginal cost = change in total L1 monthly cost when adding one validator.

| L1 Size Transition | ACP-77 Total Cost | ACP-255 Total Cost | ACP-77 Marginal | ACP-255 Marginal | Change |
|-------------------|-------------------|--------------------|-----------------|--------------------|---------|
| 1 → 2 | 1.33 → 2.66 | 81.50 → 101.83 | **+1.33** | **+20.33** | 15.3x more expensive |
| 2 → 3 | 2.66 → 3.99 | 101.83 → 117.05 | **+1.33** | **+15.22** | 11.4x more expensive |
| 3 → 4 | 3.99 → 5.32 | 117.05 → 129.88 | **+1.33** | **+12.83** | 9.6x more expensive |
| 4 → 5 | 5.32 → 6.65 | 129.88 → 138.05 | **+1.33** | **+8.17** | 6.1x more expensive |
| 5 → 6 | 6.65 → 7.98 | 138.05 → 143.81 | **+1.33** | **+5.76** | 4.3x more expensive |
| 6 → 7 | 7.98 → 9.31 | 143.81 → 148.13 | **+1.33** | **+4.32** | 3.2x more expensive |
| 7 → 8 | 9.31 → 10.64 | 148.13 → 151.57 | **+1.33** | **+3.44** | 2.6x more expensive |
| 8 → 9 | 10.64 → 11.97 | 151.57 → 154.38 | **+1.33** | **+2.81** | 2.1x more expensive |
| 9 → 10 | 11.97 → 13.30 | 154.38 → 156.74 | **+1.33** | **+2.36** | 1.8x more expensive |
| **10 → 11** | **13.30 → 14.63** | **156.74 → 158.72** | **+1.33** | **+1.98** | **1.5x more expensive** |
| **11 → 12** | **14.63 → 15.96** | **158.72 → 160.42** | **+1.33** | **+1.70** | **1.3x more expensive** |
| **12 → 15** | **15.96 → 19.95** | **160.42 → 164.64** | **+1.33/ea** | **+1.41/ea** | **~1.1x more expensive** |
| **15 → 20** | **19.95 → 26.60** | **164.64 → 173.60** | **+1.33/ea** | **+1.79/ea** | **1.3x more expensive** |
| **20 → 50** | **26.60 → 66.50** | **173.60 → 216.50** | **+1.33/ea** | **+1.43/ea** | **~1.1x more expensive** |

### Key Findings

1. **1-9 validators:** ACP-255 marginal costs are significantly higher (2-15x), creating strong economic pressure to grow beyond single-digit validator sets.

2. **10-15 validators (PLATEAU ZONE):**
   - Marginal costs approach ACP-77 levels (1.3-1.8x)
   - This is the **cost-optimal equilibrium point**
   - Adding validators from 10→15 costs only ~$1.50-2.00/validator (vs. $1.33 flat under ACP-77)

3. **15+ validators:**
   - Marginal costs stabilize around 1.1-1.3x ACP-77
   - No penalty for additional decentralization beyond the plateau

### Economic Insight: "Gravity Well" Effect

ACP-255 creates a **cost gravity well** at 10-15 validators:

- **Below 10:** High marginal costs pull L1s upward toward the plateau
- **At 10-15:** Minimal marginal cost creates stable equilibrium
- **Above 15:** Continued decentralization is economically viable (marginal cost only slightly higher than ACP-77)

**Example: 5-validator L1 → 15-validator L1**

| Metric | ACP-77 | ACP-255 | Change |
|--------|--------|---------|--------|
| **Total monthly cost (5 validators)** | 6.65 AVAX | 138.05 AVAX | 20.8x |
| **Total monthly cost (15 validators)** | 19.95 AVAX | 164.64 AVAX | 8.3x |
| **Cost increase for 10 additional validators** | +13.30 AVAX | +26.59 AVAX | +$532/month |
| **Per-validator average cost (5 val)** | 1.33 AVAX | 27.61 AVAX | 20.8x |
| **Per-validator average cost (15 val)** | 1.33 AVAX | 10.98 AVAX | 8.3x |

**Conclusion:** Adding 10 validators (5→15) costs an additional ~$530/month, but **reduces per-validator cost by 60%** (27.61 → 10.98 AVAX). This makes decentralization economically attractive for L1s with sufficient budget.

### Marginal Cost at Different Network Scales

As the network grows, the Gaussian factor amplifies all costs. Here's how marginal costs evolve:

**10 → 11 validator transition:**

| Network Size | ACP-77 Marginal | ACP-255 Marginal | Ratio |
|--------------|-----------------|------------------|-------|
| 800 validators | +1.33 AVAX | +1.98 AVAX | 1.5x |
| 5,000 validators | +1.33 AVAX | +2.88 AVAX | 2.2x |
| 10,000 validators (PEAK) | +1.33 AVAX | +3.84 AVAX | 2.9x |
| 20,000 validators | +1.33 AVAX | +2.05 AVAX | 1.5x |

**Key insight:** Even at the Gaussian peak (10K validators), the marginal cost of the 11th validator is only 2.9x ACP-77, making the 10-15 validator plateau economically viable across all network growth phases.

## Game-Theoretic Equilibrium

This section analyzes how ACP-255 changes the optimal validator count for L1s from a game-theoretic perspective.

### ACP-77 Equilibrium (Current State)

**Optimal validator count:** **1** (minimize costs)

**Economic forces:**
- **Cost minimization:** Every additional validator adds flat +1.33 AVAX/month
- **Security/decentralization incentive:** Non-economic (reputation, regulatory, community pressure)
- **Result:** L1s only add validators for non-economic reasons (e.g., avoiding "single point of failure" optics)

**Observed outcome:** Most L1s cluster around 5-8 validators—above the cost-optimal point (1), but below true decentralization (15+).

**Nash equilibrium:** Single-validator L1s are rational profit-maximizers under ACP-77.

---

### ACP-255 Equilibrium (Proposed State)

**Optimal validator count:** **10-15** (minimize per-validator cost)

**Economic forces:**

1. **Cost minimization (1-10 validators):**
   - High marginal costs create pressure to add validators
   - Each additional validator reduces per-validator average cost
   - Strong economic incentive to grow toward plateau

2. **Cost plateau (10-15 validators):**
   - Marginal cost approaches baseline ACP-77 levels
   - **Economic equilibrium:** No strong incentive to add more, but no penalty either
   - L1s naturally stabilize in this range

3. **Beyond plateau (15+ validators):**
   - Marginal costs remain low (~1.1-1.3x ACP-77)
   - Community/decentralization goals can be pursued without prohibitive cost
   - Secondary incentives (token distribution, governance) become primary drivers

**Nash equilibrium:** 10-15 validator L1s represent the cost-optimal rational strategy.

---

### Expected Behavioral Changes

| L1 Type | Current (ACP-77) | Expected (ACP-255) | Mechanism |
|---------|------------------|-------------------|-----------|
| **Single-validator L1** | Rational (cost-optimal) | Economically unviable ($1,000/month @ $20 AVAX) | Forced decentralization or shutdown |
| **3-5 validator L1** | Common (security vs. cost trade-off) | Strong pressure to reach 10+ | Marginal cost of adding 5-7 validators is high but justified by 60% per-validator cost reduction |
| **10-15 validator L1** | Rare (expensive) | New equilibrium (cost-optimal) | Natural convergence point |
| **20+ validator L1** | Very rare (altruistic) | More common (economically viable) | Plateau effect removes cost penalty |

---

### Time-Dependent Equilibrium Shift

The Gaussian network factor creates **time-dependent equilibrium dynamics**:

**At 800 validators (current):**
- Optimal L1 size: 10-15 validators
- Cost pressure: 8.5x ACP-77
- Expected behavior: Slow migration from 5→10 validators

**At 5,000 validators (12-18 months):**
- Optimal L1 size: 10-15 validators (unchanged)
- Cost pressure: 14.7x ACP-77
- Expected behavior: Rapid consolidation toward 10-15 validator equilibrium

**At 10,000 validators (24-36 months, PEAK):**
- Optimal L1 size: 10-15 validators (unchanged)
- Cost pressure: 20.8x ACP-77 (maximum)
- Expected behavior: L1s below 10 validators face existential cost pressure; plateau becomes universal

**At 20,000 validators (mature network):**
- Optimal L1 size: 10-15 validators (still optimal, but less critical)
- Cost pressure: 5.2x ACP-77 (declining toward baseline)
- Expected behavior: Stable equilibrium; decentralization driven by non-economic factors

---

### Caveat: Enterprise L1s May Accept Higher Costs

**Not all L1s will conform to the 10-15 validator equilibrium.**

Enterprise L1s with strict control requirements may rationally choose to remain at 3-5 validators and absorb higher costs (e.g., $500-1,000/month vs. $200-300/month at plateau).

**Why this is acceptable:**
- ACP-255 **taxes centralization** without making it impossible
- Higher fees fund protocol development (offsetting network externality of centralized L1s)
- Enterprise budgets can absorb premium for control (analogous to AWS dedicated hosting vs. multi-tenant)

**Expected distribution at equilibrium:**
- 30-40% of L1s: 3-8 validators (enterprise, willing to pay premium)
- 50-60% of L1s: 10-15 validators (cost-optimal plateau)
- 10% of L1s: 20+ validators (maximally decentralized, community-driven)

---

### Summary: Equilibrium Shift

| Metric | ACP-77 | ACP-255 |
|--------|--------|---------|
| **Optimal L1 size** | 1 validator | 10-15 validators |
| **Observed average** | 5-8 validators | 10-15 validators (projected) |
| **Single-validator viability** | Economically rational | Economically unviable ($1K/month) |
| **Decentralization incentive** | Non-economic (optics) | Economic (cost reduction) |
| **Equilibrium stability** | Weak (no cost pressure) | Strong (cost plateau creates attractor) |

**Conclusion:** ACP-255 shifts the Nash equilibrium from centralized (1 validator) to meaningfully decentralized (10-15 validators) through pure economic incentives, without mandates or artificial caps.

## Interaction with ACP-247 (L1 Validator Rewards)

ACP-255 is designed to work in tandem with [ACP-247](https://github.com/avalanche-foundation/ACPs/pull/247), which proposes AVAX rewards for L1 validators based on their participation in Primary Network validation.

### Combined Economic Model

**ACP-255 (fees) + ACP-247 (rewards) = Net validator economics**

#### Without ACP-247 (Fee-Only Model)

| Role | Monthly Cost | Monthly Revenue | Net |
|------|--------------|-----------------|-----|
| L1 validator (10-val L1, 800 network) | 15.67 AVAX | 0 AVAX | **-15.67 AVAX** |
| L1 operator (10-validator L1) | 156.74 AVAX | 0 AVAX | **-156.74 AVAX** |

**Result:** Pure cost model discourages validator participation, especially for community-driven L1s.

---

#### With ACP-247 (Fee + Reward Model)

Assuming ACP-247 distributes rewards proportional to L1 validator uptime and P-Chain stake:

| Role | Monthly Cost | Monthly Reward (ACP-247) | Net |
|------|--------------|--------------------------|-----|
| L1 validator (10-val L1, 800 network) | 15.67 AVAX | ~10-20 AVAX (estimated) | **-5 to +4 AVAX** |
| L1 operator (10-validator L1) | 156.74 AVAX | 100-200 AVAX (if all validators earn rewards) | **-50 to +43 AVAX** |

**Result:** Validator participation becomes revenue-neutral or revenue-positive, dramatically improving L1 decentralization economics.

---

### Economic Balance: ACP-255 Push + ACP-247 Pull

**ACP-255 (the "stick"):**
- High fees for centralized L1s (1-5 validators)
- Cost plateau incentive to reach 10-15 validators
- Network-wide burn increases (funding protocol development)

**ACP-247 (the "carrot"):**
- Rewards for L1 validators who secure the Primary Network
- Offsets ACP-255 fee increases
- Creates positive-sum game: Validate L1 + earn P-Chain rewards

**Combined effect:**
1. **L1s grow validator sets** (ACP-255 cost pressure + ACP-247 revenue opportunity)
2. **Validators join L1s** (ACP-247 rewards offset ACP-255 fees)
3. **Primary Network security increases** (more L1 validators = more P-Chain validators via ACP-247 incentives)
4. **AVAX burn increases** (ACP-255) while **validator revenue increases** (ACP-247)

---

### Timing and Implementation Strategy

**Recommendation:** Implement ACP-255 and ACP-247 **together or in rapid succession** to avoid fee shock.

#### Scenario 1: ACP-255 Alone (Suboptimal)
- Month 0: ACP-255 activates
- Fees increase 8.5-20x
- No reward offset
- **Result:** Community backlash, L1 validator churn, calls for fee rollback

#### Scenario 2: ACP-247 Alone (Incomplete)
- Month 0: ACP-247 activates
- L1 validators earn rewards
- Fees remain flat (ACP-77)
- **Result:** Insufficient burn, no decentralization pressure, rewards feel like "free money"

#### Scenario 3: Coordinated Launch (Optimal)
- Month 0: ACP-255 + ACP-247 activate simultaneously
- Fees increase 8.5x (ACP-255)
- Validators earn rewards (ACP-247)
- **Net effect:** Revenue-neutral to revenue-positive for active validators
- **Result:** Sustainable burn + decentralization incentives + positive community sentiment

---

### Example: 10-Validator Community L1

**Without ACP-247:**
- Total L1 cost: 156.74 AVAX/month (at 800 network validators)
- Revenue: 0
- **Net: -156.74 AVAX/month (-$3,135/month @ $20 AVAX)**

**With ACP-247 (assuming 15 AVAX/validator/month reward):**
- Total L1 cost: 156.74 AVAX/month
- Revenue: 150 AVAX/month (10 validators × 15 AVAX)
- **Net: -6.74 AVAX/month (-$135/month @ $20 AVAX)**

**Result:** A 95% cost reduction for L1s whose validators actively participate in Primary Network validation.

---

### Open Question: ACP-247 Reward Magnitude

The effectiveness of the ACP-255 + ACP-247 pairing depends critically on **how much ACP-247 rewards per L1 validator**.

**If ACP-247 rewards are too low (e.g., 5 AVAX/month):**
- Net L1 cost remains high (156.74 - 50 = 106.74 AVAX/month for 10-val L1)
- ACP-255 fee pressure dominates
- Risk: Validator churn, L1 shutdowns

**If ACP-247 rewards are too high (e.g., 30 AVAX/month):**
- Net L1 cost becomes negative (156.74 - 300 = -143.26 AVAX/month for 10-val L1)
- L1 validation becomes a profit center (unsustainable token distribution)
- Risk: Sybil attacks, low-quality L1s farming rewards

**Optimal range (estimated): 10-20 AVAX/validator/month**
- Offsets 60-95% of ACP-255 costs
- Keeps L1 validation slightly net-negative or neutral (ensuring quality)
- Sustainable long-term (rewards decline as network matures)

**Recommendation:** ACP-247 reward parameters should be calibrated to **offset 70-80% of ACP-255 costs** at the 10-15 validator plateau.

---

### Governance Coordination

Given the tight coupling between ACP-255 and ACP-247, the following governance process is recommended:

1. **Joint proposal review:** Both ACPs should be evaluated together by the Avalanche Foundation and community
2. **Synchronized activation:** Implement both ACPs in the same network upgrade (or within 1-2 months)
3. **Parameter tuning:** Allow 3-6 month observation period, then adjust:
   - ACP-255 Gaussian parameters (if burn is too high/low)
   - ACP-247 reward rates (if validator economics are net-negative)
4. **Future ACPs:** Create mechanism for joint parameter updates (e.g., "ACP-XXX: Adjust 255/247 Balance")

---

### Summary: Why ACP-255 + ACP-247 Work Together

| Without ACP-247 | With ACP-247 |
|-----------------|--------------|
| High L1 costs (8.5-20x) | Costs offset by rewards |
| Validator churn risk | Validator retention via revenue |
| Pure fee pressure | Balanced incentive (cost + reward) |
| Community resistance likely | Community support likely (net-neutral economics) |
| AVAX burn increase only | AVAX burn + validator growth + P-Chain security |

**Conclusion:** ACP-255 alone creates deflationary pressure but risks validator attrition. ACP-247 alone rewards participation but lacks burn mechanism. **Together, they create a balanced economic system** that funds protocol development while incentivizing decentralization.

## AVAX Burn Scenarios

_AVAX price is fixed at 20$ unless otherwise specified._

### Current Network (800 Validators, 100 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 1,064 AVAX | 9,040 AVAX | 8.5x |
| Annual burn | 12,768 AVAX | 108,480 AVAX | 8.5x |
| Annual value | $255K | $2.17M | +$1.91M |

### Growth to 2,000 Validators (250 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 2,660 AVAX | 26,360 AVAX | 9.9x |
| Annual burn | 31,920 AVAX | 316,320 AVAX | 9.9x |
| Annual value | $639K | $6.33M | +$5.69M |

### Growth to 5,000 Validators (625 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 6,650 AVAX | 97,800 AVAX | 14.7x |
| Annual burn | 79,800 AVAX | 1,173,600 AVAX | 14.7x |
| Annual value | $1.60M | $23.47M | +$21.87M |

### Growth to 10,000 Validators (PEAK) (1,250 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 13,300 AVAX | 266,200 AVAX | **20.8x** |
| Annual burn | 159,600 AVAX | 3,194,400 AVAX | **20.8x** |
| Annual value | $3.19M | $63.89M | **+$60.70M** |

### Scale to 20,000 Validators (2,500 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 26,600 AVAX | 218,000 AVAX | 8.2x |
| Annual burn | 319,200 AVAX | 2,616,000 AVAX | 8.2x |
| Annual value | $6.38M | $52.32M | +$45.94M |

## Fee Evolution Example (8-Validator L1)

| Validators | Network Factor | Fee/Validator | Per-L1 Cost | Burn vs ACP-77 |
|---|---|---|---|---|
| 800 | 1.63 | 11.3 AVAX | 90.4 AVAX | 8.8x |
| 2,000 | 1.90 | 13.2 AVAX | 105.6 AVAX | 10.3x |
| 5,000 | 2.82 | 19.6 AVAX | 156.8 AVAX | 15.3x |
| **10,000** | **3.84** | **26.6 AVAX** | **212.8 AVAX** | **20.8x** |
| 20,000 | 1.57 | 10.9 AVAX | 87.2 AVAX | 8.5x |

## Technical Implementation

### Fee Calculation Algorithm

```python
import math

def calculate_network_factor(V):
    """Gaussian network factor centered at 10,000 validators"""
    A = 2.84          # Peak amplitude
    W = 7500          # Width parameter
    center = 10000    # Peak location
    
    exponent = -((V - center) / W) ** 2
    return 1.0 + A * math.exp(exponent)

def calculate_validator_fee(total_validators, l1_validator_count):
    """Calculate per-validator monthly fee in AVAX"""
    
    # L1 multiplier
    n = l1_validator_count
    if n <= 0:
        n = 1
    l1_mult = 1.0 + 17.84 * math.exp(-0.3 * (n - 1))
    l1_mult = max(1.0, min(18.84, l1_mult))
    
    # Network factor
    net_factor = calculate_network_factor(total_validators)
    
    # Base rate
    M = 1024  # nAVAX/sec
    
    # Seconds per month
    seconds_per_month = 30 * 24 * 60 * 60  # 2,592,000
    
    # Calculate fee
    fee_nAVAX = M * l1_mult * net_factor * seconds_per_month
    fee_AVAX = fee_nAVAX / 1e9
    
    return fee_AVAX
```

### Smart Contract Constants

```solidity
uint256 constant M_BASE_RATE = 1024;              // nAVAX/sec
uint256 constant MAX_PREMIUM = 17.84;             // L1 multiplier coefficient
float constant DECAY_RATE = 0.3;                  // L1 multiplier decay
float constant GAUSSIAN_AMPLITUDE = 2.84;         // Peak amplitude
float constant GAUSSIAN_WIDTH = 7500;             // Bell curve width
uint32 constant GAUSSIAN_CENTER = 10000;          // Peak location
```

## Governance Parameters (Adjustable via future ACPs)

| Parameter | Current | Range | Adjustment Impact |
|---|---|---|---|
| M | 1,024 | 512-2,048 | Scales all fees proportionally |
| GAUSSIAN_AMPLITUDE | 2.84 | 1.5-4.0 | Controls peak height |
| GAUSSIAN_WIDTH | 7,500 | 4,000-12,000 | Controls curve width |
| GAUSSIAN_CENTER | 10,000 | 5,000-15,000 | Shifts peak location |

## Security Considerations

**Validator Count Verification:** `L1ValidatorCount` tracked on-chain and verified by consensus rules. Misreporting results in automatic deactivation.

**Rapid Growth:** Gaussian curve prevents hyperinflation of fees. Governance can adjust parameters if adoption exceeds projections.

**Small L1 Pressure:** Community grants and foundation support available for projects struggling with increased costs during transition.

## Economic Incentives

**Decentralization:** Cost plateau (10-15 validators) makes validator set growth economically irresistible. Single-validator L1s become uneconomical at $1,000/month.

**Early Adoption:** Bootstrap fees (8.5x) reward early validators and fund critical protocol development.

**Network Effects:** Each new validator reduces per-validator cost (due to L1 multiplier decay), creating virtuous adoption spiral.

## Open Questions

- Should the Base Commission Rate be doubled? Or maybe another multiplier would be better?
- Should the Fee Multiplier start at 50 AVAX, or more, or less? How?

## Conclusion

ACP-255 transforms Avalanche's validator economics through an elegant Gaussian fee curve that:

1. **Funds Growth:** 8.5-20.8x burn multiplier during critical 800-10K validator phase
2. **Incentivizes Decentralization:** Cost plateau at 10-15 validators eliminates centralization
3. **Sustainable Scale:** Gracefully declines to 1x baseline, providing 5.2x sustainable burn indefinitely
4. **Mathematically Elegant:** Continuous, smooth, symmetric bell curve with no discontinuities

The result: Protocol sustainability, validator decentralization, and self-reinforcing network growth.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
