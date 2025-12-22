| ACP | 256 |
| :- | :- |
| **Title** | Hardware and Bandwidth Recommendations for Validators and Full Nodes |
| **Author(s)** | Martin Eckardt (https://github.com/martineckardt), Meaghan FitzGerald (https://github.com/meaghanfitzgerald) |
| **Status** | Proposed |
| **Track** | Best Practices |

## Abstract

This ACP specifies hardware and bandwidth recommendations for Avalanche Primary Network participants, distinguishing between Primary Network validators maintaining active state for consensus and full archival nodes serving complete historical data. Physically-mounted NVMe SSDs are recommended for all node types. Validators maintaining state below 500GB through regular management may use network-attached storage as a compromise. Full archival nodes must use physically-mounted NVMe exclusively, as network-attached alternatives fail to meet performance requirements. Bandwidth guidelines are 100 Mbps for low-stake validators and up to 1 Gbps for high-stake validators. These recommendations aim to improve network performance, reduce operator costs, and provide clear guidance for deployment scenarios.

## Motivation

The Avalanche network's growth has created a need for standardized hardware recommendations. Node operators currently face:

1. Unclear guidance on storage architecture choices
2. Confusion about state management practices and infrastructure impact
3. Uncertainty about hardware configurations for different node types
4. Fragmented community advice leading to suboptimal deployments

Without clear recommendations, operators may choose inappropriate infrastructure that degrades network performance or incurs unnecessary costs. This proposal provides evidence-based guidance while maintaining emphasis on network health.

## Specification

### 1. Nomenclature

Primary Network Validator Node: Node participating in Primary Network consensus, maintaining current network state. Validates transactions on X-Chain, P-Chain, and C-Chain, produces or attests to blocks, maintains active UTXO set. Does not require complete historical data beyond consensus requirements.

Low Stake Validator: Primary Network validator with stake below 0.01% of total network stake. Requires 100 Mbps symmetric bandwidth.

High Stake Validator: Primary Network validator with stake at or above 0.01% of total network stake (~21,000 AVAX based on current 210M AVAX staked). Receives prioritized transaction dissemination due to higher block proposal probability, with transaction data volumes driving bandwidth requirements toward 1 Gbps capacity.

Full Archival Node: Node maintaining complete historical blockchain data from genesis. Serves archival API requests for historical state queries and blockchain analysis. Cannot use pruning without losing archival capability.

Physically-Mounted Storage: Storage attached via direct PCIe/NVMe connection (AWS Instance Store, Azure Local NVMe, Google Cloud Local SSD, bare metal NVMe, etc.). Provides lowest latency and highest performance.

Network-Attached Storage: Storage accessed over network (AWS EBS, Azure Managed Disks, Google Cloud Persistent Disks). Introduces network latency to I/O operations.

### 2. Hardware Requirements

#### 2.1 Suggested Storage Architecture

All Avalanche nodes should use physically-mounted NVMe SSDs. This recommendation applies to validators and archival nodes based on superior performance and throughput capabilities. Network-attached storage represents a compromise for validators only, under conditions specified in Section 2.2.

#### 2.2 Primary Network Validator Nodes

CPU: 8 cores minimum

Memory: 16 GiB RAM minimum

Storage Capacity: 1 TB minimum (2 TB recommended)

Storage Type: Physically-mounted NVMe SSD

Network-attached storage may be used if all following conditions are met:

- State maintained below 500 GB through regular pruning or state sync
- Operator implements monitoring and state management procedures
- Operator accepts performance trade-offs

State Management: Validators using network-attached storage must periodically implement either of the following:

- Offline pruning - Reduces database size 30-50%, requires downtime on mainnet
- State sync - Deploy new node with state sync, migrate operations, decommission old node

Operators must implement management before exceeding 500 GB.

#### 2.3 Full Archival Nodes

CPU: 8 cores minimum (16 cores GHz recommended)

Memory: 32 GB RAM minimum (64 GB recommended)

Storage Capacity: 12 TB minimum (15 TB recommended)

Storage Type: Physically-mounted NVMe SSD

Network-attached storage is not supported for archival nodes. Historical queries require consistent low-latency random reads across the entire dataset.

Note for archival nodes run on cloud infra: Physically-mounted NVMe storage is ephemeral. Operators must avoid instance stops, implement backup strategies, and use reserved instances to prevent involuntary termination.

### 3. Cloud Provider Guidance

#### 3.1 Primary Network Validators

AWS: Instance Store (i3, i4i, i4g families) recommended; EBS gp3 acceptable with state management

Azure: Lsv3-series VMs recommended; Premium SSD Managed Disks (P30+) acceptable with state management

Google Cloud: Local SSD (NVMe) recommended; Persistent SSD acceptable with state management

#### 3.2 Archival Nodes

AWS: Instance Store (i3, i3en, i4i, i4g families)

Azure: Lsv3-series VMs

Google Cloud: Local SSD (NVMe)

### 4. Bandwidth Allocations

Primary Network Validator Nodes:

- Low Stake Validators: 100 Mbps symmetric, stable connection
- High Stake Validators: 1 Gbps symmetric, low-latency connection

Note: Higher-stake validators receive proportionally more traffic and must process more data, requiring enhanced network capacity.

Full Archival Nodes:

- Recommended: 1 Gbps symmetric, low-latency connection
- Archival nodes serving API traffic require bandwidth comparable to high-stake validators

### 5. Operating System

Recommended: Ubuntu 22.04/24.04 LTS, macOS >= 12

Not supported for production: Windows, macOS

The following table lists currently supported platforms and their corresponding
AvalancheGo support tiers:

| Architecture | Operating system | Support tier  |
| :----------: | :--------------: | :-----------: |
|    amd64     |      Linux       |       1       |
|    arm64     |      Linux       |       2       |
|    arm64     |      Darwin      |       2       |
|    amd64     |      Darwin      | Not supported |
|    amd64     |     Windows      | Not supported |
|     arm      |      Linux       | Not supported |
|     i386     |      Linux       | Not supported |

To officially support a new platform, one must satisfy the following requirements:

| AvalancheGo continuous integration | Tier 1  | Tier 2  | Tier 3  |
| ---------------------------------- | :-----: | :-----: | :-----: |
| Build passes                       | &check; | &check; | &check; |
| Unit and integration tests pass    | &check; | &check; |         |
| End-to-end and stress tests pass   | &check; |         |         |

### 6. Monitoring Requirements

Operators should monitor:

1. Storage: Disk usage (alert at 80%, critical at 90%), I/O latency (alert if sustained >10ms), IOPS utilization
2. Network: Bandwidth utilization (alert at 80%), packet loss (investigate if >0.1%)
3. Validation: Uptime percentage, validation success rate, block height synchronization

## Rationale

### Storage Performance Requirements

Blockchain operations require high throughput and low latency. Physically-mounted NVMe provide higher throughput with lower latency compared to network-attached storage. This difference directly impacts validator responsiveness, state sync operations, and API response times.

### State Management Threshold

The 500 GB threshold reflects observed performance characteristics of network-attached storage. Above this threshold, increased I/O operations cause performance degradation affecting consensus participation.

### Archival Node Requirements

Archival nodes generate random reads across arbitrary blockchain history. The latency difference between physically-mounted NVMe and network-attached storage compounds with database operations, often causing API timeouts. While network-attached storage can be over-provisioned, the cost exceeds physically-mounted NVMe instances while delivering inferior latency.

### Bandwidth Requirements

Primary Network validators require 100 Mbps for low-stake configurations and up to 1 Gbps for high-stake configurations, as higher-stake validators receive proportionally more consensus traffic.

Archival nodes serving API traffic require 1 Gbps symmetric connections to handle concurrent client requests and historical data queries. The bandwidth requirements account for both consensus participation (for validator-archival nodes) and API traffic serving.

### Cost Considerations

Validators may use network-attached storage with state management as a cost compromise, though this requires operational maturity. Archival nodes require physically-mounted NVMe for functional service; network-attached alternatives fail performance requirements regardless of provisioning.

## Backwards Compatibility

This ACP introduces no protocol changes. All recommendations are compatible with current and historical versions of AvalancheGo. Existing configurations will continue to function, though they may experience suboptimal performance or higher costs.

## Security Considerations

### Performance Impact

Validators with inadequate storage performance may fall behind during high-load periods, reducing effective participation and network security. During stress events, under-resourced validators may become unresponsive, temporarily reducing decentralization.

Validators with insufficient bandwidth may experience delayed block propagation, reducing consensus participation and rewards. This could incentivize centralization to well-connected data centers.

### State Management

Offline pruning requires validator downtime. Operators should coordinate pruning windows to avoid simultaneous pruning, which could reduce network capacity.

State sync relies on validator quorum for state correctness, which is acceptable as state is validated against consensus. Operators concerned about validity can perform full replay from genesis.

### Cloud Infrastructure

Ephemeral storage is lost on instance stop/termination. Operators must implement termination protection, backup procedures, and monitoring for unexpected state changes.

Network-attached storage has documented failure modes. Operators should monitor provider health dashboards and maintain failover procedures.

### Operations

Lack of monitoring can lead to unnoticed degradation. Operators should implement disk space alerts, I/O monitoring, bandwidth tracking, and validation success monitoring. Failure to implement state management before disk exhaustion can cause validator crashes.

## Open Questions

## Acknowledgments

This ACP was inspired by Ethereum's EIP-7870 approach to hardware recommendations. The authors acknowledge the Avalanche validator community for operational feedback and cloud provider documentation for performance specifications.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

---

## Appendix: State Management Procedures

### Offline Pruning

Configuration in ~/.avalanchego/configs/chains/C/config.json:

```json
{
  "offline-pruning-enabled": true,
  "offline-pruning-data-directory": "/path/to/pruning/workspace",
  "offline-pruning-bloom-filter-size": 536870912
}
```

Runs on node startup. Reduces database size 30-50% while experiencing downtime on mainnet. Requires 512 MB additional disk space for bloom filter.

### State Sync

Configuration in ~/.avalanchego/configs/chains/C/config.json:

```json
{
  "state-sync-enabled": true
}
```

Enables nodes to download current state from peers rather than replaying all blocks. Reduces bootstrap time from days to hours. Operators can deploy new state-synced nodes and migrate operations for zero-downtime management.

Reference: https://build.avax.network/docs/nodes/maintain/chain-state-management#managing-disk-usage
