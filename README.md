<div align="center">
  <img width="80%" src="LOGO.png">
</div>

## What is an Avalanche Community Proposal (ACP)?

An Avalanche Community Proposal is a concise document that introduces a change or best practice for adoption on the [Avalanche Network](https://www.avax.com). ACPs should provide clear technical specifications of any proposals and a compelling rationale for their adoption.

ACPs are an open framework for proposing improvements and gathering consensus around changes to the Avalanche Network. ACPs can be proposed by anyone and will be merged into this repository as long as they are well-formatted and coherent. Once an overwhelming majority of the Avalanche Network/Community have [signaled their support for an ACP](https://docs.avax.network/nodes/configure/avalanchego-config-flags#avalanche-community-proposals), it may be scheduled for activation on the Avalanche Network by Avalanche Network Clients (ANCs). It is ultimately up to members of the Avalanche Network/Community to adopt ACPs they support by running a compatible ANC, such as [AvalancheGo](https://github.com/ava-labs/avalanchego). 

## ACP Tracks

There are three kinds of ACP:

* A `Standards Track` ACP describes a change to the design or function of the Avalanche Network, such as a change to the P2P networking protocol, P-Chain design, Subnet architecture, or any change/addition that affects the interoperability of Avalanche Network Clients (ANCs).
* A `Best Practices Track` ACP describes a design pattern or common interface that should be used across the Avalanche Network to make it easier to integrate with Avalanche or for Subnets to interoperate with each other. This would include things like proposing a smart contract interface, not proposing a change to how smart contracts are executed.
* A `Meta Track` ACP describes a change to the ACP process or suggests a new way for the Avalanche Community to collaborate.
* A `Subnet Track` ACP describes a change to a particular Subnet. This would include things like configuration changes or coordinated Subnet upgrades.

## ACP Statuses

There are four statuses of an ACP:

* A `Proposed` ACP has been merged into the main branch of the ACP repository. It is actively being discussed by the Avalanche Community and may be modified based on feedback.
* An `Implementable` ACP is considered "ready for implementation" by the author(s) and will no longer change meaningfully from its current form (which would require a new ACP).
* An `Activated` ACP has been activated on the Avalanche Network via a coordinated upgrade by the Avalanche Community. Once an ACP is `Activated`, it is locked.
* A `Stale` ACP has been abandoned by its author(s) because it is not supported by the Avalanche Community or has been replaced with another ACP.

## ACP Workflow

### Step 0: Think of a Novel Improvement to Avalanche

The ACP process begins with a new idea for Avalanche. Each potential ACP must have an author(s): someone who writes the ACP using the style and format described below, shepherds the associated GitHub Discussion, and attempts to build consensus around the idea. Note that ideas and any resulting ACP is public. Authors should not post any ideas or anything in an ACP that the Author wants to keep confidential or to keep ownership rights in (such as intellectual property rights).

### Step 1: Post Your Idea to [GitHub Discussions](https://github.com/avalanche-foundation/ACPs/discussions/categories/ideas)

The author(s) should first attempt to ascertain whether there is support for their idea by posting in the "Ideas" category of GitHub Discussions. Vetting an idea publicly before going as far as writing an ACP is meant to save both the potential author(s) and the wider Avalanche Community time. Asking the Avalanche Community first if an idea is original helps prevent too much time being spent on something that is guaranteed to be rejected based on prior discussions (searching the Internet does not always do the trick). It also helps to make sure the idea is applicable to the entire community and not just the author(s). Small enhancements or patches often don't need standardization between multiple projects; these don't need an ACP and should be injected into the relevant development workflow with a patch submission to the applicable ANC issue tracker.

### Step 2: Propose an ACP via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

Once the author(s) feels confident that an idea has a decent chance of acceptance, an ACP should be drafted and submitted as a pull request (PR). This draft must be written in ACP style as described below. It is highly recommended that a single ACP contain a single key proposal or new idea. The more focused the ACP, the more successful it tends to be. If in doubt, split your ACP into several well-focused ones. The PR number of the ACP will become its assigned number.

### Step 3: Build Consensus on [GitHub Discussions](https://github.com/avalanche-foundation/ACPs/discussions/categories/discussion) and Provide an Implementation (if Applicable)

ACPs will be merged by ACP maintainers if the proposal is generally well-formatted and coherent. ACP editors will attempt to merge anything worthy of discussion, regardless of feasibility or complexity, that is not a duplicate or incomplete. After an ACP is merged, an official GitHub Discussion will be opened for the ACP and linked to the proposal for community discussion. It is recommended for author(s) or supportive Avalanche Community members to post an accompanying non-technical overview of their ACP for general consumption in this GitHub Discussion. The ACP should be reviewed and broadly supported before a reference implementation is started, again to avoid wasting the author(s) and the Avalanche Community's time, unless a reference implementation will aid people in studying the ACP.

### Step 4: Mark ACP as `Implementable` via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

Once an ACP is considered complete by the author(s), it should be marked as `Implementable`. At this point, all open questions should be addressed and an associated reference implementation should be provided (if applicable). As mentioned earlier, the Avalanche Foundation meets periodically to recommend the ratification of specific ACPs but it is ultimately up to members of the Avalanche Network/Community to adopt ACPs they support by running a compatible Avalanche Network Client (ANC), such as [AvalancheGo](https://github.com/ava-labs/avalanchego).

### [Optional] Step 5: Mark ACP as `Stale` via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

An ACP can be superseded by a different ACP, rendering the original obsolete. If this occurs, the original ACP will be marked as `Stale`. ACPs may also be marked as `Stale` if the author(s) abandon work on it for a prolonged period of time (12+ months). ACPs may be reopened and moved back to `Proposed` if the author(s) restart work.

### Maintenance

ACP maintainers will only merge PRs updating an ACP if it is created or approved by at least one of the author(s). ACP maintainers are not responsible for ensuring ACP author(s) approve the PR. ACP author(s) are expected to review PRs that target their unlocked ACP (`Proposed` or `Implementable`). Any PRs opened against a locked ACP (`Activated` or `Stale`) will not be merged by ACP maintainers.

## What belongs in a successful ACP?

Each ACP must have the following parts:

* `Preamble`: Markdown table containing metadata about the ACP, including the ACP number, a short descriptive title, the author(s), and optionally the contact info for each author, etc.
* `Abstract`: Concise (~200 word) description of the ACP
* `Motivation`: Rationale for adopting the ACP and the specific issue/challenge/opportunity it addresses
* `Specification`: Complete description of the semantics of any change should allow any ANC/Avalanche Community member to implement the ACP
* `Security Considerations`: Security implications of the proposed ACP

Each ACP can have the following parts:

* `Open Questions`: Questions that should be resolved before implementation

Each `Standards Track` ACP must have the following parts:

* `Backwards Compatibility`: List of backwards incompatible changes required to implement the ACP and their impact on the Avalanche Community
* `Reference Implementation`: Code, documentation, and telemetry (from a local network) of the ACP change

Each `Best Practices Track` ACP can have the following parts:

* `Backwards Compatibility`: List of backwards incompatible changes required to implement the ACP and their impact on the Avalanche Community
* `Reference Implementation`: Code, documentation, and telemetry (from a local network) of the ACP change

### ACP Formats and Templates

Each ACP is allocated a unique subdirectory in the `ACPs` directory. The name of this subdirectory must be of the form `N-T` where `N` is the ACP number and `T` is the ACP title with any spaces replaced by hyphens. ACPs must be written in [markdown](https://daringfireball.net/projects/markdown/syntax) format and stored at `ACPs/N-T/README.md`. Please see the [ACP template](./ACPs/TEMPLATE.md) for an example of the correct layout.

### Auxiliary Files

ACPs may include auxiliary files such as diagrams or code snippets. Such files should be stored in the ACP's subdirectory (`ACPs/N-T/*`). There is no required naming convention for auxiliary files.

### Waived Copyright

ACP authors must waive any copyright claims before an ACP will be merged into the repository. This can be done by including the following text in an ACP:

```text
## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
```

## Proposals

_You can view the status of each ACP on the [ACP Tracker](https://github.com/orgs/avalanche-foundation/projects/1/views/1)._

| Number | Title |  Author(s) | Type |
|:-------|:------|:-------|:-----|
|[13](./ACPs/13-subnet-only-validators/README.md)|Subnet-Only Validators (SOVs)|Patrick O'Grady (contact@patrickogrady.xyz)|Standards|
|[20](./ACPs/20-ed25519-p2p/README.md)|Ed25519 p2p|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[23](./ACPs/23-p-chain-native-transfers/README.md)|P-Chain Native Transfers|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[24](./ACPs/24-shanghai-eips/README.md)|Activate Shanghai EIPs on C-Chain|Darioush Jalali (https://github.com/darioush)|Standards|
|[25](./ACPs/25-vm-application-errors/README.md)|Virtual Machine Application Errors|Joshua Kim (https://github.com/joshua-kim)|Standards|
|[30](./ACPs/30-avalanche-warp-x-evm/README.md)|Integrate Avalanche Warp Messaging into the EVM|Aaron Buchwald (aaron.buchwald56@gmail.com)|Standards|
|[31](./ACPs/31-enable-subnet-ownership-transfer/README.md)|Enable Subnet Ownership Transfer|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[41](./ACPs/41-remove-pending-stakers/README.md)|Remove Pending Stakers|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[62](./ACPs/62-disable-addvalidatortx-and-adddelegatortx/README.md)|Disable `AddValidatorTx` and `AddDelegatorTx`|Jacob Everly (https://twitter.com/JacobEv3rly), Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[75](./ACPs/75-acceptance-proofs/README.md)|Acceptance Proofs|Joshua Kim (https://github.com/joshua-kim)|Standards|
|[77](./ACPs/77-reinventing-subnets/README.md)|Reinventing Subnets|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[83](./ACPs/83-dynamic-multidimensional-fees/README.md)|Dynamic Multidimensional Fees for P-Chain and X-Chain|Alberto Benegiamo (https://github.com/abi87)|Standards|
|[84](./ACPs/84-table-preamble/README.md)|Table Preamble for ACPs|Gauthier Leonard (https://github.com/Nuttymoon)|Meta|
|[99](./ACPs/99-validatorsetmanager-contract/README.md)|Validator Manager Solidity Standard|Gauthier Leonard (https://github.com/Nuttymoon)|Best Practices|
|[103](./ACPs/103-dynamic-fees/README.md)|Add Dynamic Fees to the X-Chain and P-Chain|Dhruba Basu (https://github.com/dhrubabasu), Alberto Benegiamo (https://github.com/abi87), Stephen Buttolph (https://github.com/StephenButtolph)|Standards|
|[108](./ACPs/108-evm-event-importing/README.md)|EVM Event Importing|Michael Kaplan (https://github.com/mkaplan13)|Best Practices|
|[113](./ACPs/113-provable-randomness/README.md)|Provable Virtual Machine Randomness|Tsachi Herman (http://github.com/tsachiherman)|Standards|
|[118](./ACPs/118-warp-signature-request/README.md)|Standardized P2P Warp Signature Request Interface|Cam Schultz (https://github.com/cam-schultz)|Best Practices|
|[125](./ACPs/125-basefee-reduction/README.md)|Reduce C-Chain minimum base fee from 25 nAVAX to 1 nAVAX|Stephen Buttolph (https://github.com/StephenButtolph), Darioush Jalali (https://github.com/darioush)|Standards|
|[131](./ACPs/131-cancun-eips/README.md)|Activate Cancun EIPs on C-Chain and Subnet-EVM chains|Darioush Jalali (https://github.com/darioush), Ceyhun Onur (https://github.com/ceyonur)|Standards|
|[151](./ACPs/151-use-current-block-pchain-height-as-context/README.md)|Use current block P-Chain height as context for state verification|Ian Suvak (https://github.com/iansuvak)|Standards|
|[176](./ACPs/176-dynamic-evm-gas-limit-and-price-discovery-updates/README.md)|Dynamic EVM Gas Limits and Price Discovery Updates|Stephen Buttolph (https://github.com/StephenButtolph), Michael Kaplan (https://github.com/michaelkaplan13)|Standards|
|[191](./ACPs/191-seamless-l1-creation/README.md)|Seamless L1 Creations (CreateL1Tx)|Martin Eckardt (https://github.com/martineckardt), Aaron Buchwald (https://github.com/aaronbuchwald), Michael Kaplan (https://github.com/michaelkaplan13), Meag FitzGerald (https://github.com/meaghanfitzgerald)|Standards|

## Contributing

Before contributing to ACPs, please read the [ACP Terms of Contribution](./CONTRIBUTING.md).
