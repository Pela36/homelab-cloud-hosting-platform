# Case Study: Solving the Storage Redundancy Challenge

## The Problem

When designing a high-availability hosting platform, the obvious first step is to create a Proxmox cluster with multiple nodes. This allows containers to migrate between nodes if one fails.

However, there's a critical flaw in basic HA setups: **the data**.

### Initial Architecture (Flawed)

```
Node 1 (Proxmox)
├── Container A (running)
│   └── Disk: local-lvm storage on Node 1
└── Container B (running)
    └── Disk: local-lvm storage on Node 1

Node 2 (Proxmox)
└── [Empty, ready for failover]
```

**Scenario: Node 1 fails**

1. Proxmox detects Node 1 is down (1-2 minutes)
2. HA manager decides to migrate Container A to Node 2
3. Container A configuration migrates instantly
4. But Container A's disk is still on Node 1 (which is offline)
5. System must transfer 20GB disk over network
6. Transfer time: 10-30 minutes depending on network speed
7. Container finally starts on Node 2

**Result**: 15-35 minutes of downtime

For a hosting platform, this is **unacceptable**. Customers expect near-instant failover, not half-hour outages.

## Understanding the Bottleneck

The fundamental issue:
- **Compute is portable** (container config is tiny, migrates instantly)
- **Storage is not portable** (disk data is large, stuck on failed node)

Traditional HA solutions:
1. **SAN (Storage Area Network)**: Expensive enterprise hardware ($10k+)
2. **NFS**: Network filesystem - works but has performance issues and single point of failure
3. **Ceph**: Distributed storage - complex, requires minimum 3 nodes, high overhead

None of these fit the requirements:
- Cost-effective (homelab budget)
- High performance (no NFS latency)
- 2-node setup (not 3+ nodes)
- Simple to maintain

## Research Phase

Spent ~2 weeks researching distributed storage solutions:

**Evaluated:**
- Ceph: Too complex, requires 3 nodes minimum
- GlusterFS: Performance issues, complexity
- NFS: Single point of failure, latency
- DRBD: Promising, but manual setup is complex
- Linstor: DRBD with automation layer

**Key insight**: DRBD (Distributed Replicated Block Device) is the Linux kernel's solution for real-time replication. Linstor is a management layer on top of DRBD that makes it practical.

## Solution: Linstor

### What is Linstor?

Linstor creates a distributed storage pool that:
- Replicates every write to multiple nodes simultaneously
- Presents as local block device (no network overhead for reads)
- Integrates with Proxmox natively
- Handles failures automatically

### How It Solves the Problem

**New Architecture:**

```
Node 1                              Node 2
├── 2TB SSD (Linstor)              ├── 2TB SSD (Linstor)
│   └── Mirror of all data         │   └── Mirror of all data
│                                  │
├── Container A                    ├── [Ready to run Container A]
│   └── Disk on Linstor            │   └── Disk already present
└── Container B                    └── [Ready to run Container B]
    └── Disk on Linstor                └── Disk already present
```

**Write operation:**
```
Container writes file
    ↓
Linstor receives write
    ↓
Simultaneously writes to both Node 1 and Node 2 SSDs
    ↓
Write confirmed when both nodes acknowledge
```

**Read operation:**
```
Container reads file
    ↓
Read from local disk (Node 1)
    ↓
No network involved, full SSD speed
```

**Failover scenario (Node 1 fails):**
```
1. Node 1 fails
2. Proxmox detects failure (1-2 minutes)
3. HA manager migrates Container A to Node 2
4. Container starts on Node 2
5. Disk already exists locally on Node 2 (Linstor replica)
6. Container running in 5-10 seconds
```

**Result**: 2 minutes of downtime (mostly detection time), not 30 minutes

## Implementation

### Hardware Setup

Added to each node:
- 2x 2TB Samsung 870 EVO SSD
- SATA connections
- No RAID controller needed (Linstor handles redundancy in software)

### Linstor Installation

```bash
# On both nodes
apt update
apt install -y linstor-controller linstor-satellite linstor-client drbd-utils

# Start services
systemctl enable --now linstor-controller
systemctl enable --now linstor-satellite

# Create storage pool
linstor physical-storage create-device-pool \
    --pool-name linstor_pool \
    LVM pve1 /dev/sdb

linstor physical-storage create-device-pool \
    --pool-name linstor_pool \
    LVM pve2 /dev/sdb

# Create resource group with 2-way replication
linstor resource-group create \
    --place-count 2 \
    --storage-pool linstor_pool \
    customer_vms

# Set as default storage in Proxmox
linstor storage-pool set-property linstor_pool DfltStorPool true
```

### Integration with Proxmox

```bash
# Add Linstor as storage in Proxmox
pvesm add linstor linstor-pool \
    --controller-vm-list pve1,pve2 \
    --content images,rootdir
```

Now Proxmox can create containers directly on Linstor storage through the GUI.

## Testing

### Controlled Failure Test

**Setup:**
- 3 active customer WordPress containers
- All running on Node 1
- Monitoring connected

**Test procedure:**
1. Hard power off Node 1 (simulate complete failure)
2. Monitor HA manager response
3. Time until containers running on Node 2
4. Verify data integrity
5. Verify websites accessible

**Results:**
```
T+0:00 - Node 1 powered off
T+1:23 - HA manager detects failure
T+1:25 - Container migration initiated
T+1:30 - Container A starts on Node 2
T+1:35 - Container B starts on Node 2
T+1:40 - Container C starts on Node 2
T+1:45 - All sites responding normally
```

**Total downtime per site: ~2 minutes**

**Data integrity:**
- MySQL databases: All data present, no corruption
- WordPress files: All present
- Customer uploads: All intact
- No errors in logs

### Performance Impact

**Write performance:**
- Single disk: ~450 MB/s
- Linstor (2-way replication): ~380 MB/s
- **Overhead: ~15%** (acceptable trade-off)

**Read performance:**
- Single disk: ~500 MB/s
- Linstor: ~500 MB/s
- **Overhead: 0%** (reads are local)

**Latency:**
- Single disk: 0.1ms average
- Linstor write: 0.12ms average
- **Added latency: 0.02ms** (negligible)

### Network Load

Replication uses dedicated network link between nodes:
- Average bandwidth during normal operation: 5-10 Mbps
- Peak during high write activity: 100-150 Mbps
- Well within 2.5Gbps link capacity

## What I Learned

### 1. Storage is the Hardest Part of HA

Compute is easy to make redundant. Storage is hard because:
- Data is large (slow to move)
- Data must be consistent
- Performance matters
- Complexity increases failure modes

### 2. Real-Time Replication vs Backup

**Backup** (Proxmox native):
- Periodic copies (daily, weekly)
- Storage-intensive (full copies)
- Recovery: restore from backup (minutes to hours)
- Data loss: everything since last backup

**Real-time replication** (Linstor):
- Every write replicated immediately
- Storage: same as single copy (no snapshots stored)
- Recovery: instant (data already there)
- Data loss: zero (up to last write)

They solve different problems. Need both.

### 3. Network Architecture Matters

Initial attempt used management network for replication:
- Shared with Proxmox management traffic
- Occasional slowdowns during maintenance
- Risk of saturation

Solution: Dedicated replication network
- Separate physical connection
- Isolated from other traffic
- Consistent performance

### 4. Testing Failures is Critical

Theory vs practice:
- **Theory**: "HA should work"
- **Practice**: "Let's actually pull the plug and see"

Found several issues during testing:
- Timeout values too aggressive
- HA group configuration suboptimal
- Monitoring alerts insufficient

Only found these by actually causing failures.

## Alternative Approaches Considered

### Why Not Ceph?

**Pros:**
- Industry standard
- Feature-rich
- Strong consistency guarantees

**Cons:**
- Requires minimum 3 nodes (I have 2)
- Complex setup and maintenance
- Higher overhead (OSD daemons, monitors)
- Overkill for use case

**Verdict**: Good for large clusters, too complex for 2-node homelab

### Why Not ZFS Replication?

**Pros:**
- Native to FreeBSD/Linux
- Excellent for snapshots
- Strong data integrity

**Cons:**
- Replication is periodic, not real-time
- Manual failover required
- No automatic consistency maintenance

**Verdict**: Good for backups, not for HA

### Why Not Simple DRBD?

**Pros:**
- Exactly what's needed (real-time replication)
- Kernel-level (fast)
- Battle-tested

**Cons:**
- Manual configuration is complex
- No Proxmox integration
- Volume management is tedious

**Verdict**: Perfect technology, but Linstor adds essential automation

## Cost-Benefit Analysis

**Investment:**
- 4x 2TB SSD: ~$600
- Time to research: ~20 hours
- Time to implement: ~10 hours
- Time to test: ~5 hours

**Benefit:**
- Downtime reduced from 30+ minutes to 2 minutes
- Zero data loss during failures
- Customer confidence
- Professional-grade platform

**ROI**: Worth it. The alternative is customers leaving after a 30-minute outage.

## Cloud Comparison

This problem-solving process parallels cloud decisions:

| Question | Homelab Answer | Cloud Answer |
|----------|---------------|--------------|
| Need redundancy? | Add second node | Use multiple availability zones |
| Storage bottleneck? | Add Linstor | Use EBS multi-attach or EFS |
| Testing failures? | Physically pull power | Chaos engineering / fault injection |
| Cost vs benefit? | $600 for enterprise-grade HA | Pay for multi-AZ deployment |

The **thought process** is identical, just different tools.

## Key Takeaways

1. **Identified the real problem** - Not "need HA", but "need fast failover"
2. **Researched solutions** - Evaluated multiple technologies
3. **Chose appropriate tool** - Linstor fits requirements perfectly
4. **Tested thoroughly** - Verified solution actually works
5. **Measured impact** - Quantified performance trade-offs

This demonstrates:
- Distributed systems thinking
- Understanding of storage architecture
- Problem-solving methodology
- Testing and validation
- Production-grade thinking

---

*The best infrastructure is invisible. Customers shouldn't know there was a failure. With Linstor, they don't.*