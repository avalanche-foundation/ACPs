| ACP           | 236                                                         |
|:--------------|:------------------------------------------------------------|
| **Title**     | Continuous Staking                                          |
| **Author(s)** | Razvan Angheluta ([@rrazvan1](https://github.com/rrazvan1)) |
| **Status**    |                                                             |
| **Track**     | Standards                                                   |

## Abstract

This proposal introduces continuous staking for validators on the Avalanche P-Chain. Validators can stake their tokens
continuously, allowing their stake to compound over time.

## Motivation

The current staking system on the Avalanche P-Chain restricts flexibility for stakers, limiting their ability to respond
to changing market conditions or liquidity needs. Managing a large number of nodes is also challenging, as re-staking at
the end of each period is labor-intensive, time-consuming, and poses security risks due to
the required transaction signing. Additionally, tokens can remain idle at the end of a staking period
until stakers initiate the necessary transactions to stake them again.

## Specification

When validators wish to start staking, they submit an `AddContinuousValidatorTx`, specifying the desired cycle
duration (period). To stop staking at any time, they submit a `StopContinuousValidatorTx`.

Note: Submitting an AddContinuousValidatorTx immediately followed by a StopContinuousValidatorTx effectively replicates
the behavior of the current staking system.

### New P-Chain Transaction Types

The following new transaction types are introduced on the P-Chain to support this functionality:

- `AddContinuousValidatorTx`
- `StopContinuousValidatorTx`

```golang
type AddContinuousValidatorTx struct {
  // Metadata, inputs and outputs
  BaseTx `serialize:"true"`
  
  // Node ID of the validator
  ValidatorNodeID ids.NodeID `serialize:"true" json:"validatorNodeID"`
  
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

```golang
type StopContinuousValidatorTx struct {
  // Metadata, inputs and outputs
  BaseTx `serialize:"true"`
  
  // ID of the tx that created the continuous validator.
  TxID ids.ID `serialize:"true" json:"txID"`
  
  // Authorizes this validator to be stopped.
  // It is a BLS Proof of Possession signature using validator key of the TxID.
  StopSignature [bls.SignatureLen]byte `serialize:"true" json:"stopSignature"`
}
```

### New Transactions

- P-Chain
    - `AddContinuousValidatorTx`
    - `StopContinuousValidatorTx`

## Backwards Compatibility

This change requires a network upgrade to make sure that all validators are able to verify and execute the new
introduced transactions.

Another subsequent ACP could eventually remove `AddPermissionlessValidatorTx`, and use `AddContinuousValidatorTx` and
`StopContinuousValidatorTx` to replicate the same behaviour.

## Acknowledgements

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
