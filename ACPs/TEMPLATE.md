```text
ACP: <PR Number>
Title: <ACP title>
Author(s): <a list of the author's name(s) and optionally contact info: FirstName LastName <foo@bar.com>>
Discussions-To: <GitHub Discussion URL>
Status: <Proposed, Implementable, Activated, Stale>
Track: <Standards, Best Practices, Meta, Subnet>
Replaces (*optional): <ACP number>
Superseded-By (*optional): <ACP number>
```

This is the suggested template for new ACPs.

## Abstract

A concise (~200 word) description of the ACP being proposed. This should be a very clear and human-readable version of the specification section. Someone should be able to read only the abstract to get the gist of what this ACP is about.

## Motivation

The motivation is critical for ACPs that want to change the functionality of the Avalanche Network. It should clearly explain why the existing Avalanche Network implementation is inadequate to address the problem that the ACP solves.

## Specification

The technical specification should describe the syntax and semantics of any ACP. The specification should be detailed enough to allow any Avalanche Network Client (ANC) to implement the ACP without consultation from the author(s).

## Backwards Compatibility

All ACPs that introduce backwards incompatibilities must include a section describing these incompatibilities and their severity. The ACP should provide recommendations for dealing with these incompatibilities.

## Reference Implementation

A reference/example implementation that people can use to assist in understanding or implementing this specification. If the implementation is too large to reasonably be included inline, then consider adding it as one or more files in `./ACP-####-implementation/*` or linking to a PR on an external repository.

## Security Considerations

Optional section that discusses the security implications/considerations relevant to the proposed change.

## Open Questions

Optional section that lists any concerns that should be resolved prior to implementation.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
