| ACP | 256 |
| :- | :- |
| **Title** | Update Hardware Requirements for Primary Network Nodes |
| **Author(s)** | Martin Eckardt ([@martineckardt](https://github.com/martineckardt)), Meaghan FitzGerald ([@meaghanfitzgerald](https://github.com/meaghanfitzgerald)) |
| **Status** | Proposed ([Discussion](https://github.com/avalanche-foundation/ACPs/discussions/257))|
| **Track** | Best Practices |

## Abstract

This ACP updates the recommended minimum hardware for Avalanche Primary Network nodes.

[Call to Action]: Move Primary Network node storage to a physically-mounted (local) NVMe SSD.

_The majority of node operators currently fulfill these requirements. All other operators are encouraged to update by January 17th, 2026._

Avalanche L1 nodes: No change required.

## Motivation

EVM throughput on Avalanche is primarily limited by state access latency. Lower-latency storage enables higher C-Chain throughput (gas/sec). This ACP standardizes Primary Network node storage guidance on local NVMe, consistent with common practice across major L1 blockchains.

## Specification

### Updated Minimum Recommended Hardware for Primary Network nodes

- CPU: Equivalent of 8 AWS vCPU
- RAM: 16 GB
- Storage Size: 1 TB
  - Nodes storing large historal/archival state, or running custom configs may require up to ~15 TB
- Storage Type: Physically-mounted (local) NVMe SSD (new)
- Network: Reliable IPv4 or IPv6 connectivity, with an open public port

### Migration (Recommended Path)

If a node does not meet the storage requirement, the operator should migrate to a freshly state-synced node (zero downtime, can be done during a validation period):
[https://build.avax.network/docs/nodes/node-storage/periodic-state-sync](https://build.avax.network/docs/nodes/node-storage/periodic-state-sync).

Validators that cannot update to low-latency storage can periodically [manually delete state-sync snapshots](https://build.avax.network/docs/nodes/node-storage/state-sync-snapshot-deletion).

## Validator Storage Exception and Required State Management

Validators may use higher-latency storage (e.g., network-attached NVMe / SSD) only if stored state is kept to < 500 GB.

### How to meet the < 500 GB requirement:

- Periodically replace the node with a freshly state-synced node:
[https://build.avax.network/docs/nodes/node-storage/periodic-state-sync](https://build.avax.network/docs/nodes/node-storage/periodic-state-sync)

- Or manually delete state sync snapshots:
[https://build.avax.network/docs/nodes/node-storage/state-sync-snapshot-deletion](https://build.avax.network/docs/nodes/node-storage/state-sync-snapshot-deletion)

Non-validator nodes (RPC, archival, history-heavy): Use local NVMe. If you need historical access, state management alone is not a substitute for low-latency disks.

### Cloud Service Provider Guidance

- AWS: Instance Store NVMe (i3 / i4i / i4g) recommended; EBS gp3 acceptable only for validators with state management
- Azure: Lsv3-series (local NVMe) recommended; Premium SSD Managed Disks (P30+) acceptable for validators only with state management
- Google Cloud: Local SSD (NVMe) recommended; Persistent SSD acceptable for validators only with state management

Note: These offerings are subject to change at the discretion of the CSP. It is imperative that all operators independently confirm that their cloud instance offering is, in fact, locally-mounted NVMe.

## Background

State access latency is driven by:

1. Disk latency: Network-attached disks add latency from network/protocol overhead.
2. Stored state size: Larger state increases storage pressure; needs vary by node type.

Reducing either one enables the node to process higher throughput. The second option is not available to nodes that require historical state.

## Cost Considerations

- Validators: Network-attached storage can be a cost compromise only with disciplined state management.
- Archival / history-heavy: Treat local NVMe as mandatory for functional performance.

## Backwards Compatibility

This ACP introduces no protocol changes. All recommendations are compatible with current and historical versions of AvalancheGo. Existing configurations will continue to function, but higher-latency storage increases the risk of falling behind during load spikes.

## Security Considerations

1. Under-resourced validators may become unresponsive during stress, reducing effective participation.
2. State management procedures require operational care (and sometimes downtime).
3. Cloud storage failure modes exist for both ephemeral local disks and network-attached volumes—use monitoring, protections, and recovery plans.
