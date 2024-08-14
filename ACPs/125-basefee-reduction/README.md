| ACP | 125 |
| :--- | :--- |
| **Title** | Reduce C-Chain minimum base fee from 25 nAVAX to 1 nAVAX |
| **Author(s)** | Stephen Buttolph ([@StephenButtolph](https://github.com/StephenButtolph)), Darioush Jalali ([@darioush](https://github.com/darioush)) |
| **Status** | Implementable ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/127)) |
| **Track** | Standards |

## Abstract

Reduce the minimum base fee on the Avalanche C-Chain from 25 nAVAX to 1 nAVAX.

## Motivation

With dynamic fees, the gas price is supposed to be a result of a continuous auction such that the consumed gas per second converges to the target gas usage per second.

When dynamic fees were first introduced, safeguards were added to ensure the mechanism worked as intended, such as a relatively high minimum gas price and a maximum gas price.

The maximum gas price has since been entirely removed. The minimum gas price has been reduced significantly. However, the base fee is often observed pinned to this minimum. This shows that it is higher than what the market demands, and therefore it is artificially reducing network usage.

## Specification

The dynamic fee calculation currently must enforce a minimum base fee of 25 nAVAX.

This change proposes reducing the minimum base fee to 1 nAVAX upon the next network upgrade activation.

## Backwards Compatibility

Modifies the consensus rules for the C-Chain, therefore it requires a network upgrade.

## Reference Implementation

A draft implementation of this ACP for the coreth VM can be found [here](https://github.com/ava-labs/coreth/pull/604/files).

## Security Considerations

Lower gas costs may increase state bloat. However, we note that the dynamic fee algorithm responded appropriately during periods of high use (such as Dec. 2023), which gives reasonable confidence that enforcing a 25 nAVAX minimum fee is no longer necessary.

## Open Questions

N/A

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
