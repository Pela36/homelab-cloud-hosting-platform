# High Availability Design

## Problem Statement

Running production workloads on a single server creates a critical single point of failure. If the node fails, all customer websites go offline and data stored on local disks becomes inaccessible until the node recovers. This is unacceptable for any hosting platform.

## Solution Architecture

### Two-Node Proxmox Cluster

Implemented a 2-node Proxmox cluster that provides:
- Automatic failure detection
- Seamless workload migration
- Zero data loss during failures
- Minimal downtime (seconds, not minutes)

### The Volume Transfer Problem

Initial HA implementation revealed a critical bottleneck: when a node fails, Proxmox can migrate VMs/containers to the surviving node, but their **disk volumes remain on the failed node**. This means:

1. Container migrates to Node 2
2. Container's disk is still on Node 1 (failed)
3. Migration must wait for volume transfer over network
4. Transfer time: minutes to hours depending on disk size

**This defeats the purpose of HA.**

## Linstor: The Game Changer

### Why Linstor?

Linstor is a distributed storage solution that creates a unified storage pool across multiple nodes with real-time replication. Key benefits:

- **Distributed Block Storage**: Storage spans both nodes
- **Real-Time Mirroring**: Every write goes to both nodes simultaneously
- **No Transfer Needed**: Data already exists on both nodes
- **Fast Failover**: Container starts immediately using local replica

### Implementation

**Hardware:**
- 2x 2TB SSD per node
- Dedicated network connection for storage replication

**Configuration:**
- DRBD (Distributed Replicated Block Device) backend
- 2-way replication (both nodes have full copy)
- Resource group: linstor_vg
- Automatic consistency checks

### How It Works

```
Write Operation:
User updates WordPress → LXC writes to disk → 
Linstor replicates to both nodes simultaneously

Failover Scenario:
Node 1 fails → Proxmox detects failure (1-2 min) → 
Container migrates to Node 2 → 
Disk already exists on Node 2 → 
Container starts immediately (seconds)
```

## Testing Results

### Controlled Failure Test

**Setup:**
- 5 active customer containers
- Simulated hard power-off of Node 1

**Results:**
- Failure detection: 1-2 minutes
- Migration initiation: Automatic
- Container startup on Node 2: 5-10 seconds per container
- Data integrity: 100% (zero data loss)
- Total customer-facing downtime: ~2 minutes

### Why So Fast?

Traditional HA (without Linstor):
```
Failure → Detection (1-2 min) → 
Volume transfer (10-30 min depending on size) → 
Container start (30 sec)
Total: 11-32 minutes
```

Linstor-based HA:
```
Failure → Detection (1-2 min) → 
Container start (5-10 sec)
Total: ~2 minutes
```

The volume transfer step is **completely eliminated** because the data is already present on the surviving node.

## Additional Benefits

### Resource Limits with Failsafe

Each LXC container has:
- CPU limits (prevents resource starvation)
- Memory limits (prevents OOM on host)
- Disk I/O limits (prevents one container affecting others)
- **Auto-restart on overload**: If container hits limits and crashes, it automatically restarts

### Snapshots

Linstor supports instant snapshots:
- Initial clean snapshot after provisioning
- Customer-initiated rollback capability
- Zero additional disk space (copy-on-write)
- Restoration in seconds

## Storage Redundancy

### Disk Failure Handling

**Scenario**: One disk fails on Node 1

**Resolution**:
1. Linstor detects failed disk
2. Marks volume as degraded (but still operational)
3. All operations continue using Node 2's replica
4. Replace failed disk
5. Linstor automatically rebuilds data from Node 2
6. Full redundancy restored

**Customer impact**: None. Service continues normally.

### Split-Brain Prevention

Linstor uses quorum-based decision making:
- 2-node setup with witness (router node acts as tiebreaker)
- Prevents split-brain scenarios
- Network partition handling: majority side stays active

## Performance Considerations

### Replication Overhead

- Write performance: ~5-10% slower than local disk (negligible for web hosting)
- Read performance: Local speed (no network involved)
- Network saturation: Dedicated storage network prevents interference

### Scalability

Current setup handles:
- 50+ containers comfortably
- Can scale to 100+ with current hardware
- Adding Node 3 would provide N+2 redundancy (even better HA)

## Comparison to Cloud Solutions

This homelab HA setup directly parallels cloud infrastructure:

| Homelab | AWS | Azure | Concept |
|---------|-----|-------|---------|
| 2-node Proxmox cluster | Multi-AZ deployment | Availability Sets | Fault domain separation |
| Linstor replication | EBS multi-attach | Azure Shared Disks | Distributed storage |
| HA resource groups | Auto Scaling Groups | VM Scale Sets | Automated failover |
| Container migration | ECS task rescheduling | AKS pod rescheduling | Workload portability |

## Key Takeaways

1. **HA without shared storage is incomplete** - Solved with Linstor
2. **Real-time replication eliminates transfer bottleneck** - Failover in seconds, not minutes
3. **Tested under real failure conditions** - Not theoretical, actually works
4. **Production-grade thinking** - Identified problem before implementation, not after failure

This design demonstrates understanding of:
- Distributed systems
- Storage architecture
- Failure modes and recovery
- Performance vs. reliability tradeoffs
- Infrastructure automation

---

*The difference between 2 minutes and 30 minutes of downtime can mean the difference between annoyed customers and lost customers. This architecture ensures the former.*