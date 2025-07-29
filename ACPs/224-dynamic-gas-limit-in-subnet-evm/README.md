| ACP | 224 |
| :--- | :--- |
| **Title** | Introduce ACP-176-Based Dynamic Gas Limits and Fee Manager Precompile in Subnet-EVM  |
| **Author(s)** |Ceyhun Onur ([@ceyonur](https://github.com/ceyonur)), Michael Kaplan ([@michaelkaplan13](https://github.com/michaelkaplan13)) |
| **Status** | Proposed ([Discussion](POPULATED BY MAINTAINER, DO NOT SET)) |
| **Track** | Standards |

## Abstract

Proposes implementing [ACP-176](https://github.com/avalanche-foundation/ACPs/blob/aa3bea24431b2fdf1c79f35a3fd7cc57eeb33108/ACPs/176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md) in Subnet-EVM, along with the addition of a new optional `ACP224FeeManagerPrecompile` that can be used configure fee parameters on-chain dynamically after activation, in the same way that the existing `FeeManagerPrecompile` can be used today prior to ACP-176.

## Motivation

ACP-176 updated the EVM dynamic fee mechanism to more accurately achieve the target gas consumption on-chain. It also added a mechanism for the target gas consumption rate to be dynamically updated. Until now, ACP-176 was only added to Coreth (C-Chain), primarily because most L1s prefer to control their fees and gas targets through the `FeeManagerPrecompile` and `FeeConfig` in genesis chain configuration.

[ACP-194](https://github.com/avalanche-foundation/ACPs/blob/aa3bea24431b2fdf1c79f35a3fd7cc57eeb33108/ACPs/194-streaming-asynchronous-execution/README.md) (SAE) depends on having a gas target and capacity mechanism aligned with ACP-176. Specifically, there must be a known gas capacity added per second, and maximum gas capacity. The existing windower fee mechanism employed by Subnet-EVM does not provide these properties because it does not have a fixed capacity rate, making it difficult to calculate worst-case bounds for gas prices. As such, adding ACP-176 into Subnet-EVM is a functional requirement for L1s to be able to use SAE in the future. Adding ACP-176 fee dynamics to Subnet-EVM also has the added benefit of aligning with Coreth such that only a single mechanism needs to be maintained on a go-forwards basis.

While both ACP-176 and ACP-194 will be required upgrades for L1s, this ACP aims to provide similar controls for chains with a new precompile. A new dynamic fee configuration and fee manager precompile that maps well into the ACP-176 mechanism will be added, optionally allowing admins to adjust fee parameters dynamically.

## Specification

### ACP-176 Parameters

This ACP uses same parameters as in the [ACP-176 specification](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md#configuration-parameters), and allows their values to be configured on a chain-by-chain basis. The parameters and their current values used by the C-Chain are as follows:

| Parameter | Description | C-Chain Configuration |
| :--- | :--- | :--- |
| T | target gas consumed per second | dynamic |
| R | gas capacity added per second | 2*T |
| C | maximum gas capacity | 10*T |
| P | minimum target gas consumption per second | 1,000,000 |
| D | target gas consumption rate update constant | 2^25 |
| Q | target gas consumption rate update factor change limit | 2^15 |
| M | minimum gas price | 1∗10^−18 AVAX |
| K | initial gas price update factor | 87*T |

### Prior Subnet-EVM Fee Configuration Parameters

Prior to this ACP, the Subnet-EVM fee configuration and fee manager precompile used the following parameters to control the fee mechanism:

**GasLimit**:
  Sets the max amount of gas consumed per block.

**TargetBlockRate**:
  Sets the target rate of block production in seconds used for fee adjustments. If the actual block rate is faster than this target, block gas cost will be increased, and vice versa.

**MinBaseFee**:
  The minimum base fee sets a lower bound on the EIP-1559 base fee of a block. Since the block's base fee sets the minimum gas price for any transaction included in that block, this effectively sets a minimum gas price for any transaction.

**TargetGas**:
  Specifies the targeted amount of gas (including block gas cost) to consume within a rolling 10s window. When the dynamic fee algorithm observes that network activity is above/below the `TargetGas`, it increases/decreases the base fee proportionally to how far above/below the target actual network activity is.

**BaseFeeChangeDenominator**:
  Divides the difference between actual and target utilization to determine how much to increase/decrease the base fee. A larger denominator indicates a slower changing, stickier base fee, while a lower denominator allows the base fee to adjust more quickly.

**MinBlockGasCost**:
  Sets the minimum amount of gas to charge for the production of a block.

**MaxBlockGasCost**:
  Sets the maximum amount of gas to charge for the production of a block.

**BlockGasCostStep**:
  Determines how much to increase/decrease the block gas cost depending on the amount of time elapsed since the previous block. If the block is produced at the target rate, the block gas cost will stay the same as the block gas cost for the parent block. If it is produced faster/slower, the block gas cost will be increased/decreased by the step value for each second faster/slower than the target block rate accordingly.
  Note: if the `BlockGasCostStep` is set to a very large number, it effectively requires block production to go no faster than the `TargetBlockRate`.
  Ex: if a block is produced two seconds faster than the target block rate, the block gas cost will increase by `2 * BlockGasCostStep`.

### ACP-176 Parameters in Subnet-EVM

ACP-176 will make `GasLimit` and `BaseFeeChangeDenominator` configurations obsolete in Subnet-EVM.

`TargetBlockRate`, `MinBlockGasCost`, `MaxBlockGasCost`, and `BlockGasCostStep` will be kept same because ACP-176 still uses block gas cost to control the block production rate and surcharge for producing a block faster than the target rate. Subnet-EVM is configured to use following default values, which will be kept same:
- `TargetBlockRate`: 2 (seconds)
- `MinBlockGasCost`: 0 (gas)
- `MaxBlockGasCost`: 1,000,000 (gas)
- `BlockGasCostStep`: 200,000 (gas)

`MinGasPrice` is equivalent to `M` in ACP-176 and will be used to set the minimum gas price for ACP-176. This is similar to `MinBaseFee` in old Subnet-EVM fee configuration, and roughly gives the same effect. Currently default value is `25 * 10^-18^` (25 nAVAX/Gwei). This default will be changed to the minimum possible denomination of the native EVM asset (1 Wei), which is aligned with the C-Chain.

`TargetGas` is equivalent to `T` (target gas consumed per second) in ACP-176 and will be used to set the target gas consumed per second for ACP-176.

`MaxCapacityFactor` is equivalent to factor in `C` in ACP-176 and controls the maximum gas capacity (i.e block gas limit). This determines the `C` as `C = MaxCapacityFactor * T`. The default value will be 10, which is aligned with the C-Chain.

`TimeToDouble` will be used to control the speed of the fee adjustment (`K`). This determines the `K` as `K = (RMult-1) * 1/ln2 * TimeToDouble`, where `RMult` is the factor in `R` which is defined as 2. The default value for `TimeToDouble` will be 60 (seconds), making `K=~87*T`, which is aligned with the C-Chain.

As a result parameters will be set as follows:

| Parameter | Description | Default Value | Is Configurable |
| :--- | :--- | :--- | :--- |
| T | target gas consumed per second | 1,000,000 | :white_check_mark: |
| R | gas capacity added per second | 2*T | :x:
| C | maximum gas capacity | 10*T | :white_check_mark: Through `MaxCapacityFactor` (default 10)
| P | minimum target gas consumption per second | 1,000,000 | :x:
| D | target gas consumption rate update constant | 2^25 | :x:
| Q | target gas consumption rate update factor change limit | 2^15 | :x:
| M | minimum gas price | 1 Wei | :white_check_mark:
| K | initial gas price update factor | ~87*T | :white_check_mark: Through `TimeToDouble` (default 60s)

The gas capacity added per second (`R`) always being equal to `2*T` keeps it such that the gas price is capable of increases and decrease at the same rate. The values of `Q` and `D` affect the magnitude of change to `T` that each block can have, and the granularity at which the target gas consumption rate can be updated. The proposed values match the C-Chain, allowing each block to modify the current gas target by roughly $\frac{1}{1024}$ of its current value. This has provided sufficient responsiveness and granularity as is, removing the need to make `D` and `Q` dynamic or configurable. Similarly, 1,000,000 gas/second should be a low enough minimum target gas consumption for any EVM L1. The target gas for a given L1 will be able to be increased from this value dynamically and has no maximum.

#### Adjustment to ACP-176 calculations for price discovery

// TODO: MinGasPrice requires a binary search to find the correct value (similar to how we do for DesiredTargetExcess in Coreth)

### Genesis Configuration

There will be a new genesis chain configuration to set the parameters for the chain without requiring the ACP176FeeManager precompile to be activated. This will be similar to the existing fee configuration parameters in chain configuration. If there is no genesis configuration for the new fee parameters the default values for C-Chain will be used. This will look like the following:

```json
{
  ...
  "acp224Timestamp": ...,
  "acp224FeeConfig": {
    "minGasPrice": ...,
    "maxCapacityFactor": ...,
    "timeToDouble": ...,
    "targetBlockRate": ...,
    "minBlockGasCost": ...,
    "maxBlockGasCost": ...,
    "blockGasCostStep": ...
  }
}

```

### `ACP224FeeManagerPrecompile`

A new fee manager precompile will be required to dynamically changing the parameters. The precompile will off similar controls as the existing `FeeManagerPrecompile` implemented in Subnet-EVM [here](https://github.com/ava-labs/subnet-evm/tree/53f5305/precompile/contracts/feemanager). The solidity interface will be as follows:

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;
import "./IAllowList.sol";

interface IACP224FeeManager is IAllowList {
  struct FeeConfig {
    uint256 targetGas;
    uint256 minGasPrice;
    uint256 maxCapacityFactor;
    uint256 timeToDouble;
    uint256 targetBlockRate;
    uint256 minBlockGasCost;
    uint256 maxBlockGasCost;
    uint256 blockGasCostStep;
  }
  event FeeConfigUpdated(address indexed sender, FeeConfig oldFeeConfig, FeeConfig newFeeConfig);

  // Set fee config fields to contract storage
  function setFeeConfig(
    FeeConfig calldata config
  ) external;

  // Get fee config from the contract storage
  function getFeeConfig()
    external
    view
    returns (FeeConfig memory config);

  // Get the last block number changed the fee config from the contract storage
  function getFeeConfigLastChangedAt() external view returns (uint256 blockNumber);
}
```

For chains with the precompile activated, `setFeeConfig` can be used to dynamically change each of the values in the fee configurations. Importantly, any updates made via calls to `setFeeConfig` in a transaction will take effect only as of _settlement_ of the transaction, not as of _acceptance_ or _execution_ (for transaction life cycles/status, refer to ACP-194 [here](https://github.com/avalanche-foundation/ACPs/tree/61d2a2a/ACPs/194-streaming-asynchronous-execution#description)). This ensures that all nodes apply the same worst-case bounds validation on transactions being accepted into the queue, since the worst-case bounds are effected by changes to the fee configuration.

Similar to the [desired target excess calculation in Coreth](https://github.com/ava-labs/coreth/blob/0255516f25964cf4a15668946f28b12935a50e0c/plugin/evm/upgrade/acp176/acp176.go#L170), which takes a node's desired gas target and calculates its desired target excess value, the `ACP224FeeManagerPrecompile` will use binary search to determine the resulting dynamic target excess value given the `targetGas` value passed to `setFeeConfig`. All blocks accepted after the settlement of such a call must have the correct target excess value as derived from the binary search result.

For chains without the precompile activated, the target gas consumption will be able to be dynamically updated by validators by setting their `gas-target` configuration values, the same as on the C-Chain.

Block building logic can follow the below diagram for determining the target excess of blocks.
```mermaid
flowchart TD
    A[ACP-224 activated] --> B{Is ACP224FeeManager precompile active?}

    B -- Yes --> C[Use `targetExcess` from precompile storage at latest settled root]

    B -- No --> D{Is `gas-target` set in node chain config file?}
    D -- Yes --> E[Calculate `targetExcess` from configured preference and allowed update bounds]

    D -- No --> F{Does parent block have ACP176 fields?}
    F -- Yes --> G[Use parent block ACP176 gas target]
    F -- No --> H[Use `MinTargetPerSecond` (i.e. `P`)]
```

## Backwards Compatibility

ACP-224 will require a network update in order to activate the new fee mechanism. Another activation will also be required to activate the new fee manager precompile. The activation of precompile should never occur before the activation of ACP-224 (the fee mechanism) since the precompile depends on ACP-224’s fee update logic to function correctly.

Activation of ACP-224 mechanism will deactivate the prior fee mechanism and the prior fee manager precompile. This ensures that there is no ambiguity or overlap between legacy and new pricing logic In order to provide a configuration for existing networks, a network upgrade override for both activation time and ACP-176 configuration parameters will be introduced.

These upgrades will be optional at the moment. However with introduction of ACP-194 (SAE), it will be required to activate this ACP; otherwise the network will not be able to use ACP-194.

## Reference Implementation


## Security Considerations

Generally it has same security considerations as [ACP-176](https://github.com/avalanche-foundation/ACPs/blob/main/ACPs/176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md#security-considerations). Due to the nature of these parameters are dynamic, the risk here is generally more than ACP-176 as it allows for more misconfiguration.

Potentially any misconfiguration of parameters could leave the network vulnerable to a DoS attack or over charge transactions. If `TargetGas` (`T`) is set too low, then the network might not be able to process bigger transactions. similarly if it is set too high, then validators might not be able to keep up in the processing of blocks.

## Open Questions

* Should activation of the `ACP224FeeManager` precompile disable the old precompile itself or should we require it to be disabled as a separate upgrade?

## Acknowledgements

* [Stephen Buttolph](https://github.com/StephenButtolph)
* [Arran Schlosberg](https://github.com/ARR4N)
* [Austin Larson](https://github.com/alarso16)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
