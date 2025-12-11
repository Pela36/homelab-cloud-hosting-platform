# Automated Provisioning Workflow

## Overview

The platform achieves zero-touch provisioning through a Python Flask API that orchestrates container creation, network configuration, and WordPress installation. From payment to live website takes 3-5 minutes with no manual intervention.

## Workflow Architecture

### Trigger Event

**Paymenter** (billing system) sends HTTP webhook when payment succeeds. See [Paymenter Integration](../infrastructure/paymenter.md) for complete details on the billing system architecture and extension development.

```
POST /create-site
{
  "service_id": "12345",
  "email": "customer@example.com",
  "plan": "basic",
  "public_ip": "203.0.113.10"
}
```

The `service_id` tracks the service throughout its lifecycle and links all resources to the customer.

## Provisioning Steps

### 1. Resource Allocation

**Subdomain Generation:**
- Extracts username from customer email
- Sanitizes (removes special characters, lowercase)
- Checks for collision and adds random suffix if needed
- Result: `customer.nodepoint.eu`

**IP Assignment:**
- Scans private network (192.168.1.100-253) for next available IP
- Assigns to container for internal communication

**VMID Selection:**
- Queries Proxmox for next available container ID
- Sequential allocation starting from defined range

**Plan Resources:**
- Reads resource allocation from plan configuration
- Basic: 2GB RAM, 20GB disk
- Premium tiers: Higher allocations

### 2. Database Entry

Creates tracking record in SQLite database:

```
service_id: "12345"
email: "customer@example.com"
plan: "basic"
vmid: 907
private_ip: "192.168.1.105"
subdomain: "customer"
public_ip: "203.0.113.10"
created_at: timestamp
```

Additionally initializes domain management row with default quota.

### 3. Container Creation

**Proxmox API Call:**
- Creates unprivileged LXC container
- Storage: DRBD (distributed between node1 and node2)
- Template: Ubuntu 24.04 cloud image
- Network bridge: vmbr1 (private LAN)
- Resource limits: RAM and disk from plan

**Configuration:**
```
node: node2 (home node)
vmid: 907
hostname: customer.nodepoint.eu
memory: 2048 MB
disk: 20 GB (on DRBD storage)
bridge: vmbr1
ip: 192.168.1.105/24
unprivileged: true
```

Container boots automatically after creation.

### 4. High Availability Configuration

**HA Resource:**
- Registers container with HA manager
- Max restarts: 3 (on same node)
- Max relocations: 2 (migrations between nodes)
- State: started (auto-restart enabled)

**HA Rule:**
- Creates rule linking container to node group
- Eligible nodes: node1, node2
- Allows automatic failover on node failure

### 5. DNS Record Creation

**Cloudflare API:**
- Creates A record for subdomain
- Points to: 203.0.113.10 (NPM reverse proxy)
- TTL: 300 seconds (5 minutes)
- Propagation: Nearly instant

Result: `customer.nodepoint.eu` resolves to reverse proxy.

### 6. Reverse Proxy Configuration

**Nginx Proxy Manager API:**
- Creates proxy host entry
- Domain: `customer.nodepoint.eu`
- Forward target: `192.168.1.105:80` (container private IP)
- SSL: Automatic Let's Encrypt certificate
- Features: HTTP/2, caching, WebSocket support

**3-Second Delay:**
Brief wait allows container to fully boot before proxy configuration.

### 7. WordPress Installation

**SSH Command Execution:**
- Connects to Proxmox node via SSH
- Uses `pct exec` to run commands inside container
- Executes bash setup script via stdin

**Wait for Container Ready:**
- Polls container with simple echo command
- Retries every 5 seconds (max 6 attempts)
- Ensures network and services are available

**Installation Script:**
The script performs complete LAMP stack setup:

1. **System Configuration:**
   - Fixes locale settings
   - Updates package repositories
   - Installs required packages

2. **LAMP Stack:**
   - Nginx web server
   - MySQL database
   - PHP-FPM with required extensions

3. **Database Setup:**
   - Creates WordPress database
   - Generates secure random password
   - Creates database user with privileges

4. **WordPress Core:**
   - Downloads latest WordPress release
   - Extracts to `/var/www/html/`
   - Configures `wp-config.php` with database credentials
   - Sets site URL to HTTPS subdomain

5. **Web Server:**
   - Configures Nginx for WordPress
   - Enables PHP-FPM
   - Sets proper file permissions
   - Enables URL rewriting

6. **Security:**
   - Restricts file permissions (755/644)
   - Configures firewall rules
   - Disables root MySQL access

**Verification:**
- Checks for `wp-config.php` existence
- Confirms WordPress is accessible internally

### 8. Initial Snapshot

**Proxmox Snapshot:**
- Creates snapshot named "initial"
- Provides clean rollback point
- Customer can restore if site breaks
- Zero additional disk space (copy-on-write)

### 9. Final Response

API returns success with provisioned details:
```json
{
  "success": true,
  "vmid": 907,
  "private_ip": "192.168.1.105",
  "subdomain": "customer"
}
```

Customer receives access information and can begin WordPress setup at:
`https://customer.nodepoint.eu/wp-admin/install.php`

## Complete Traffic Flow

```
Customer Browser
    ↓ (HTTPS to customer.nodepoint.eu)
Internet
    ↓
Cloudflare DNS → 203.0.113.10
    ↓
Cloud Provider (via WireGuard)
    ↓
OpenWrt Router (DMZ bridge)
    ↓
NPM (203.0.113.10) - Reverse Proxy
    ↓ (proxies to private IP)
WordPress Container (192.168.1.105)
    ↓ (Nginx → PHP-FPM → MySQL)
Response back through same path
```

## Additional Operations

### Suspend Service

**Endpoint:** `POST /suspend`

**Actions:**
1. Lookup service by service_id
2. Find container node
3. Graceful shutdown via Proxmox API
4. Container stopped, data preserved

**Use Case:** Non-payment, policy violation

### Unsuspend Service

**Endpoint:** `POST /unsuspend`

**Actions:**
1. Lookup service by service_id  
2. Find container node
3. Start container via Proxmox API
4. Services resume automatically

**Result:** Site accessible again within seconds.

### Delete Service

**Endpoint:** `POST /delete-site`

**Actions:**
1. Lookup service and locate container
2. Stop container (graceful then forced if needed)
3. Remove HA resource and rules
4. Delete snapshots
5. Destroy container and disk
6. Delete DNS record (Cloudflare)
7. Delete proxy host (NPM)
8. Purge custom domains (if any)
9. Remove database entries

**Cleanup:** Complete removal of all resources.

### Custom Domain Management

**Endpoint:** `POST /customdomain/add`

**Actions:**
1. Verify domain ownership (DNS TXT record)
2. Create ISPConfig DNS zone
3. Add proxy host for custom domain
4. Request SSL certificate
5. Track in domains table

**Quota:** Default 3 domains per service (configurable per plan)

### Status Query

**Endpoint:** `GET /status`

**Returns:**
- Service details (vmid, IPs, subdomain)
- Container status (running/stopped)
- Resource usage (CPU, RAM, disk)
- Uptime
- Custom domains list

## Error Handling

**Rollback Capability:**
- Each step logged to database
- On failure, previous steps are reversed
- Example: Container created but DNS fails → Delete container

**Retry Logic:**
- Container operations: 3 attempts with delays
- API calls: Retry on transient failures
- SSH commands: Verify execution success

**Logging:**
- All operations logged with timestamps
- Errors include full stack traces
- Enables debugging and audit trails

## Performance Characteristics

**Timing Breakdown:**
- Database entry: <1 second
- Container creation: 30-60 seconds
- HA configuration: 5-10 seconds
- DNS record: 1-2 seconds (API call, instant propagation)
- NPM configuration: 5 seconds
- WordPress installation: 60-90 seconds
- Snapshot: 10-20 seconds

**Total:** 3-5 minutes end-to-end

**Bottlenecks:**
- WordPress installation (largest component)
- Container boot time
- SSL certificate generation (done asynchronously)

## Concurrency

**Parallel Processing:**
- Each provisioning request runs independently
- No blocking between customers
- Proxmox handles resource contention

**Limits:**
- Proxmox API: ~10 concurrent requests
- Cloudflare API: 1200 requests/5 minutes
- Practical: 10-20 concurrent provisions

## Security Considerations

**API Authentication:**
- Bearer token required for all endpoints
- Token validation on every request

**Input Validation:**
- Service ID format validation
- Email sanitization for subdomain
- Plan verification against allowed values

**Container Security:**
- Unprivileged containers (root inside ≠ root on host)
- Resource limits prevent abuse
- Network isolation via private IPs
- No direct internet access (proxied)

**Database:**
- SQLite with file permissions
- Parameterized queries prevent SQL injection
- Service ID tracking prevents resource orphaning

## Cloud Equivalents

| Component | AWS | Azure |
|-----------|-----|-------|
| Flask API | Lambda + API Gateway | Azure Functions + API Management |
| Container creation | ECS API | Container Instances API |
| SQLite tracking | DynamoDB | CosmosDB |
| Proxmox API | EC2 API | Virtual Machines API |
| WordPress install | User data script | Cloud-init |
| Snapshots | EBS snapshots | Managed disk snapshots |

## Key Takeaways

1. **Full Automation** - Zero manual steps from payment to live site
2. **Robust Error Handling** - Rollback capability on failures
3. **Audit Trail** - Complete logging for debugging
4. **Scalable Design** - Handles concurrent requests
5. **Security Focus** - Authentication, validation, isolation
6. **Production Ready** - Tested with hundreds of provisions

This workflow demonstrates:
- API-driven infrastructure automation
- Multi-service orchestration
- Event-driven architecture
- Error handling and resilience
- Database-backed state tracking

---

## See Also

- **[Paymenter Integration](../infrastructure/paymenter.md)** - Complete billing system architecture
- **[ProxmoxWP Extension](../../code-samples/extension-proxmoxwp.md)** - PHP extension implementation
- **[Python Backend](python-backend.md)** - API architecture details

---

*The automation layer transforms infrastructure APIs into a seamless customer experience, handling complexity behind a simple webhook interface.*
