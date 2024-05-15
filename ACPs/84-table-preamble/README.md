```text
ACP: 84
Title: Table Preamble for ACPs
Author(s): Gauthier Leonard <https://github.com/Nuttymoon>
Discussions-To: https://github.com/avalanche-foundation/ACPs/discussions/86
Status: Proposed
Track: Meta
```

## Abstract

The current ACP template features a plain-text code block containing "RFC 822 style headers" as `Preamble` (see [What belongs in a successful ACP?](https://github.com/avalanche-foundation/ACPs?tab=readme-ov-file#what-belongs-in-a-successful-acp)). This header includes multiple links to discussions, authors, and other ACPs.

This ACP proposes to replace the `Preamble` code block with a Markdown table format (similar to what is used in [Ethereum EIPs](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1.md)).

## Motivation

The current ACPs `Preamble` is (i) not very readable and (ii) not user-friendly as links are not clickable. The proposed table format aims to fix these issues.

## Specification

The following Markdown table format is proposed:

| ACP                            | PR Number                                                                                    |
| :----------------------------- | :------------------------------------------------------------------------------------------- |
| **Title**                      | ACP title                                                                                    |
| **Author(s)**                  | A list of the author's name(s) and optionally contact info: FirstName LastName <foo@bar.com> |
| **Discussions-To**             | [Hyperlinked GitHub Discussion name](./README.md)                                            |
| **Status**                     | Proposed, Implementable, Activated, Stale                                                    |
| **Track**                      | Standards, Best Practices, Meta, Subnet                                                      |
| **Replaces (\*optional)**      | [ACP-XX](./README.md)                                                                        |
| **Superseded-By (\*optional)** | [ACP-XX](./README.md)                                                                        |

It features all the existing fields of the current ACP template, and would replace the current `Preamble` code block in [ACPs/TEMPLATE.md](../TEMPLATE.md).

## Backwards Compatibility

Existing ACPs could be updated to use the new table format, but it is not mandatory.

## Reference Implementation

For this ACP, the table would look like this:

| ACP                | 84                                              |
| :----------------- | :---------------------------------------------- |
| **Title**          | Table Preamble for ACPs                         |
| **Author(s)**      | Gauthier Leonard <https://github.com/Nuttymoon> |
| **Discussions-To** | [ACP-84: Table Preable for ACPs](./README.md)   |
| **Status**         | Proposed                                        |
| **Track**          | Meta                                            |

## Security Considerations

NA

## Open Questions

NA

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
