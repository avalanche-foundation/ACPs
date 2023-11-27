```text
ACP: 41
Title: Remove Pending Stakers
Author(s): Dhruba Basu <https://github.com/dhrubabasu>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/42
Status: Implementable
Track: Standards
```

## Abstract

Remove user-specified `StartTime` for stakers. Start the staking period for a staker as soon as their staking transaction is accepted. This greatly reduces the computational load on the P-chain, increasing the efficiency of all Avalanche Network validators.

## Motivation

Stakers currently set a `StartTime` for their staking period. This means that Avalanche Network Clients, like AvalancheGo, need to maintain a pending set of all stakers that have not yet started. This places a nontrivial amount of work on the P-chain:

- When a new delegator transaction is verified, the pending set needs to be checked to ensure that the validator they are delegating to will not exceed `MaxValidatorStake` while they are active
- When a new staker transaction is accepted, it gets added to the pending set
- When time is advanced on the P-chain, any stakers in the pending set whose `StartTime <= CurrentTime` need to be moved to the current set

By immediately starting every staker on acceptance, the validators do not have to do the above work when validating the P-chain. `MaxValidatorStake` will become an `O(1)` operation as only the current stake of the validator needs to be checked. The pending set can be fully removed.

## Specification

1. When adding a new staker, the current on-chain time should be used for the staker's start time.
2. When determining when to remove the staker from the staker set, the `EndTime` specified in the transaction should continue to be used. Staking transactions should now be rejected if it does not satisfy `MinStakeDuration <= EndTime - CurrentTime <= MaxStakeDuration`. `StartTime` will no longer be validated.

## Backwards Compatibility

Modifying the state transition of a transaction type is an execution change and requires a mandatory upgrade for activation. Implementors must take care to not alter the execution behavior prior to activation. This ACP only details the new state transition.

Current wallet implementations will continue to work as-is post-activation of this ACP since no transaction formats are modified or added. Wallet implementations may run into issues with their txs being rejected as a result of this ACP if `EndTime >= CurrentChainTime + MaxStakeDuration`. `CurrentChainTime` is guaratanteed to be >= the latest block timestamp on the P-chain.

## Reference Implementation

A reference implementation has not been created for this ACP since it deals with state management. Each ANC will need to adjust their execution step to follow the Specification detailed above. For AvalancheGo, this work is tracked in this PR: https://github.com/ava-labs/avalanchego/pull/2175

If modifications are made to the specification of the new execution behavior as part of the ACP process, the code must be updated prior to activation.

## Security Considerations

No security considerations.

## Open Questions

_How will stakers stake for `MaxStakeDuration` if they cannot determine their `StartTime`?_

As mentioned above, the beginning of your staking period is the block acceptance timestamp. Unless you can accurately predict the block timestamp, you will *not* be able to fully stake for `MaxStakeDuration`. This is an explicit trade-off to guarantee that stakers will receive their original stake + any staking rewards at `EndTime`.

Delegators can maximize their staking period by setting the same `EndTime` as the Validator they are delegating to.

## Straw Poll

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). [This PR must include a message + signature indicating ownership of a given amount of $AVAX](https://github.com/avalanche-foundation/ACPs#acp-straw-poll).

### Supporters
* `<message>/<signature>`

### Objectors
* `<message>/<signature>`

## Acknowledgements

Thanks to [@StephenButtolph](https://github.com/StephenButtolph) and [@abi87](https://github.com/abi87) for their feedback on these ideas.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).