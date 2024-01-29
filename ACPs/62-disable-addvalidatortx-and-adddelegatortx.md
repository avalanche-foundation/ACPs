```text
ACP: 62
Title: Disable `AddValidatorTx` and `AddDelegatorTx`
Author(s): Jacob Everly <https://twitter.com/JacobEv3rly>, Dhruba Basu <https://github.com/dhrubabasu>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/63
Status: Implementable
Track: Standards
```

## Abstract

Disable `AddValidatorTx` and `AddDelegatorTx` to push all new stakers to use `AddPermissionlessValidatorTx` and `AddPermissionlessDelegatorTx`. `AddPermissionlessValidatorTx` requires validators to register a BLS key. Wide adoption of registered BLS keys accelerates the timeline for future P-Chain upgrades. Additionally, this reduces the number of ways to participate in Primary Network validation from two to one.

## Motivation

`AddPermissionlessValidatorTx` and `AddPermissionlessDelegatorTx` were activated on the Avalanche Network in October 2022 with Banff (v1.9.0). This unlocked the ability for Subnet creators to activate Proof-of-Stake validation using their own token on their own Subnet. See more details about Banff [here](https://medium.com/avalancheavax/banff-elastic-subnets-44042f41e34c).

These new transaction types can also be used to register a Primary Network validator, leaving two redundant transactions: `AddValidatorTx` and `AddDelegatorTx`.

[`AddPermissionlessDelegatorTx`](https://github.com/ava-labs/avalanchego/blob/v1.10.18/vms/platformvm/txs/add_permissionless_delegator_tx.go#L25-L37) contains the same fields as [`AddDelegatorTx`](https://github.com/ava-labs/avalanchego/blob/v1.10.18/vms/platformvm/txs/add_delegator_tx.go#L29-L39) with an additional `Subnet` field.

[`AddPermissionlessValidatorTx`](https://github.com/ava-labs/avalanchego/blob/v1.10.18/vms/platformvm/txs/add_permissionless_validator_tx.go#L35-L59) contains the same fields as [`AddValidatorTx`](https://github.com/ava-labs/avalanchego/blob/v1.10.18/vms/platformvm/txs/add_validator_tx.go#L29-L42) with additional `Subnet` and `Signer` fields. `RewardsOwner` was also split into `ValidationRewardsOwner` and `DelegationRewardsOwner` letting validators divert rewards they receive from delegators into a separate rewards owner.

By disabling support of `AddValidatorTx`, all new validators on the Primary Network must use `AddPermissionlessValidatorTx` and register a BLS key with their NodeID. As more validators attach BLS keys to their nodes, future upgrades using these BLS keys can be activated through the ACP process. BLS keys can be used to efficiently sign a common message via [Public Key Aggregation](https://crypto.stanford.edu/~dabo/pubs/papers/BLSmultisig.html). Applications of this include, but are not limited to:

- **Arbitrary Subnet Rewards**: The P-Chain currently restricts Elastic Subnets to follow the reward curve defined in a `TransformSubnetTx``. With sufficient BLS key adoption, Elastic Subnets can define their own reward curve and reward conditions. The P-Chain can be modified to take in a message indicating if a Subnet validator should be rewarded with how many tokens signed with a BLS Multi-Signature.
- **Subnet Attestations**: Elastic Subnets can attest to the state of their Subnet with a BLS Multi-Signature. This can enable clients to fetch the current state of the Subnet without syncing the entire Subnet. `StateSync` enables clients to download chain state from peers up to a recent block near tip. However, it is up to the client to query these peers and resolve any potential conflicts in the responses. With Subnet Attestations, clients can query an API node to prove information about a Subnet without querying the Subnet's validators. This can especially be useful for [Subnet-Only Validators](./13-subnet-only-validators.md) to prove information about the C-Chain.

To accelerate future BLS-powered advancements in the Avalanche Network, this ACP aims to disable `AddValidatorTx` and `AddDelegatorTx` in Durango.

## Specification

`AddValidatorTx` and `AddDelegatorTx` should be marked as dropped when added to the mempool after activation. Any blocks including these transactions should be considered invalid.

## Backwards Compatibility

Disabling a transaction type is an execution change and requires a mandatory upgrade for activation. Implementers must take care to not alter the execution behavior prior to activation.

After this ACP is activated, any new issuance of `AddValidatorTx` or `AddDelegatorTx` will be considered invalid and dropped by the network. Any consumers of these transactions must transition to using `AddPermissionlessValidatorTx` and `AddPermissionlessDelegatorTx` to participate in Primary Network validation. The [Avalanche Ledger App](https://github.com/LedgerHQ/app-avalanche) supports both of these transaction types.

Note that `AddSubnetValidatorTx` and `RemoveSubnetValidatorTx` are unchanged by this ACP.

## Reference Implementation

An implementation disabling `AddValidatorTx` and `AddDelegatorTx` was created [here](https://github.com/ava-labs/avalanchego/pull/2662). Until activation, these transactions will continue to be accepted by AvalancheGo.

If modifications are made to the specification as part of the ACP process, the code must be updated prior to activation.

## Security Considerations

No security considerations.

## Open Questions

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Acknowledgements

Thanks to [@StephenButtolph](https://github.com/StephenButtolph) and [@patrick-ogrady](https://github.com/patrick-ogrady) for their feedback on these ideas.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
