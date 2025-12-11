# Paymenter Integration

## Overview

**Paymenter** is the billing and customer portal that ties the entire platform together. Custom-built server extensions bridge Paymenter with the Python backend API, enabling automated provisioning, lifecycle management, and customer self-service features.

## Architecture

### Integration Flow

```
Customer Payment (Paymenter)
    ↓ (Webhook)
Extension (ProxmoxWP / SSupport / DNS)
    ↓ (HTTP API Call)
Python Backend (192.168.1.5:8080)
    ↓ (Orchestration)
Infrastructure (Proxmox, ISPConfig, NPM, Cloudflare)
```

**Key Principle:** Paymenter never talks directly to infrastructure. Extensions act as adapters that translate Paymenter events into backend API calls.

## Extension System

Paymenter uses a **server extension** architecture for service provisioning. Each extension is a PHP class that implements lifecycle hooks and communicates with the backend API.

### Extension Types

**1. ProxmoxWP Extension**
- **Purpose:** WordPress hosting automation
- **API Endpoint:** `/create-site`, `/delete-site`, `/suspend`, etc.
- **Features:** Container management, custom domains, backups, actions

**2. SSupport Extension**
- **Purpose:** Support subscription management
- **API Endpoint:** `/support/create`, `/support/suspend`, etc.
- **Features:** Support key generation, subscription lifecycle

**3. DNS Extension**
- **Purpose:** Customer DNS zone management
- **API Endpoint:** `/customdomain/*` (ISPConfig integration)
- **Features:** Zone creation, record management, nameserver verification

## ProxmoxWP Extension

### Configuration

**Admin Panel Settings:**
- Backend API Base URL: `http://192.168.1.5:8080`
- API Key: Bearer token for authentication
- Public IP: 203.0.113.10 (reverse proxy IP)

### Lifecycle Hooks

**Service Creation:**
```php
createServer(Service $service)
    → POST /create-site
    {
        "service_id": "12345",
        "email": "customer@example.com",
        "plan": "basic",
        "public_ip": "203.0.113.10"
    }
```

Triggers complete WordPress provisioning (container, DNS, proxy, installation).

**Service Termination:**
```php
terminateServer(Service $service)
    → POST /delete-site
    {"service_id": "12345"}
```

Deletes container, DNS records, proxy configuration, and database entries.

**Service Suspension:**
```php
suspendServer(Service $service)
    → POST /suspend
    {"service_id": "12345"}
```

Stops container, preserves data.

**Service Unsuspension:**
```php
unsuspendServer(Service $service)
    → POST /unsuspend
    {"service_id": "12345"}
```

Starts container, restores service.

**Plan Upgrade:**
```php
upgradeServer(Service $service)
    → POST /wp/upgrade/
    {
        "service_id": "12345",
        "plan": "premium"
    }
```

Resizes container resources (RAM, disk) based on new plan.

### Customer Actions

**Control Buttons:**
- **Start:** Powers on container
- **Stop:** Gracefully shuts down container
- **Restart:** Stop + Start
- **Reset:** Rollback to initial snapshot

**Custom Domains:**
- Add Domain: Verifies DNS, creates proxy, decrements quota
- Delete Domain: Removes proxy, increments quota
- List Domains: Shows active domains and quota

**Backups:**
- Create Backup: Snapshot with BAK{date}-{N} naming
- Delete Backup: Removes snapshot, updates quota
- Rollback: Restore container to snapshot
- List Backups: Shows available snapshots with quota (e.g., "2/3")

**Toggle Proxy:**
- Enable/Disable reverse proxy for subdomain
- Stops traffic without deleting container

### Client Area View

**Overview Tab:**
- Plan details
- Expiration date
- Subdomain (clickable link)
- Container status (Running/Stopped)
- Public IP

**Domains Section:**
- Current custom domains
- Add domain form (with PROXYSRV alias support)
- Delete domain button
- Remaining quota display

**Backups Section:**
- Available backups list
- Create backup button
- Delete backup button (per backup)
- Rollback button (per backup)
- Quota display (used/allowed)

## SSupport Extension

### Configuration

**Admin Panel Settings:**
- Support API Base URL: `http://192.168.1.5:8080`
- API Key: Bearer token

### Lifecycle Hooks

**Create Support Subscription:**
```php
createServer(Service $service)
    → POST /support/create
    {
        "service_id": "12345",
        "email": "customer@example.com",
        "firstname": "John",
        "lastname": "Doe"
    }
    
    ← Returns: {"support_key": "ABC123..."}
```

Generates unique support key and stores in MySQL database.

**Suspend/Unsuspend/Terminate:**
Similar pattern to ProxmoxWP, updates status in support database.

### Client Area View

Displays:
- Support key (for ticket submission)
- Subscription status
- Plan tier
- Customer email

## DNS Extension

### Configuration

**Environment Variable:**
```
DNS_BACKEND_URL=http://192.168.1.5:8080/customdomain
```

### Routes

**Zone Management:**
- `GET /dns` - List all customer zones
- `POST /dns/add-zone` - Create new DNS zone
- `POST /dns/{zone}/delete` - Delete entire zone

**Record Management:**
- `GET /dns/{zone}` - View zone records
- `POST /dns/{zone}/records/add` - Add DNS record
- `POST /dns/{zone}/records/{record}/delete` - Delete record

### Integration with ISPConfig

**Workflow:**
1. Customer submits zone creation request
2. Extension calls backend `/customdomain/add/zone`
3. Backend calls ISPConfig API to create zone
4. Backend creates default NS and A records
5. Zone appears in customer panel

**Supported Record Types:**
- A (IPv4 address)
- AAAA (IPv6 address)
- CNAME (alias)
- MX (mail exchange)
- NS (nameserver)
- TXT (text records)

### PROXYSRV Alias

Special alias for reverse proxy IP:
- Customer enters `PROXYSRV` as record content
- Extension translates to actual proxy IP (203.0.113.10)
- Simplifies customer experience

### Nameserver Verification

Before zone creation:
- Backend verifies domain uses correct nameservers
- Prevents zone creation for improperly configured domains
- Required nameservers: ns1.nodepoint.eu, ns2.nodepoint.eu

## Authentication

### Bearer Token

All extensions use Bearer token authentication:

```php
Http::withHeaders([
    'Authorization' => 'Bearer <api_key>',
    'Accept' => 'application/json'
])
```

Backend validates token on every request via `@auth` decorator.

### Security

**Token Storage:**
- Stored in Paymenter extension configuration
- Never exposed to customers
- Encrypted in database

**Request Validation:**
- Service ID format validation
- Email validation
- Domain ownership verification (DNS)
- Quota enforcement

## Error Handling

### Extension-Level

**HTTP Status Codes:**
- 200: Success
- 400: Bad request (missing parameters)
- 401: Unauthorized (invalid token or suspended service)
- 404: Service not found
- 409: Conflict (duplicate, quota exceeded)
- 500: Server error

**Response Format:**
```json
{
    "success": false,
    "error": "Human-readable error message"
}
```

### User-Friendly Messages

Extensions clean error messages before display:
- Extract JSON error field if present
- Strip HTML tags from error responses
- Remove API prefixes (e.g., "API POST /endpoint failed: HTTP 500")
- Show concise message to customer

### Graceful Degradation

**Non-Critical Failures:**
- If domain list fails to load, show error but don't block page
- If backup creation fails, show specific error to user
- If toggle fails, explain NPM connection issue

## Service ID Tracking

**Consistent Identifier:**
- Paymenter assigns unique `service_id` to each service
- Used throughout backend as primary key
- Tracks: container VMID, IPs, domains, backups, support tickets

**Example:**
```
service_id: "12345"
  ↓
sites table: vmid=907, private_ip=192.168.1.105, subdomain=customer
domains table: domain=example.com, domain2=null, count=2
```

## API Communication

### Request Pattern

**Synchronous Operations:**
Most operations complete within seconds:
- Start/stop: 5-10 seconds
- Domain add: 10-15 seconds (DNS + SSL)
- Backup create: 20-30 seconds

**Asynchronous Operations:**
Provisioning takes 3-5 minutes:
- Extension returns immediately with success
- Backend processes in background
- Customer receives notification when complete

### Timeouts

**HTTP Timeouts:**
- Standard requests: 25 seconds
- Long operations (provisioning): 60 seconds
- DNS verification: 10 seconds

## Plan-Based Features

### Resource Allocation

Plans map to backend resources:

```php
"basic" → 2GB RAM, 20GB disk, 2 backups, 3 domains
"standard" → 4GB RAM, 40GB disk, 3 backups, 5 domains
"premium" → 8GB RAM, 80GB disk, 5 backups, 10 domains
```

Backend reads plan name and applies appropriate resources.

### Quota Enforcement

**Domain Quota:**
- Tracked in `domains.count` column
- Decremented on domain add
- Incremented on domain delete
- Enforced at backend level

**Backup Quota:**
- Tracked in `domains.backups` column
- Incremented on backup create
- Decremented on backup delete
- Plan-specific limits enforced

## Customer Self-Service

### What Customers Can Do

**Via ProxmoxWP Extension:**
- Start/stop/restart their WordPress site
- Add up to N custom domains (plan-dependent)
- Create backups (up to plan limit)
- Rollback to previous backup
- View site status and resource usage
- Toggle proxy (enable/disable traffic)

**Via DNS Extension:**
- Create DNS zones for their domains
- Add/edit/delete DNS records
- View zone status and nameservers
- Delete entire zones

**Via SSupport Extension:**
- View their support key
- Check subscription status
- Access support portal (future)

### What Customers Cannot Do

- Direct SSH access to containers
- Access Proxmox interface
- Modify infrastructure directly
- View other customers' resources
- Bypass quota limits

## Comparison to Cloud Billing

| Paymenter + Extensions | AWS | Azure |
|------------------------|-----|-------|
| Custom extensions | AWS Marketplace | Azure Marketplace |
| Service lifecycle hooks | CloudFormation | ARM Templates |
| Bearer token auth | IAM API keys | Managed Identities |
| Webhook triggers | EventBridge | Event Grid |
| Customer portal | AWS Console | Azure Portal |

## Key Takeaways

1. **Separation of Concerns** - Paymenter handles billing, backend handles infrastructure
2. **Extensible Architecture** - Easy to add new service types via extensions
3. **Customer Empowerment** - Self-service features reduce support burden
4. **Quota Enforcement** - Plan-based limits prevent abuse
5. **Error Resilience** - Graceful handling of backend failures
6. **Audit Trail** - Service ID tracks all operations

This integration demonstrates:
- API-driven architecture
- Event-driven provisioning
- Multi-service orchestration
- Customer portal development
- Quota and billing integration
- Production-grade error handling

---

## See Also

- **[Automated Workflow](../provisioning/automated_workflow.md)** - Complete provisioning pipeline
- **[Python Backend](../provisioning/python-backend.md)** - Backend API architecture
- **[ProxmoxWP Extension Code](../../code-samples/extension-proxmoxwp.md)** - PHP implementation details
- **[DNS Extension Code](../../code-samples/extension-dns.md)** - DNS management implementation
- **[Support Extension Code](../../code-samples/extension-ssupport.md)** - Support subscription implementation

---

*Paymenter extensions transform infrastructure APIs into a seamless customer experience, handling billing, provisioning, and self-service management through a unified interface.*
