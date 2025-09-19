| ACP           | 236                                                         |
|:--------------|:------------------------------------------------------------|
| **Title**     | Continuous Staking                                          |
| **Author(s)** | Razvan Angheluta ([@rrazvan1](https://github.com/rrazvan1)) |
| **Status**    |                                                             |
| **Track**     | Standards                                                   |

## Abstract

This proposal introduces continuous staking for validators on the Avalanche P-Chain. Validators can stake their tokens
continuously, allowing their stake to compound over time, accruing rewards once per specified cycle.

## Motivation

The current staking system on the Avalanche P-Chain restricts flexibility for stakers, limiting their ability to respond
to changing market conditions or liquidity needs. Managing a large number of nodes is also challenging, as re-staking at
the end of each period is labor-intensive, time-consuming, and poses security risks due to
the required transaction signing. Additionally, tokens can remain idle at the end of a staking period
until stakers initiate the necessary transactions to stake them again.

## Specification

Continuous staking introduces a mechanism that allows validators to remain staked indefinitely, without having to
manually submit new staking transactions at the end of each period.

Instead of committing to a fixed end time upfront, validators specify a cycle duration (period) when they
submit an `AddContinuousValidatorTx`. At the end of each cycle, the validator is automatically re-staked for a new cycle
of the same duration, unless the validator has submitted a `StopContinuousValidatorTx`.

Rewards accrue once per cycle, and they are automatically added to principal in subsequent cycles.

At the end of each cycle, if the updated stake weight (previous stake + staking rewards + delegatee rewards) exceeds the
maximum stake limit defined in the network configuration, the excess amount is automatically withdrawn and sent to the
wallets specified in the original transaction.

Note: Submitting an `AddContinuousValidatorTx` immediately followed by a `StopContinuousValidatorTx` replicates
the behavior of the current staking system.

### New P-Chain Transaction Types

The following new transaction types are introduced on the P-Chain to support this functionality:

#### AddContinuousValidatorTx

```golang
type AddContinuousValidatorTx struct {
  // Metadata, inputs and outputs
  BaseTx `serialize:"true"`
  
  // Node ID of the validator
  ValidatorNodeID ids.NodeID `serialize:"true" json:"nodeID"`
  
  // Period (in seconds).
  Period uint64 `serialize:"true" json:"period"`
  
  // [Signer] is the BLS key for this validator.
  Signer signer.Signer `serialize:"true" json:"signer"`
  
  // Where to send staked tokens when done validating
  StakeOuts []*avax.TransferableOutput `serialize:"true" json:"stake"`
  
  // Where to send validation rewards when done validating
  ValidatorRewardsOwner fx.Owner `serialize:"true" json:"validationRewardsOwner"`
  
  // Where to send delegation rewards when done validating
  DelegatorRewardsOwner fx.Owner `serialize:"true" json:"delegationRewardsOwner"`
  
  // Fee this validator charges delegators as a percentage, times 10,000
  // For example, if this validator has DelegationShares=300,000 then they
  // take 30% of rewards from delegators
  DelegationShares uint32 `serialize:"true" json:"shares"`
  
  // Weight of this validator used when sampling
  Wght uint64 `serialize:"true" json:"weight"`
}

```

#### StopContinuousValidatorTx

```golang
type StopContinuousValidatorTx struct {
  // Metadata, inputs and outputs
  BaseTx `serialize:"true"`
  
  // ID of the tx that created the continuous validator.
  TxID ids.ID `serialize:"true" json:"txID"`
  
  // Authorizes this validator to be stopped.
  // It is a BLS Proof of Possession signature of the TxID using validator key.
  StopSignature [bls.SignatureLen]byte `serialize:"true" json:"stopSignature"`
}
```

`StopSignature` is the BLS Proof of Possession signature of the tx ID of `AddContinuousValidatorTx` using the validator
key.

## Backwards Compatibility

This change requires a network upgrade to make sure that all validators are able to verify and execute the new
introduced transactions.

Another subsequent ACP could eventually remove `AddPermissionlessValidatorTx`, since the old staking behaviour is
replicable by using the newly added transactions.

## Considerations

Continuous staking makes it easier for users to keep their funds staked longer than with fixed-period staking, since it
involves fewer transactions, lower friction, and reduced risks.
Greater staking participation leads to stronger overall network security.

Validators benefit by not having to manually restart at the end of each cycle, which reduces transaction volume and the
risk of network congestion.

However, the risk per cycle slightly increases depending on cycle length and validator performance. For example, missing
five days in a one-year cycle may still yield rewards, whereas missing five days in a two-week cycle may affect rewards.

## Acknowledgements

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
