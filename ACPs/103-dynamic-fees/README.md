```text
ACP: 103
Title: Add Dynamic Fees to the X-Chain and P-Chain
Author(s): Dhruba Basu (https://github.com/dhrubabasu), Stephen Buttolph (https//github.com/StephenButtolph), Alberto Benegiamo (https://github.com/abi87)
Discussions-To: <GitHub Discussion URL (POPULATED BY MAINTAINER, DO NOT SET)>
Status: Proposed
Track: Standards
```

## Abstract

Introduce a dynamic fee mechanism to the X-Chain and the P-Chain. Preview a future transition to a multidimensional fee mechanism.

## Motivation

Under a fixed fee mechanism, there is a greater likelihood of higher network congestion. During periods of high demand, there is no economic incentive for users to withhold issuance of their transactions, extending the period of high demand.

The C-Chain, in [Apricot Phase 3](https://medium.com/avalancheavax/apricot-phase-three-c-chain-dynamic-fees-432d32d67b60), employs a dynamic fee mechanism to raise the price during periods of high demand and lowering the price during periods of low demand. As the price gets too expensive, network utilization will decrease, which drops the price.

The X-Chain and P-Chain currently operate under a fixed fee mechanism. To better account for increased load that may result from [ACP-77](../77-reinventing-subnets/README.md), they should be migrated to a dynamic fee mechanism.

## Specification

### Dimensions

There are four dimensions that will be used to approximate the computational cost ("gas" usage) of a transaction:

1. `Bandwidth` ($B$) is the amount of network bandwidth used in transaction broadcast. This is set to the size of the transaction in bytes.
2. `Reads` ($R$) is the number of state/database reads used in transaction execution.
3. `Writes` ($W$) is the number of state/database writes used in transaction execution.
4. `Compute` ($C$) is the total amount of compute used in transaction execution. This is set to the number of signatures verified during transaction execution.

For each transaction, gas ($G$) can be computed (TODO: The weights for each dimension must be specified prior to this ACP being considered "Implementable"):

$$G = B + R + W + C$$

A future ACP should remove the gas standardization to granularly meter usage of each resource in a multidimensional scheme.

### Mechanism

This mechanism aims to have a target gas usage ($T$) per second and adjusts based on the excess gas usage ($x$), defined as the difference between the current gas usage and $T$.

At the start of building/executing block $b$, $x$ is updated:

$$x = \max(x - (T \cdot \Delta t), 0)$$

Where $\Delta t$ is the number of seconds between $b$ and $b$'s parent block.

The required fee per gas for block $b$ is:

$$M \cdot \exp\left(\frac{x}{K}\right)$$

Where:

- $M$ is the minimum gas price
- $\exp\left(x\right)$ is an approximation of $e^x$
- $K$ is a constant to control the rate of change for gas price

After processing block $b$, $x$ is updated with the total gas in the block $G$:

$$x = x + G$$

Whenever $x$ increases by $K$, the gas price increases by a factor of `~2.7`. If the gas price gets too expensive, average usage drops, and $x$ starts decreasing, automatically dropping the price again. The gas price constantly adjusts to make sure that, on average, the blockchain uses $T$ gas per second.

$L$ is defined as the maximum gas per second that the blockchain has capacity for. For monotonically increasing blocktimes (as on P-Chain and X-Chain), there may be multiple blocks issued at the same on-chain timestamp. $L$ limits the amount of gas that can be used at $t$, placing a bound on the amount that the required fee can increase from $t$ to $t'$. Once the chain has used $L$ amount of gas at time $t$, a new block will only be considered valid if its timestamp is at least $t + 1$.

The initial parameters for the X-Chain will be set to:

| Parameter | Value |
| - | - |
| $M$ - minimum gas price | TODO |
| $T$ - target gas per second | TODO |
| $L$ - max gas per second | TODO |
| $K$ - change constant | TODO |

The initial parameters for the P-Chain will be set to:

| Parameter | Value |
| - | - |
| $M$ - minimum gas price | TODO |
| $T$ - target gas per second | TODO |
| $L$ - max gas per second | TODO |
| $K$ - change constant | TODO |

As the network gains capacity to handle additional load (through the activation of future protocol enhancements), this algorithm can be tuned to increase the number of transactions that can be processed at a specific fee rate.

#### A note on $e^x$

There is a subtle reason why an exponential adjustment function was chosen: The adjustment function should be _equally_ reactive irrespective of the actual fee.

Define $b_n$ as the current block's required fee, $b_{n+1}$ as the next block's required fee, and $x$ as the excess gas usage.

Let's use a linear adjustment function:

$$b_{n+1} = b_n + 10x$$

Assume $b_n = 100$ and the current block is 1 unit above target utilization, or $x = 1$. Then, $b_{n+1} = 100 + 10 \cdot 1 = 110$, an increase of `10%`. If instead $b_n = 10,000$, $b_{n+1} = 10,000 + 10 \cdot 1 = 10,010$, an increase of `0.00001%`. The fee is _less_ reactive as the fee increases. This is because the rate of change is constant: $\frac{d}{dx}10x = 10$.

Now, let's use an exponential adjustment function:

$$b_{n+1} = b_n \cdot e^x$$

Assume $b_n = 100$ and the current block is 1 unit above target utilization, or $x = 1$. Then, $b_{n+1} = 100 \cdot e^1 \approx 271.828$, an increase of `271%`. If instead $b_n = 10,000$, $b_{n+1} = 10,000 \cdot e^1 \approx 27,182.8$, an increase of `271%` again. The fee is _equally_ reactive as the fee increases. This is because the rate of change scales with $x$: $\frac{d}{dx}e^x = e^x$.

### Block Building Procedure

When a transaction is constructed on the X-Chain and P-Chain, the amount of $AVAX burned in a transaction is given by `sum($AVAX outputs) - sum($AVAX inputs)`. The amount of gas used by the transaction can also be deterministically calculated after construction. Dividing the amount of $AVAX burned by the amount of gas yields the maximum gas price that the transaction can pay.

Instead of using a FIFO queue for the mempool (like the X-Chain and P-Chain do now), the mempool should be a priority queue ordered by the maximum gas price of each transaction. This ensures that higher paying transactions are included first.

## Backwards Compatibility

Modfication of a fee mechanism is an execution change and requires a mandatory upgrade for activation. Implementers must take care to not alter the execution behavior prior to activation.

After this ACP is activated, any transaction issued on the P-Chain or X-Chain must account for the fee mechanism defined above. There is a concern of underpriced transactions filling the mempool and causing memory issues on nodes. To alleviate memory pressure, the mempool should be periodically cleared of underpriced transactions. Users are responsible for re-broadcasting underpriced transactions when the fee lowers or re-constructing their transaction to include a larger fee.

## Reference Implementation

A full reference implementation has not been provided yet. It must be provided prior to this ACP being considered "Implementable".

## Security Considerations

The existing fixed fee mechanism on the X-Chain and P-Chain have led to periods of instability when larger transactions are accepted. Metering these transactions according to their computational cost or "gas" usage will remove those periods of instability at the cost of higher fees for users.

## Acknowledgements

Thank you to [@aaronbuchwald](https://github.com/aaronbuchwald) and [@patrick-ogrady](https://github.com/patrick-ogrady) for providing feedback prior to publication.

Thank you to the authors of [EIP-4844](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-4844.md) for creating the fee design that inspired the above mechanism.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
