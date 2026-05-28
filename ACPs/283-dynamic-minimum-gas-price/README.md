| ACP | 283 |
| :- | :- |
| **Title** | Dynamic Minimum Gas Price |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Martin Eckardt ([@martineckardt](https://github.com/martineckardt)) |
| **Status** | Proposed [Discussion](https://github.com/avalanche-foundation/ACPs/discussions/284) |
| **Track** | Standards |

## Abstract

Proposes making the minimum gas price parameter on the C-Chain dynamically adjustable via validator voting, using the same mechanism established by [ACP-176](../176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) (gas target) and [ACP-226](../226-dynamic-minimum-block-times/README.md) (minimum block delay).

Currently, ACP-176 sets the minimum gas price to 1 wei, allowing spam transactions to flood the chain at near-zero cost during low-activity periods. This has been exploited by protocols such as XEN, which performs proof-of-work mining on-chain when gas prices are negligible, causing unnecessary state growth and validator resource consumption. The current defense relies on wallets artificially inflating gas prices via failing transactions — a stopgap measure costing the community several AVAX per day.

This ACP applies the proven validator voting pattern from ACP-176 and ACP-226 to the minimum gas price. Validators express their preferred minimum gas price via node configuration, and the network converges on the median preference weighted by stake. The dynamic minimum gas price acts as a true floor: during low-activity periods the floor binds, while during congestion the existing ACP-176 fee mechanism dominates unchanged. This requires no future upgrades for parameter adjustments and allows the network to organically respond to changing spam conditions.

## Motivation

[ACP-176](../176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) introduced a dynamic fee mechanism for the C-Chain, setting the minimum gas price $M$ to 1 wei — the smallest possible denomination of the native EVM asset. The rationale was that the dynamic fee mechanism would raise prices under load, making a near-zero floor acceptable for price discovery.

In practice, the lowest observed C-Chain gas prices have been on the order of $10^5$ wei (~0.0001 nAVAX), cheap enough that spam protocols like XEN remain profitable, with their own demand keeping ACP-176's price discovery from pushing prices even lower.

The validator voting mechanism for adjusting network parameters has already been proven twice on the Avalanche network:

1. **ACP-176**: Validators dynamically adjust the target gas consumption rate $T$
2. **ACP-226**: Validators dynamically adjust the minimum block delay time

This ACP applies the same proven pattern to the minimum gas price. The benefits are:

- **No future upgrades required**: Future adjustments to the minimum gas price do not require coordinated network upgrades
- **Stake-weighted convergence**: The effective value converges on the median preference weighted by validator stake
- **Decentralized governance**: Each validator independently sets their preference based on local assessment of network conditions

Pre-ACP-176, the C-Chain gas price was approximately 25 nAVAX. Even a minimum gas price floor of 0.01 nAVAX — 2,500x below this baseline — would be entirely invisible to normal users while making spam economically infeasible.

## Specification

### Block Header Changes

A single new field is added to block headers: `minimumGasPriceExponent`, represented as a `uint64`.

The value of `minimumGasPriceExponent` MAY be updated in each block, similar to the gas target exponent introduced in ACP-176. The mechanism is specified below.

### Dynamic minimum gas price mechanism

The effective minimum gas price $M$, in wei, is defined as:

$$M = e^{\dfrac{q}{D}}$$

Where:
- $q$ is a non-negative integer carried in the `minimumGasPriceExponent` header field, initialized to $q_{initial}$ at activation.
- $D$ is a fixed conversion constant.

Both $q_{initial}$ and $D$ are specified in [Activation Parameters](#activation-parameters).

After the execution of transactions in block $b$, the value of $q$ can be increased or decreased by up to $Q$. It MUST be the case that $\left|\Delta q\right| \leq Q$, or block $b$ is considered invalid. The amount by which $q$ changes after executing block $b$ is specified by the block builder.

Block builders (i.e., validators) MAY set their desired value for $M$ in their configuration via the `min-price-target` setting (specified in wei). Their desired value for $q$ is then calculated locally as:

$$q_{desired} = \left\lceil D \cdot \ln(M_{desired}) \right\rceil$$

Since $q_{desired}$ is only used locally, it is safe for implementations to approximate the value of $\ln(M_{desired})$ and round the resulting value to the nearest integer. Alternatively, client implementations MAY choose to use binary search to find the closest integer solution.

If a validator does not set `min-price-target`, the validator SHOULD default to the previous block's value of $q$. This matches the abstention default of ACP-176 and ACP-226.

When building a block, builders calculate their next preferred value for $q$ based on the network's current value (`q_current`) according to the same procedure used in ACP-176 and ACP-226:

```python
# Calculates a node's new desired value for q for a given block
def calc_next_q(q_current: int, q_desired: int, max_change: int) -> int:
    if q_desired > q_current:
        return q_current + min(q_desired - q_current, max_change)
    else:
        return q_current - min(q_current - q_desired, max_change)
```

As $q$ is updated after the execution of transactions within the block, $M$ is also updated such that $M = e^{q/D}$ at all times. The change to $q$ in block $b$ takes effect for the floor used when validating and building block $b+1$; it does not affect the gas price of transactions included in block $b$ itself.

Allowing block builders to adjust the minimum gas price in blocks that they produce makes it such that the effective value should converge over time to the point where 50% of the voting stake weight wants it increased and 50% of the voting stake weight wants it decreased, because the number of blocks each validator produces is proportional to their stake weight.

### Compatibility with ACP-176

ACP-176 defines the per-block gas price as $P = M \cdot e^{x/K}$, where $M$ is a static minimum, $x$ is the gas price excess, and $K$ is the gas price update factor. Making $M$ dynamic without changing the structure of this formula would require rebalancing $x$ on every floor change to keep $P$ continuous. This is workable when $M$ changes rarely but breaks down once $M$ may oscillate every block.

To support oscillating $M$ values, the price formula is changed to:

$$P = \max\left(M, e^{\dfrac{x}{K}}\right)$$

The $\max$ enforces the floor directly: $P$ is never below $M$, even when integer rounding of $e^{x/K}$ produces a value just under $M$ (see the bias-low case below). When $M = 1$ wei (the value at activation), $e^{x/K} \geq 1$ always, so the $\max$ is a no-op and the formula reduces to ACP-176's $P = M \cdot e^{x/K}$.

The state variable $x$ is additionally subject to a ratchet constraint:

$$x \geq x_{floor}$$

where $x_{floor}$ is derived from $M$ via binary search over $\text{price}(\cdot)$, the integer-rounded forward price function used elsewhere in block validation.

Let $x^* = \min\{x : \text{price}(x) \geq M\}$:

- If $\text{price}(x^*) = M$, $x_{floor} = x^*$.
- If $\text{price}(x^*) > M$ (the integer-rounded price function cannot represent $M$ exactly), $x_{floor} = x^* - 1$, the largest integer where $\text{price}(x_{floor}) < M$.

The second case biases toward the lower price when $M$ is not exactly representable by $\text{price}(\cdot)$, keeping the floor enforcement consistent with prices the implementation actually produces. Implementations MUST use this binary-search procedure rather than evaluating the closed-form $\left\lceil q \cdot K / D \right\rceil$, which may diverge from $\text{price}(\cdot)$.

After updating $q$ for the new block, the block-execution rule is:

1. If the current $x < x_{floor}$, set $x \leftarrow x_{floor}$.
2. Otherwise, leave $x$ unchanged.

In particular, $x$ is never reduced as a consequence of a floor change. A decrease in $q$ relaxes the constraint but does not modify $x$.

### Activation Parameters

The only value this ACP specifies is `BlocksToDouble`; the remaining values are either derived from it or set implicitly.

**Parameters**

<div align="center">

| Parameter | Description | Value |
| - | - | - |
| `BlocksToDouble` | max-change blocks required to halve or double $M$ | $3{,}600$ |

</div>

`BlocksToDouble` expresses the number of consecutive maximum-change blocks required to halve or double $M$ under sustained voting pressure, from which $Q$ is mechanically derived. A value of $3{,}600$ corresponds to roughly one hour at one-second block times, the current C-Chain block rate.

**Constants**

<div align="center">

| Constant | Description | Value |
| - | - | - |
| $D$ | exponential-update conversion constant | $415{,}828{,}534{,}307{,}635{,}077$ |
| $Q$ | per-block max $\left\|\Delta q\right\|$ | $80{,}063{,}993{,}375{,}475$ |
| $q_{initial}$ | initial value of `minimumGasPriceExponent` | $0$ |

</div>

$D$ is fully determined by the type constraint. The allowed range of $M$ is $[1, 2^{64})$ wei (the full `uint64` range), realized when $q$ ranges over $[0, 2^{64})$.

Setting $M_{max} = q_{max} = 2^{64}-1$ and solving for $D$:

$$
\begin{align}
M_{max} &= e^{q_{max}/D} \\
\ln(M_{max}) &= \dfrac{q_{max}}{D} \\
D &= \left\lfloor \dfrac{q_{max}}{\ln(M_{max})} \right\rfloor \\
&= \left\lfloor \dfrac{2^{64}-1}{\ln(2^{64}-1)} \right\rfloor
\end{align}
$$

$Q$ is derived from $D$ and `BlocksToDouble`:

$$Q = \left\lfloor \dfrac{D \cdot \ln(2)}{\text{BlocksToDouble}} \right\rfloor$$

$q_{initial} = 0$ means the effective minimum gas price at activation is $M = e^0 = 1$ wei, identical to the pre-activation value. Activation introduces the mechanism but does not change any user-visible parameter; subsequent movement of the floor requires explicit validator votes.

### Choosing `min-price-target`

This new mechanism allows for validators to specify their desired minimum gas price ($M_{desired}$) in their configuration via the `min-price-target` setting (specified in wei), and the value that they set impacts the effective minimum gas price of the network over time. When choosing what value makes sense for them, validators should consider:

- The cost of spam during low-activity periods (the floor primarily binds when there is limited organic demand to drive the ACP-176 base fee).
- The risk of pricing out legitimate users.
- The trajectory of organic gas prices.

While Avalanche Network Clients MAY suggest reference values, each validator chooses `min-price-target` independently. The default behavior SHOULD be to abstain.

## Backwards Compatibility

The changes proposed in this ACP require a network upgrade in order to take effect. Prior to its activation, the current minimum gas price of 1 wei continues to apply. Its activation should have minimal compatibility effects:

- **Transaction formats**: Unchanged. Wallets and transaction signing are not impacted.
- **User fees**: Activation does not change the effective minimum gas price — the initial value remains 1 wei, identical to the pre-activation behavior. The mechanism enables future increases, which validators must explicitly vote for.
- **Tooling**: Any tooling parsing the RLP block bytes will need to update.
- **Gas price estimation APIs**: `eth_gasPrice` and related APIs will need to respect the new dynamic minimum gas price floor.

## Reference Implementation

This section will be updated with a tagged release once a complete reference implementation has been merged.

## Security Considerations

This ACP changes the minimum gas price from a static constant to a dynamically-adjusted parameter governed by validator voting. Several security aspects should be considered:

**Validators setting the minimum gas price too high**: The exponential mechanism bounds how quickly the minimum gas price can change per block. With `BlocksToDouble = 3,600` (approximately one hour at one-second block times), the floor cannot halve or double in less than that many consecutive maximum-change blocks, giving the community time to detect and respond to concerning changes. There is no policy ceiling on the floor; the implicit ceiling at $M = 2^{64}-1$ wei is an artifact of the `uint64` representation of $q$, not a design choice. Validators are also economically incentivized not to price out users, as doing so reduces network utility and the value of their staked AVAX.

**Validators setting the minimum gas price too low**: The same bounded adjustment speed applies in the downward direction, providing significant time for detection and response.

**Comparison to existing approaches**: The minimum gas price adjustment mechanism has the same structure as the proven gas target adjustment (ACP-176) and minimum block delay adjustment (ACP-226), providing confidence in its security properties. Compared to Base's approach of administrator-set static floors, the validator voting mechanism achieves the same spam deterrence in a decentralized manner.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
