# Proxmox Cluster Infrastructure

## Cluster Overview

**Cluster Name**: `cluster0`  
**Cluster Version**: 5  
**Total Nodes**: 3  
**Quorum Status**: ✅ Quorate (cluster operational)  
**High Availability**: Enabled with DRBD distributed storage

The Proxmox infrastructure forms the compute and virtualization foundation of the entire platform, providing high-availability virtual machines and containers with distributed storage replication.

## Cluster Topology

### Node Architecture

| Node | IP Address | Role | CPU Cores | RAM | Storage | Status |
|------|-----------|------|-----------|-----|---------|--------|
| **node1** | 192.168.1.10 | Primary Compute | 56 cores | 128 GB | 430 GB | ✅ Online |
| **node2** | 192.168.1.11 | Secondary Compute | 12 cores | 128 GB | 428 GB | ✅ Online |
| **node3** | 192.168.1.5 | Router + Services | 4 cores | 16 GB | 68 GB | ✅ Online |

### Node Details

#### Node1 (Primary Compute - High Performance)
```
Hostname: node1
IP: 192.168.1.10
Uptime: 11 days, 18 hours
CPU: 56 cores @ 0.26% utilization
RAM: 92.8 GB / 128 GB used (73.7%)
Disk: 41 GB / 430 GB used
Role: Primary hosting node for production workloads
```

**Characteristics:**
- Highest CPU core count (56 cores)
- Primary node for compute-intensive workloads
- Hosts customer containers with HA capability
- Production WordPress hosting

#### Node2 (Secondary Compute - Balanced)
```
Hostname: node2
IP: 192.168.1.11
Uptime: 11 days, 18 hours
CPU: 12 cores @ 0.96% utilization
RAM: 33.9 GB / 128 GB used (26.9%)
Disk: 30.9 GB / 428 GB used
Role: Secondary hosting node and infrastructure services
```

**Characteristics:**
- Balanced configuration for general workloads
- ISPConfig full installation (VM 100)
- Lower CPU utilization provides headroom for growth
- HA failover target for node1 workloads

#### Node3 (Router + Lightweight Services)
```
Hostname: node3
IP: 192.168.1.5
Uptime: 35 days, 17 hours
CPU: 4 cores @ 6.09% utilization
RAM: 13.5 GB / 16 GB used (86.4%)
Disk: 19.6 GB / 68 GB used
Role: Network infrastructure and lightweight services
```

**Characteristics:**
- Dedicated router node (OpenWrt LXC - CT 10001)
- Longest uptime (most stable node)
- Runs Paymenter billing (VM 107)
- Nginx reverse proxy (VM 201)
- Lower resource capacity optimized for network functions

## Storage Architecture

### Distributed Storage (DRBD - High Availability)

**Storage Name**: `global-lvm`  
**Type**: DRBD (Distributed Replicated Block Device)  
**Total Capacity**: 1.73 TB  
**Used**: 158 GB (8.9%)  
**Controller**: 192.168.1.10 (node1)  
**Replication**: Real-time synchronous between node1 and node2  
**Purpose**: HA-enabled VMs and containers

**Key Features:**
- **Shared Storage**: Both node1 and node2 have simultaneous access
- **Synchronous Replication**: Every write is replicated to both nodes
- **Automatic Failover**: If one node fails, the other has complete data
- **No Volume Transfer**: Eliminates 30+ minute failover times

**How it Works:**
```
Write Operation:
VM/Container → node1 → DRBD → [replicates] → node2
                     ↓
               Both nodes have data immediately
               
Failover Scenario:
node1 fails → HA manager detects → Starts VM on node2 → Data already present
Result: ~2 minute failover (vs 30+ minutes without DRBD)
```

**Storage Breakdown by Node:**
- **node1**: 1.73 TB total, 158 GB used
- **node2**: 1.73 TB total, 158 GB used (mirrored)

### Local Storage (Per-Node)

#### ZFS Pools (node1 & node2)
```
Storage: local-zfs
Type: ZFS
node1: 389 GB total, 94 MB used
node2: 399 GB total, 1.2 GB used
Content: VM images, container root filesystems
Features: Snapshots, compression, checksums
```

**ZFS Benefits:**
- Fast snapshot creation (instant)
- Copy-on-write for efficiency
- Data integrity verification
- Compression to save space

#### LVM Thin (node3)
```
Storage: local-lvm
Type: LVM Thin Provisioning
Capacity: 141 GB total, 103 GB used (72.9%)
Content: VM images, container root
Node: node3 only
```

#### Local Directories (All Nodes)
```
Path: /var/lib/vz
Content: ISO images, templates, backups, snippets
node1: 389 GB available
node2: 398 GB available  
node3: 68 GB available
```

#### Backup Storage (ZFS-backed)
```
Path: /backup/vzdump
Type: Directory on ZFS
node1: 430 GB
node2: 428 GB
node3: 4.5 TB (primary backup destination)
Content: VM/CT backups and snapshots
```

**Backup Strategy:**
- Node3 has dedicated 4.5 TB backup pool
- Automated Proxmox backup jobs
- Snapshot-based rollback capability

## Virtual Machines

### Production VMs

#### VM 100 - ISPConfig Full Stack
```
Node: node2
vCPU: 12 cores
RAM: 64 GB
Status: Running (8 days uptime)
Purpose: Complete DNS, Email, Web/FTP server
Services:
- Authoritative DNS (ns1/ns2.nodepoint.eu)
- Mail server with DKIM/SPF/DMARC
- Web hosting control panel
- FTP server
Network: 788 MB out, 2.06 GB in
```

**ISPConfig provides:**
- DNS zone management via API
- Email hosting for custom domains
- Multi-IP support (4 public IPs)
- Customer domain management

#### VM 107 - Paymenter (Billing)
```
Node: node3
vCPU: 4 cores
RAM: 8 GB
Status: Running (31 days uptime)
Purpose: Customer billing and provisioning system
Disk I/O: 1.94 GB read, 32.5 GB written
Network: 1.14 GB in, 1.08 GB out
Integration: Triggers Python API for WordPress provisioning
```

**Paymenter handles:**
- Payment processing
- Service provisioning webhooks
- Customer portal
- Subscription management
- Integration with Python backend

For complete details on how Paymenter integrates with the infrastructure, see the [Paymenter Integration documentation](../infrastructure/paymenter.md).

#### VM 201 - Nginx Proxy Manager
```
Node: node3
vCPU: 2 cores
RAM: 8 GB
Status: Running (31 days uptime)
Purpose: Reverse proxy for customer websites
Disk I/O: 1.15 GB read, 53.5 GB written
Network: 5.11 GB in, 3.94 GB out
```

**NPM provides:**
- Reverse proxy for all customer sites
- Automatic SSL certificate management (Let's Encrypt)
- Proxies traffic from DMZ to LAN containers
- API for automated proxy host creation

### Template VMs (Stopped)

#### VM 8000 - Ubuntu 24 Cloud Template
```
Node: node2
Purpose: Cloud-init template for rapid Ubuntu VM deployment
Status: Template (stopped)
Usage: Clone this template for new VMs
```

#### VM 9000 - Ubuntu Template
```
Node: node3
Purpose: Base Ubuntu template for cloning
Status: Template (stopped)
Usage: Standard Ubuntu LXC template base
```

## Containers (LXC)

### CT 10001 - OpenWrt Router ⭐
```
Node: node3
vCPU: 4 cores
RAM: 512 MB (only 10 MB actually used!)
Status: Running (15 days uptime)
Tags: router
Purpose: Virtual router managing all network zones
Uptime: Longest-running service (critical infrastructure)
Network: 130 GB in, 127 GB out (heavy routing traffic)
Storage: 51 MB used

Configuration:
- Handles PPPoE connection (VLAN 1203)
- Manages WireGuard VPN tunnel
- Three network zones (WAN, LAN, DMZ)
- Policy-based routing
- Firewall and NAT
```

**Why LXC for Router:**
- Extremely lightweight (512 MB RAM, uses only 10 MB)
- Native Linux networking performance
- Privileged container for full network control
- Can restart without affecting node

**Critical Functions:**
- All internet traffic flows through this container
- PPPoE authentication to ISP
- WireGuard tunnel termination
- Firewall protecting internal networks
- NAT for LAN traffic

### CT 102 - Website Container
```
Node: node3
vCPU: 4 cores
RAM: 2 GB
Status: Running (14 days uptime)
Purpose: Web hosting container
Storage: 1.65 GB used
Network: 135 MB in, 234 MB out
```

### Customer Containers (WordPress Hosting)

Customer WordPress sites run as unprivileged LXC containers:

**Standard Configuration:**
```
vCPU: 2 cores
RAM: 2 GB
Storage: 20 GB (on DRBD)
Network: Private LAN IP (192.168.1.x)
Proxied through: Nginx Proxy Manager
HA: Optional (enabled for premium customers)
```

**Provisioning Flow:**
1. Paymenter receives payment
2. Python API creates LXC on DRBD storage
3. WordPress auto-installed
4. NPM reverse proxy configured
5. DNS record created (Cloudflare)
6. Customer receives access

## High Availability Configuration

### HA Resources

HA is available but selectively enabled for customers:

**HA Capabilities:**
- Automatic failover detection (~2 minutes)
- Zero data loss (DRBD replication)
- Configurable restart/relocate limits
- API-driven HA enablement

**HA Manager Behavior:**

**Failure Detection:**
- Health checks every 60 seconds
- Node considered failed after 2 missed heartbeats (~2 minutes)

**Automatic Actions:**
1. **Service Crash**: Restart on same node (up to 3 times)
2. **Node Failure**: Migrate to another node (up to 2 times)
3. **Both Failed**: Manual intervention required

**Quorum Requirements:**
- Cluster has 3 nodes
- Quorum requires 2 nodes online
- Currently: 3/3 nodes online ✅

### Why Selective HA?

Customer containers typically don't need HA because:
- Low-traffic WordPress sites (acceptable brief downtime)
- Budget hosting tier
- DRBD replication ensures data safety regardless
- HA can be enabled per-customer on premium plans

**Benefits of DRBD even without HA enabled:**
- Data is always on both nodes
- Manual failover takes minutes (not hours)
- No data loss on node failure
- Can enable HA retroactively if needed

## Software-Defined Networking (SDN)

### SDN Configuration

**Network Name**: `localnetwork`  
**Status**: ✅ OK on all nodes  
**Purpose**: Software-defined network abstraction

The SDN layer provides:
- Consistent network configuration across cluster
- VLAN management
- Network isolation
- API-driven network provisioning

## Cluster Performance Metrics

### CPU Utilization
```
node1: 0.26% (56 cores) - Very low, high capacity available
node2: 0.96% (12 cores) - Low utilization
node3: 6.09% (4 cores) - Moderate (router overhead)
```

**Analysis**: Cluster has significant CPU headroom for expansion.

### Memory Utilization
```
node1: 92.8 GB / 128 GB (73.7%)
node2: 33.9 GB / 128 GB (26.9%) - Low
node3: 13.5 GB / 16 GB (86.4%) - High (limited capacity by design)
```

**Analysis**: 
- node1: Moderate usage, room for more containers
- node2: Plenty of room for more VMs
- node3: Operating near capacity (expected for router node)

### Storage Utilization
```
DRBD (global-lvm): 158 GB / 1.73 TB (8.9%)
local-zfs (node1): 94 MB / 389 GB (0.02%)
local-zfs (node2): 1.2 GB / 399 GB (0.3%)
local-lvm (node3): 103 GB / 141 GB (72.9%)
```

**Analysis**:
- DRBD has massive headroom (1.5+ TB free)
- ZFS pools nearly empty (can host many VMs)
- node3 local storage 73% full (router + services)

### Network Traffic (Since Boot)

**Highest Traffic:**
- CT 10001 (router): 257 GB total (130 GB in, 127 GB out)
- VM 201 (NPM): 9 GB total (proxying customer traffic)

## Cluster Capabilities

### What This Infrastructure Enables

**1. High Availability**
- Automatic VM/CT migration on node failure
- ~2 minute failover time (vs 30+ without DRBD)
- Zero data loss with synchronous replication

**2. Live Migration**
- Move running VMs between nodes without downtime
- Load balancing during maintenance
- Resource optimization

**3. Distributed Storage**
- DRBD provides shared storage without SAN
- Both nodes have full copy of data
- No single point of failure

**4. Snapshot & Backup**
- ZFS instant snapshots
- Automated backup to node3 (4.5 TB pool)
- Point-in-time recovery for customer data

**5. Template-Based Deployment**
- Clone VMs from templates in seconds
- Consistent configuration
- Rapid scaling

**6. API-Driven Automation**
- Full REST API for provisioning
- Python integration for customer automation
- Infrastructure as Code

## Comparison to Cloud Platforms

| Proxmox Feature | AWS Equivalent | Azure Equivalent |
|-----------------|----------------|------------------|
| 3-node cluster | Multi-AZ deployment | Availability Zones |
| DRBD storage | EBS multi-attach | Azure Shared Disks |
| HA Manager | Auto Scaling Groups | Availability Sets |
| LXC containers | ECS with EC2 launch type | Azure Container Instances |
| QEMU VMs | EC2 instances | Azure Virtual Machines |
| Templates | AMIs | VM Images |
| SDN (localnetwork) | VPC | Virtual Network |
| Proxmox API | EC2 API / CloudFormation | Azure Resource Manager |
| Live migration | EC2 live migration | Azure VM live migration |
| Snapshots | EBS snapshots | Managed Disks snapshots |

## Operational Insights

### Uptime Analysis

**Most Stable:**
- node3: 35 days (router - critical path, rarely restarted)
- VM 107: 31 days (Paymenter billing)
- VM 201: 31 days (Nginx proxy)

**Recently Restarted:**
- node1 & node2: 11 days (likely cluster maintenance)
- VM 100: 8 days (ISPConfig updates)

### Resource Hotspots

**RAM Intensive:**
- VM 100 (ISPConfig): 64 GB (handles all DNS/email)

**Disk I/O Heavy:**
- VM 100 (ISPConfig): 55.8 GB written (logs, email storage)
- VM 201 (NPM): 53.5 GB written (access logs, SSL certs)

**Network Heavy:**
- CT 10001 (router): 257 GB total (all traffic flows through)
- VM 201 (NPM): 9 GB total (customer web traffic)

### Capacity Planning

**Current Utilization:**
- CPU: <10% average across cluster
- RAM: 45% average (plenty of headroom)
- Storage: <10% on DRBD (massive headroom)

**Growth Capacity:**
- Can add 50+ more customer containers easily
- DRBD can handle 100+ WordPress sites (20GB each)
- CPU bottleneck would be node2 (only 12 cores)
- Storage bottleneck is years away at current growth

**Scaling Strategy:**
- Current capacity: ~100 customer sites
- With optimization: 200+ sites possible
- Hardware upgrade path: Add more RAM to node1/node2
- Storage expansion: Add nodes or upgrade DRBD disks

## Management Access

### Web UI
```
URL: https://192.168.1.10:8006
     https://192.168.1.11:8006
     https://192.168.1.5:8006
Authentication: PAM (Linux users) or PVE (Proxmox users)
```

### API Access
```
Endpoint: https://192.168.1.10:8006/api2/json
Authentication: API tokens (used for automation)
Documentation: https://pve.proxmox.com/pve-docs/api-viewer/
Integration: Python backend uses API for provisioning
```

### CLI Access
```
SSH: ssh root@192.168.1.10
Commands:
  pvesh - API shell
  pveversion - Cluster version
  pvecm status - Cluster status
  ha-manager status - HA status
```

## Security Considerations

**Cluster Communication:**
- Corosync encryption between nodes
- SSH for inter-node communication
- API tokens for automation (no passwords exposed)

**VM/Container Isolation:**
- Unprivileged containers where possible
- QEMU/KVM hardware virtualization
- Resource limits enforced (CPU, RAM, I/O)
- Network isolation via bridges

**Customer Security:**
- Each container isolated from others
- Private LAN IPs (not directly internet-accessible)
- Proxied through NPM (single ingress point)
- Resource limits prevent noisy neighbors

**Storage Security:**
- DRBD replication (encryption optional, not enabled)
- ZFS encryption available (not enabled)
- Backup encryption recommended for offsite

## Automation Integration

### Python API Backend

The Python backend integrates with Proxmox API for:

**Container Lifecycle:**
```python
# Create container
proxmox.nodes(node).lxc.create(
    vmid=vmid,
    ostemplate='local:vztmpl/ubuntu-24.04.tar.gz',
    storage='global-lvm',  # DRBD storage
    memory=2048,
    cores=2,
    rootfs='global-lvm:20'
)

# Configure HA (optional)
proxmox.cluster.ha.resources.create(
    sid=f'ct:{vmid}',
    state='started',
    max_restart=3,
    max_relocate=2
)
```

**Operations Automated:**
- Container creation on DRBD storage
- Snapshot management
- HA configuration
- Resource monitoring
- Backup scheduling

## Future Enhancements

**Potential Additions:**
1. **Monitoring Integration**: Zabbix for proactive alerting
2. **Automated Scaling**: Add HA based on customer tier
3. **GPU Passthrough**: For future AI/rendering services
4. **Ceph Migration**: Upgrade from DRBD to Ceph for 3-way replication
5. **Backup Encryption**: Encrypt backups before offsite transfer
6. **Additional Nodes**: Expand to 4-5 nodes for more redundancy

## Key Takeaways

1. **Production-Ready Cluster** - 3 nodes, quorum-based, HA-capable
2. **Efficient Resource Usage** - <10% CPU, 45% RAM, 9% storage utilized
3. **Real HA Implementation** - DRBD eliminates failover bottleneck
4. **Critical Infrastructure** - Router, billing, DNS, proxy all hosted
5. **Operational Maturity** - 35-day uptimes, proven stability
6. **Automation-Ready** - Full API integration for customer provisioning
7. **Scalable Architecture** - Can host 100+ customer sites with current hardware

This infrastructure demonstrates:
- Understanding of clustered systems
- Storage replication concepts (DRBD)
- High availability design
- Resource management and capacity planning
- Production operations mindset
- API-driven automation
- Multi-tenant isolation

---

*This Proxmox cluster serves as the compute foundation for the entire platform, demonstrating enterprise infrastructure patterns that directly translate to AWS EC2/ECS, Azure VMs/AKS, or GCP Compute Engine/GKE environments.*
