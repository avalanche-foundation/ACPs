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
- At 50,000 validators: **5.2x higher burn** (66,500 → 347,000 AVAX/month)

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

This ACP proposes to introduce a new formula to account tied to the number of validators of a given L1.

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

$$ \text{network_factor}(V) = 1.0 + 2.84 \times e^{-\left(\frac{V - 10,000}{7,500}\right)^2} $$

**Properties:**
- Peak at V = 10,000: factor = **3.84** (20.8x burn)
- Baseline (V = 0, ∞): factor = **1.0** (1x burn)
- Smooth, continuous, symmetric bell curve
- Width = 7,500 validators (controls spread)

**Final Fee Formula:**

$$ \text{fee_rate}(V) = 2.65 \times \text{multiplier}(n) \times \text{network_factor}(V) $$

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
| 50,000 | 1.00 | 5.4x |

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

### Scale to 50,000 Validators (6,250 L1s)

| Metric | ACP-77 | ACP-255 | Change |
|---|---|---|---|
| Monthly burn | 66,500 AVAX | 347,000 AVAX | 5.2x |
| Annual burn | 798,000 AVAX | 4,164,000 AVAX | 5.2x |
| Annual value | $15.96M | $83.28M | +$67.32M |

## Fee Evolution Example (8-Validator L1)

| Validators | Network Factor | Fee/Validator | Per-L1 Cost | Burn vs ACP-77 |
|---|---|---|---|---|
| 800 | 1.63 | 11.3 AVAX | 90.4 AVAX | 8.8x |
| 2,000 | 1.90 | 13.2 AVAX | 105.6 AVAX | 10.3x |
| 5,000 | 2.82 | 19.6 AVAX | 156.8 AVAX | 15.3x |
| **10,000** | **3.84** | **26.6 AVAX** | **212.8 AVAX** | **20.8x** |
| 20,000 | 1.57 | 10.9 AVAX | 87.2 AVAX | 8.5x |
| 50,000 | 1.00 | 6.94 AVAX | 55.52 AVAX | 5.4x |

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

**Validator Count Verification:** L1ValidatorCount tracked on-chain and verified by consensus rules. Misreporting results in automatic deactivation.

**Rapid Growth:** Logarithmic curve prevents hyperinflation of fees. Governance can adjust parameters if adoption exceeds projections.

**Small L1 Pressure:** Community grants and foundation support available for projects struggling with increased costs during transition.

## Economic Incentives

**Decentralization:** Cost plateau (10-15 validators) makes validator set growth economically irresistible. Single-validator L1s become uneconomical at $1,000/month.

**Early Adoption:** Bootstrap fees (8.5x) reward early validators and fund critical protocol development.

**Network Effects:** Each new validator reduces per-validator cost (due to Gaussian decay after peak), creating virtuous adoption spiral.

## Open Questions

- Should the Base Commission Rate be doubled? Or maybe another multipler would be better?
- Should the Fee Multiplier start a 50 AVAX, or more, or less? How?

## Conclusion

ACP-255 transforms Avalanche's validator economics through an elegant Gaussian fee curve that:

1. **Funds Growth:** 8.5-20.8x burn multiplier during critical 800-10K validator phase
2. **Incentivizes Decentralization:** Cost plateau at 10-15 validators eliminates centralization
3. **Sustainable Scale:** Gracefully declines to 1x baseline, providing 5.2x sustainable burn indefinitely
4. **Mathematically Elegant:** Continuous, smooth, symmetric bell curve with no discontinuities

The result: Protocol sustainability, validator decentralization, and self-reinforcing network growth.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
