# What is an Avalanche Community Proposal?

[<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 496 512" width="10" height="10"><path d="M165.9 397.4c0 2-2.3 3.6-5.2 3.6-3.3 .3-5.6-1.3-5.6-3.6 0-2 2.3-3.6 5.2-3.6 3-.3 5.6 1.3 5.6 3.6zm-31.1-4.5c-.7 2 1.3 4.3 4.3 4.9 2.6 1 5.6 0 6.2-2s-1.3-4.3-4.3-5.2c-2.6-.7-5.5 .3-6.2 2.3zm44.2-1.7c-2.9 .7-4.9 2.6-4.6 4.9 .3 2 2.9 3.3 5.9 2.6 2.9-.7 4.9-2.6 4.6-4.6-.3-1.9-3-3.2-5.9-2.9zM244.8 8C106.1 8 0 113.3 0 252c0 110.9 69.8 205.8 169.5 239.2 12.8 2.3 17.3-5.6 17.3-12.1 0-6.2-.3-40.4-.3-61.4 0 0-70 15-84.7-29.8 0 0-11.4-29.1-27.8-36.6 0 0-22.9-15.7 1.6-15.4 0 0 24.9 2 38.6 25.8 21.9 38.6 58.6 27.5 72.9 20.9 2.3-16 8.8-27.1 16-33.7-55.9-6.2-112.3-14.3-112.3-110.5 0-27.5 7.6-41.3 23.6-58.9-2.6-6.5-11.1-33.3 2.6-67.9 20.9-6.5 69 27 69 27 20-5.6 41.5-8.5 62.8-8.5s42.8 2.9 62.8 8.5c0 0 48.1-33.6 69-27 13.7 34.7 5.2 61.4 2.6 67.9 16 17.7 25.8 31.5 25.8 58.9 0 96.5-58.9 104.2-114.8 110.5 9.2 7.9 17 22.9 17 46.4 0 33.7-.3 75.4-.3 83.6 0 6.5 4.6 14.4 17.3 12.1C428.2 457.8 496 362.9 496 252 496 113.3 383.5 8 244.8 8zM97.2 352.9c-1.3 1-1 3.3 .7 5.2 1.6 1.6 3.9 2.3 5.2 1 1.3-1 1-3.3-.7-5.2-1.6-1.6-3.9-2.3-5.2-1zm-10.8-8.1c-.7 1.3 .3 2.9 2.3 3.9 1.6 1 3.6 .7 4.3-.7 .7-1.3-.3-2.9-2.3-3.9-2-.6-3.6-.3-4.3 .7zm32.4 35.6c-1.6 1.3-1 4.3 1.3 6.2 2.3 2.3 5.2 2.6 6.5 1 1.3-1.3 .7-4.3-1.3-6.2-2.2-2.3-5.2-2.6-6.5-1zm-11.4-14.7c-1.6 1-1.6 3.6 0 5.9 1.6 2.3 4.3 3.3 5.6 2.3 1.6-1.3 1.6-3.9 0-6.2-1.4-2.3-4-3.3-5.6-2z"/></svg>](https://github.com/avalanche-foundation/ACPs)

An Avalanche Community Proposal is a concise document that introduces a change or best practice for adoption on the [Avalanche Network](https://www.avax.com). ACPs should provide clear technical specifications of any proposals and a compelling rationale for their adoption.

ACPs are an open framework for proposing improvements and gathering consensus around changes to the Avalanche Network. ACPs can be proposed by anyone and will be merged into this repository as long as they are well-formatted and coherent. The Avalanche Foundation may, from time to time, recommend specific ACPs that it believes benefit the Avalanche Network/Community but it is ultimately up to members of the Avalanche Network/Community to adopt ACPs they support by running a compatible Avalanche Network Client (ANC), such as [AvalancheGo](https://github.com/ava-labs/avalanchego). The Avalanche Foundation's recommendation is not binding and is made without representations, warranties or guarantees of any kind.

## ACP Tracks

There are three kinds of ACP:

* A `Standards Track` ACP describes a change to the design or function of the Avalanche Network, such as a change to the P2P networking protocol, P-Chain design, Subnet architecture, or any change/addition that affects the interoperability of Avalanche Network Clients (ANCs).
* A `Best Practices Track` ACP describes a design pattern or common interface that should be used across the Avalanche Network to make it easier to integrate with Avalanche or for Subnets to interoperate with each other. This would include things like proposing a smart contract interface, not proposing a change to how smart contracts are executed.
* A `Meta Track` ACP describes a change to the ACP process or suggests a new way for the Avalanche Community to collaborate.

## ACP Statuses

There are four statuses of an ACP:

* A `Proposed` ACP has been merged into the main branch of the ACP repository. It is actively being discussed by the Avalanche Community and may be modified based on feedback.
* An `Implementable` ACP is considered "ready for implementation" by the author(s) and will no longer change meaningfully from its current form (which would require a new ACP). ACPs that are `Implementable` may be `Recommended` by the Avalanche Foundation, if the Avalanche Foundation believes that the `Implementable` ACP benefits the Avalanche Network/Community. Such recommendation does not create any obligation on the part of any individual in the Avalanche Community or the Avalanche Foundation.
* A `Recommended` ACP means that it has been recommended by the Avalanche Foundation; it being understood that neither a recommendation nor a lack of a recommendation creates any obligation or liability on any individual or the Avalanche Foundation. A recommendation by the Avalanche Foundation is merely the opinion of the Avalanche Foundation and is made without any representations, warranties or guarantees of any kind.
* A `Stale` ACP has been abandoned by its author(s) because it is not supported by the Avalanche Community or has been replaced with another ACP.

## ACP Workflow

### Step 0: Think of a Novel Improvement to Avalanche

The ACP process begins with a new idea for Avalanche. Each potential ACP must have an author(s): someone who writes the ACP using the style and format described below, shepherds the associated GitHub Discussion, and attempts to build consensus around the idea. Note that ideas and any resulting ACP is public. Authors should not post any ideas or anything in an ACP that the Author wants to keep confidential or to keep ownership rights in (such as intellectual property rights).

### Step 1: Post Your Idea to [GitHub Discussions](https://github.com/avalanche-foundation/ACPs/discussions/categories/ideas)

The author(s) should first attempt to ascertain whether there is support for their idea by posting in the "Ideas" category of GitHub Discussions. Vetting an idea publicly before going as far as writing an ACP is meant to save both the potential author(s) and the wider Avalanche Community time. Asking the Avalanche Community first if an idea is original helps prevent too much time being spent on something that is guaranteed to be rejected based on prior discussions (searching the Internet does not always do the trick). It also helps to make sure the idea is applicable to the entire community and not just the author(s). Small enhancements or patches often don't need standardization between multiple projects; these don't need an ACP and should be injected into the relevant development workflow with a patch submission to the applicable ANC issue tracker.

### Step 2: Propose an ACP via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

Once the author(s) feels confident that an idea has a decent chance of acceptance, an ACP should be drafted and submitted as a pull request (PR). This draft must be written in ACP style as described below. It is highly recommended that a single ACP contain a single key proposal or new idea. The more focused the ACP, the more successful it tends to be. If in doubt, split your ACP into several well-focused ones. The PR number of the ACP will become its assigned number.

### Step 3: Build Consensus on [GitHub Discussions](https://github.com/avalanche-foundation/ACPs/discussions/categories/discussion) and Provide an Implementation (if Applicable)

ACPs will be merged by ACP maintainers if the proposal is generally well-formatted and coherent. ACP editors will attempt to merge anything worthy of discussion, regardless of feasibility or complexity, that is not a duplicate or incomplete. After an ACP is merged, an official GitHub Discussion will be opened for the ACP and linked to the proposal for community discussion. It is recommended for author(s) or supportive Avalanche Community members to post an accompanying non-technical overview of their ACP for general consumption in this GitHub Discussion. The ACP should be reviewed and broadly supported before a reference implementation is started, again to avoid wasting the author(s) and the Avalanche Community's time, unless a reference implementation will aid people in studying the ACP. At any point in time, Avalanche Community members can sign a message indicating their support/objection to the ACP in the "Straw Poll".

### Step 4: Mark ACP as `Implementable` via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

Once an ACP is considered complete by the author(s), it should be marked as `Implementable`. At this point, all open questions should be addressed and an associated reference implementation should be provided (if applicable). As mentioned earlier, the Avalanche Foundation meets periodically to recommend the ratification of specific ACPs but it is ultimately up to members of the Avalanche Network/Community to adopt ACPs they support by running a compatible Avalanche Network Client (ANC), such as [AvalancheGo](https://github.com/ava-labs/avalanchego).

### [Optional] Step 5: Mark ACP as `Stale` via [Pull Request](https://github.com/avalanche-foundation/ACPs/pulls)

An ACP can be superseded by a different ACP, rendering the original obsolete. If this occurs, the original ACP will be marked as `Stale`. ACPs may also be marked as `Stale` if the author(s) abandon work on it for a prolonged period of time (12+ months). ACPs may be reopened and moved back to `Proposed` if the author(s) restart work.

## What belongs in a successful ACP?

Each ACP must have the following parts:

* `Preamble`: RFC 822 style headers containing metadata about the ACP, including the ACP number, a short descriptive title, the author(s), and optionally the contact info for each author, etc.
* `Abstract`: Concise (~200 word) description of the ACP
* `Motivation`: Rationale for adopting the ACP and the specific issue/challenge/opportunity it addresses
* `Specification`: Complete description of the semantics of any change should allow any ANC/Avalanche Community member to implement the ACP
* `Security Considerations`: Security implications of the proposed ACP

Each ACP can have the following parts:

* `Open Questions`: Questions that should be resolved before implementation
* `Straw Poll`: Collection of advocates/objectors of an ACP

Each `Standards Track` ACP must have the following parts:

* `Backwards Compatibility`: List of backwards incompatible changes required to implement the ACP and their impact on the Avalanche Community
* `Reference Implementation`: Code, documentation, and telemetry (from a local network) of the ACP change

Each `Best Practices Track` ACP can have the following parts:

* `Backwards Compatibility`: List of backwards incompatible changes required to implement the ACP and their impact on the Avalanche Community
* `Reference Implementation`: Code, documentation, and telemetry (from a local network) of the ACP change

### ACP "Straw Poll"

Anyone can open a PR against an ACP and mark themselves as a supporter (you want an ACP to be adopted) or as an objector (you want the ACP to be rejected). This PR must include a message + signature indicating ownership of a given amount of $AVAX. This action is non-binding and only meant to gauge support of an ACP in the Avalanche Community.

If you wish to do this, please sign the following payload on the [Avalanche Wallet](https://wallet.avax.network/wallet/advanced):

```text
AVALANCHE_STRAW_POLL|<ACP Number>|<yes/no>|<timestamp (unix time)>|<metadata (optional)>
```

Here is an example:

```text
Address: "P-avax12xnpg0ecyvc70xqqf9ellc7kqlxpvtmt30ezzy"
Message: "AVALANCHE_STRAW_POLL|1|yes|1696288255|x.com/avax/status/<permalink>"
Signature: "34k8j3w1rxL5wtLCi5AsP93CtjwUWoV5wnyiRucRtSDs9CgAUU8bsYNbgtAAuCmHdG375qa7fimhodxJg2rMxmLfPYrD31E"
```

* `AVALANCHE_STRAW_POLL` is included to prevent signing a message that could somehow be replayed on-chain.
* `timestamp` is included so that Avalanche Community members can update their preference if they change their mind as the ACP evolves.
* `metadata` is an optional field that can be used by Avalanche Community members to link their vote to a social identity (where they must also post the same message + signature) for maintainers to verify before merging.

### ACP Formats and Templates

ACPs should be written in [markdown](https://daringfireball.net/projects/markdown/syntax) format. Please see the [ACP template](./ACPs/TEMPLATE.md) for an example of the correct layout.

### Auxiliary Files

ACPs may include auxiliary files such as diagrams. Image files should be included in a subdirectory for that ACP. Auxiliary files must be named `ACP-XXXX-Y.ext`, where "XXXX" is the ACP number, "Y" is a serial number (starting at 1), and "ext" is replaced by the actual file extension (e.g. "png").

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
|[13](./ACPs/13-subnet-only-validators.md)|Subnet-Only Validators (SOVs)|Patrick O'Grady (contact@patrickogrady.xyz)|Standards|
|[20](./ACPs/20-ed25519-p2p.md)|Ed25519 p2p|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[23](./ACPs/23-p-chain-native-transfers.md)|P-Chain Native Transfers|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[24](./ACPs/24-shanghai-eips.md)|Activate Shanghai EIPs on C-Chain|Darioush Jalali (https://github.com/darioush)|Standards|
|[25](./ACPs/25-vm-application-errors.md)|Virtual Machine Application Errors|Joshua Kim (https://github.com/joshua-kim)|Standards|
|[30](./ACPs/30-avalanche-warp-x-evm.md)|Integrate Avalanche Warp Messaging into the EVM|Aaron Buchwald (aaron.buchwald56@gmail.com)|Standards|
|[31](./ACPs/31-enable-subnet-ownership-transfer.md)|Enable Subnet Ownership Transfer|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[41](./ACPs/41-remove-pending-stakers.md)|Remove Pending Stakers|Dhruba Basu (https://github.com/dhrubabasu)|Standards|
|[62](./ACPs/62-disable-addvalidatortx-and-adddelegatortx.md)|Disable `AddValidatorTx` and `AddDelegatorTx`|Jacob Everly (https://twitter.com/JacobEv3rly), Dhruba Basu (https://github.com/dhrubabasu)|Standards|

## Contributing

Before contributing to ACPs, please read the [ACP Terms of Contribution](./CONTRIBUTING.md).
