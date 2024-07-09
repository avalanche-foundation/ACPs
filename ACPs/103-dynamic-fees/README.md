```text
ACP: 103
Title: Add Dynamic Fees to the X-Chain and P-Chain
Author(s): Dhruba Basu (https://github.com/dhrubabasu), Alberto Benegiamo (https://github.com/abi87), Stephen Buttolph (https://github.com/StephenButtolph)
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/104
Status: Proposed
Track: Standards
```

## Abstract

Introduce a dynamic fee mechanism to the X-Chain and the P-Chain. Preview a future transition to a multidimensional fee mechanism.

## Motivation

Blockchains are resource-constrained environments. Users are charged for the execution and inclusion of their transactions based on the blockchain's transaction fee mechanism. The mechanism should fluctuate based on the supply of and demand for said resources to serve as a deterrent against spam and denial-of-service attacks.

With a fixed fee mechanism, users are provided with simplicity and predictability but network congestion and resource constraints are not taken into account. There is no incentive for users to withhold transactions since the cost is fixed regardless of the demand. The fee does not adjust the execution and inclusion fee of transactions to the market clearing price.

The C-Chain, in [Apricot Phase 3](https://medium.com/avalancheavax/apricot-phase-three-c-chain-dynamic-fees-432d32d67b60), employs a dynamic fee mechanism to raise the price during periods of high demand and lowering the price during periods of low demand. As the price gets too expensive, network utilization will decrease, which drops the price. This ensures the execution and inclusion fee of transactions closely matches the market clearing price.

The X-Chain and P-Chain currently operate under a fixed fee mechanism. To more robustly handle spikes in load, they should be migrated to a dynamic fee mechanism.

## Specification

### Dimensions

There are four dimensions that will be used to approximate the computational cost of, or "gas" consumed in, a transaction:

1. Bandwidth $B$ is the amount of network bandwidth used for transaction broadcast. This is set to the size of the transaction in bytes.
2. Reads $R$ is the number of state/database reads used in transaction execution.
3. Writes $W$ is the number of state/database writes used in transaction execution.
4. Compute $C$ is the total amount of compute used to verify and execute a transaction.

The gas consumed $G$ in a transaction is:

$$G = B + R + W + C$$

TODO: Each dimension will not be equally weighted as shown above. The correct weights for each dimension must be specified prior to this ACP being considered "Implementable"

A future ACP could remove the merging of these dimensions to granularly meter usage of each resource in a multidimensional scheme.

### Mechanism

This mechanism aims to maintain a target gas consumption $T$ per second and adjusts the fee based on the excess gas consumption $x$, defined as the difference between the current gas consumption and $T$.

At the start of building/executing block $b$, $x$ is updated:

$$x = \max(x - (T \cdot \Delta t), 0)$$

Where $\Delta t$ is the number of seconds between $b$'s block timestamp and $b$'s parent's block timestamp.

The gas price for block $b$ is:

$$M \cdot \exp\left(\frac{x}{K}\right)$$

Where:

- $M$ is the minimum gas price
- $\exp\left(x\right)$ is an approximation of $e^x$
- $K$ is a constant to control the rate of change of the gas price

After processing block $b$, $x$ is updated with the total gas consumed in the block $G$:

$$x = x + G$$

Whenever $x$ increases by $K$, the gas price increases by a factor of `~2.7`. If the gas price gets too expensive, average gas consumption drops, and $x$ starts decreasing, dropping the price. The gas price constantly adjusts to make sure that, on average, the blockchain consumes $T$ gas per second.

A [leaky bucket](https://en.wikipedia.org/wiki/Leaky_bucket) is employed to meter the maximum rate of gas consumption. Define $L$ as the capacity of the bucket, $S$ as the number of seconds for the bucket to leak $L$ gas ($\frac{L}{S}$ gas leaked per second), and $r$ as the amount of gas that can be added to the bucket without it overflowing. At the beginning of processing block $b$, $r$ is set:

$$r = \min\left(r + \frac{L \cdot \Delta{t}}{S}, L\right)$$

Where $\Delta t$ is the number of seconds between $b$'s block timestamp and $b$'s parent's block timestamp. The maximum gas consumed in a given $\Delta{t}$ is $r + \frac{L \cdot \Delta{t}}{S}$. The upper bound across all $\Delta{t}$ is $L + \frac{L \cdot \Delta{t}}{S}$.

After processing block $b$, the total gas consumed in $b$, or $G$, will be known. If $G \gt r$, $b$ is considered an invalid block. If $b$ is a valid block, $r$ is updated:

$$r = r - G$$

A block gas limit does not need to be set as it is implicitly derived from $r$.

The parameters at activation are:

| Parameter | X-Chain | P-Chain |
| - | - | - |
| $T$ - target gas consumed per second | TODO | TODO |
| $M$ - minimum gas price | TODO | TODO |
| $K$ - gas price update constant | TODO | TODO |
| $L$ - gas limit constant | TODO | TODO |
| $S$ - gas limit time period | TODO | TODO |

As the network gains capacity to handle additional load, this algorithm can be tuned to increase the gas consumption rate.

#### A note on $e^x$

There is a subtle reason why an exponential adjustment function was chosen: The adjustment function should be _equally_ reactive irrespective of the actual fee.

Define $b_n$ as the current block's gas fee, $b_{n+1}$ as the next block's gas fee, and $x$ as the excess gas consumption.

Let's use a linear adjustment function:

$$b_{n+1} = b_n + 10x$$

Assume $b_n = 100$ and the current block is 1 unit above target utilization, or $x = 1$. Then, $b_{n+1} = 100 + 10 \cdot 1 = 110$, an increase of `10%`. If instead $b_n = 10,000$, $b_{n+1} = 10,000 + 10 \cdot 1 = 10,010$, an increase of `0.1%`. The fee is _less_ reactive as the fee increases. This is because the rate of change _does not scale_ with $x$.

Now, let's use an exponential adjustment function:

$$b_{n+1} = b_n \cdot e^x$$

Assume $b_n = 100$ and the current block is 1 unit above target utilization, or $x = 1$. Then, $b_{n+1} = 100 \cdot e^1 \approx 271.828$, an increase of `171%`. If instead $b_n = 10,000$, $b_{n+1} = 10,000 \cdot e^1 \approx 27,182.8$, an increase of `171%` again. The fee is _equally_ reactive as the fee increases. This is because the rate of change _scales_ with $x$.

### Block Building Procedure

When a transaction is constructed on the X-Chain and P-Chain, the amount of $AVAX burned is given by `sum($AVAX outputs) - sum($AVAX inputs)`. The amount of gas consumed by the transaction can be deterministically calculated after construction. Dividing the amount of $AVAX burned by the amount of gas consumed yields the maximum gas price that the transaction can pay.

Instead of using a FIFO queue for the mempool (like the X-Chain and P-Chain do now), the mempool should use a priority queue ordered by the maximum gas price of each transaction. This ensures that higher paying transactions are included first.

## Backwards Compatibility

Modification of a fee mechanism is an execution change and requires a mandatory upgrade for activation. Implementers must take care to not alter the execution behavior prior to activation.

After this ACP is activated, any transaction issued on the P-Chain or X-Chain must account for the fee mechanism defined above. Users are responsible for reconstructing their transactions to include a larger fee for quicker inclusion when the fee increases.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable".

## Security Considerations

The current fixed fee mechanism on the X-Chain and P-Chain does not robustly handle spikes in load. Migrating these chains to a dynamic fee mechanism will ensure that load is properly priced given allotted processing capacity.

## Acknowledgements

Thank you to [@aaronbuchwald](https://github.com/aaronbuchwald) and [@patrick-ogrady](https://github.com/patrick-ogrady) for providing feedback prior to publication.

Thank you to the authors of [EIP-4844](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md) for creating the fee design that inspired the above mechanism.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
