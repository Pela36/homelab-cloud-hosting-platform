# Python Backend API Architecture

## Overview

**Framework:** Flask (Python 3.12)  
**Runtime:** 192.168.1.5:8080 (router node)  
**Purpose:** Orchestration layer connecting Paymenter, Proxmox, Cloudflare, NPM, and ISPConfig

The Python backend serves as the central automation hub, receiving webhooks from the billing system and coordinating infrastructure provisioning across multiple services.

## Architecture Design

### Flask Application Structure

**Entry Point:** `main.py`
- Initializes Flask application
- Binds to router node (10.1.1.5:8080)
- Threaded mode for concurrent requests

**Blueprint-Based Routes:**
The application uses Flask blueprints for modular organization:

- `/create-site` - Provision new WordPress site
- `/delete-site` - Complete service termination
- `/suspend` - Stop container (preserve data)
- `/unsuspend` - Resume suspended service
- `/status` - Query service information
- `/customdomain/add` - Add custom domain
- `/health` - API health check
- `/actions/*` - Customer management operations
- `/upgrades/*` - Plan upgrade handling
- `/support/*` - Support ticket integration

### Core Components

#### Container Manager (`container_manager.py`)

**Purpose:** Proxmox LXC lifecycle management

**Key Methods:**

```python
create_container(vmid, subdomain, private_ip, memory, disk)
# Creates LXC on DRBD storage with specified resources
# Returns: Container creation result

delete_container(vmid, subdomain)
# Stops container, removes HA, deletes snapshots, destroys container
# Returns: Success boolean

add_ha_resource_and_rule(vmid)
# Configures HA resource and migration rules
# Returns: Success boolean
```

**Features:**
- Automatic node selection
- Multi-step container deletion with retries
- Graceful shutdown with forced fallback
- Snapshot cleanup before deletion
- HA resource removal

#### Proxmox Client (`proxmox_client.py`)

**Purpose:** Proxmox API wrapper

**Capabilities:**
- Container creation and management
- Node discovery and status
- HA resource configuration
- Snapshot management
- Container status monitoring

**Authentication:** API token-based (configured via environment)

#### Cloudflare Client (`cloudflare_client.py`)

**Purpose:** DNS record automation

**Operations:**
- Create A records for subdomains
- Delete DNS records
- Update existing records
- Zone management

**API Integration:** Cloudflare REST API with zone-based operations

#### NPM Client (`npm_client.py`)

**Purpose:** Nginx Proxy Manager automation

**Functions:**
- Create proxy hosts
- Configure SSL (Let's Encrypt)
- Enable caching and compression
- Delete proxy configurations
- WebSocket support

**Features:**
- Automatic SSL certificate generation
- HTTP/2 enabled
- Caching for static assets
- Proxy to private IP containers

#### Database (`database.py`)

**Storage:** SQLite3 (file-based)

**Tables:**

**sites:**
```
id, service_id, email, plan, vmid, 
private_ip, subdomain, public_ip, created_at
```

**domains:**
```
id, service_id, domain, domain2, domain3, 
count, backups
```

**Operations:**
- Insert site records
- Query by service_id
- Update plan information
- Track custom domains
- Domain quota management

### Helper Functions

**Subdomain Generation:**
- Extracts username from email
- Sanitizes special characters
- Checks for collisions
- Adds random suffix if needed

**IP Allocation:**
- Scans 192.168.1.100-253 range
- Returns next available IP
- Verifies no conflicts

**VMID Selection:**
- Queries Proxmox for existing containers
- Finds next available ID
- Avoids conflicts with templates

**Plan Resources:**
- Maps plan names to resource allocations
- Returns memory and disk quotas
- Supports multiple tiers

**Authentication:**
- Bearer token validation
- Decorator-based auth (@auth)
- Rejects unauthorized requests

**Node Detection:**
- Finds which node hosts a container
- Queries all nodes in cluster
- Returns node name for operations

## API Endpoints Detail

### POST /create-site

**Flow:**
1. Validate input (service_id, email, plan, public_ip)
2. Generate subdomain and allocate resources
3. Insert database record
4. Create LXC container on DRBD
5. Configure HA resource
6. Create Cloudflare DNS record
7. Configure NPM proxy host
8. Install WordPress via SSH
9. Create initial snapshot
10. Initialize domain quota
11. Return success with details

**Response Time:** 3-5 minutes

**Error Handling:** Rollback on failure, log to database

### POST /delete-site

**Flow:**
1. Lookup service by service_id
2. Find container node
3. Stop container (graceful → forced)
4. Remove HA resource
5. Delete snapshots
6. Destroy container
7. Delete DNS record
8. Delete proxy host
9. Purge custom domains
10. Remove database entries

**Cleanup:** Complete resource removal

### POST /suspend

**Flow:**
1. Lookup service
2. Find container node
3. Graceful shutdown via Proxmox
4. Container stopped, data intact

**Use Case:** Payment issues, policy violations

### POST /unsuspend

**Flow:**
1. Lookup service
2. Find container node
3. Start container
4. Services resume automatically

**Result:** Site online within seconds

### GET /status

**Returns:**
```json
{
  "service_id": "12345",
  "vmid": 907,
  "private_ip": "192.168.1.105",
  "subdomain": "customer",
  "public_ip": "203.0.113.10",
  "status": "running",
  "node": "node1",
  "resources": {
    "cpu": "0.5%",
    "memory": "512MB / 2048MB",
    "uptime": "5 days"
  },
  "domains": ["customer.nodepoint.eu", "example.com"]
}
```

### POST /customdomain/add

**Flow:**
1. Verify domain ownership (TXT record)
2. Check domain quota
3. Create ISPConfig DNS zone
4. Add NPM proxy host
5. Request SSL certificate
6. Update database
7. Decrement domain count

**Quota:** Enforced via domains.count column

## WordPress Installation Process

**Method:** SSH to Proxmox node + `pct exec`

**Script Execution:**
1. Wait for container ready (poll with echo command)
2. Stream bash script via stdin to container
3. Script installs complete LAMP stack
4. Verify installation (check wp-config.php)
5. Create snapshot on success

**Installation Steps:**
- Fix locale settings
- Install Nginx, PHP-FPM, MySQL
- Create database with random password
- Download and extract WordPress
- Configure wp-config.php
- Set file permissions
- Configure Nginx
- Enable services

**Verification:** File existence check before snapshot

## Configuration Management

**Environment Variables (.env):**
```
DB_PATH=./data/sites.db
NODES=node1,node2
HOME_NODE=node2
BRIDGE=vmbr1
TEMPLATE=local:vztmpl/ubuntu-24.04.tar.gz
CLOUDFLARE_API_KEY=<redacted>
NPM_URL=http://npm-url
PROXMOX_HOST=192.168.1.10
PROXMOX_USER=api@pam
PROXMOX_TOKEN=<redacted>
```

**Plan Definitions:**
```python
PLANS = {
    "basic": {"memory": 2048, "disk": 20},
    "standard": {"memory": 4096, "disk": 40},
    "premium": {"memory": 8192, "disk": 80}
}
```

## Error Handling Strategy

**Logging:**
- All operations print status messages
- Errors include full stack traces
- Database tracks success/failure

**Retry Logic:**
- Container operations: 3 attempts
- API calls: Exponential backoff
- SSH commands: Verify execution

**Rollback:**
- Track created resources
- Delete in reverse order on failure
- Prevent orphaned resources

**Graceful Degradation:**
- Core operations must succeed
- Auxiliary features can fail without aborting
- Example: Domain quota init failure doesn't stop provisioning

## Security Implementation

**API Authentication:**
```python
@auth
def protected_endpoint():
    # Token validated before execution
    pass
```

**Input Validation:**
- Service ID format checks
- Email sanitization
- Plan whitelist verification
- SQL injection prevention (parameterized queries)

**Secrets Management:**
- Environment variables for credentials
- Never hardcoded
- .env file gitignored

**Network Security:**
- API binds to router node only
- No direct internet exposure
- Proxied through firewall

## Concurrency Handling

**Threading:**
- Flask runs in threaded mode
- Each request gets dedicated thread
- SQLite handles locking

**Resource Contention:**
- Proxmox API rate limits
- Sequential IP allocation (database lock)
- VMID selection with collision check

**Practical Limits:**
- 10-20 concurrent provisions
- Cloudflare API: 1200 req/5min
- Proxmox: ~10 concurrent operations

## Monitoring and Health

**Health Check Endpoint:**
```
GET /health
Returns: {"status": "ok", "timestamp": "..."}
```

**Logging:**
- Console output for operations
- Database audit trail
- Error tracking with stack traces

**Metrics:**
- Provision success rate
- Average provision time
- API response times
- Error rates by endpoint

## Integration Points

**Paymenter → API:**
- Webhook on payment success
- JSON payload with service details
- See [Paymenter Integration](../infrastructure/paymenter.md) for extension architecture

**API → Proxmox:**
- REST API for container management
- Token-based authentication
- Node discovery and load balancing

**API → Cloudflare:**
- Zone-based DNS management
- A record creation/deletion
- Low-TTL for fast updates

**API → NPM:**
- Proxy host management
- Automatic SSL via Let's Encrypt
- Proxies DMZ to LAN

**API → ISPConfig:**
- Custom domain DNS zones
- Email configuration (future)
- FTP access (optional)

## Comparison to Cloud Patterns

| Backend Component | AWS | Azure |
|-------------------|-----|-------|
| Flask API | Lambda + API Gateway | Azure Functions |
| SQLite database | DynamoDB | CosmosDB |
| Container manager | ECS/EKS API wrapper | AKS API wrapper |
| Orchestration | Step Functions | Logic Apps |
| Secrets | Secrets Manager | Key Vault |
| Logging | CloudWatch | Application Insights |

## Key Design Decisions

**Why Flask:**
- Lightweight and fast
- Easy integration with Linux tools (SSH, subprocess)
- Blueprint architecture for modularity
- Threaded mode for concurrency

**Why SQLite:**
- Simple file-based storage
- No additional service required
- Sufficient for scale (thousands of sites)
- Easy backup (copy file)

**Why SSH for WordPress Install:**
- Proxmox API doesn't support stdin redirection
- pct exec allows running scripts in containers
- More flexible than API-only approach
- Can stream large scripts

**Why Separate Clients:**
- Modular design for maintainability
- Easy to mock for testing
- Reusable across endpoints
- Clear separation of concerns

## Future Enhancements

**Potential Improvements:**
- Async/await for better concurrency (FastAPI)
- Redis for caching and job queues
- Prometheus metrics export
- Structured logging (JSON)
- API versioning
- WebSocket for real-time updates
- Background task queue (Celery)

## Key Takeaways

1. **Orchestration Hub** - Coordinates 5+ services seamlessly
2. **Robust Error Handling** - Retry logic and rollback capability
3. **Modular Architecture** - Blueprint-based, easy to extend
4. **Security First** - Authentication, validation, secrets management
5. **Production Tested** - Handles real customer workloads
6. **Cloud Patterns** - Architecture translates to Lambda/Functions

This backend demonstrates:
- API design and implementation
- Multi-service integration
- Error handling and resilience
- Database-backed state management
- Infrastructure as Code principles
- Production operations mindset

---

## See Also

- **[Paymenter Integration](../infrastructure/paymenter.md)** - Billing system webhook architecture
- **[Automated Workflow](automated_workflow.md)** - Complete provisioning flow
- **[LXC Container Code](../../code-samples/lxc.md)** - Container provisioning implementation

---

*The Python backend is the glue that transforms disparate infrastructure APIs into a cohesive, automated provisioning system.*
