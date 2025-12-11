# ProxmoxWP Extension - Code Sample

This extension demonstrates how Paymenter integrates with the Python backend to provide automated WordPress hosting. The code shows production patterns for API communication, lifecycle management, and customer self-service features.

## Extension Structure

### Base Configuration

```php
class ProxmoxWP extends Server
{
    public string $name = 'ProxmoxWP';
    public string $version = '1.4.0';
    
    protected function api(): ProxmoxWPApi
    {
        return new ProxmoxWPApi(
            $this->config('api_base'),
            $this->config('api_key')
        );
    }
    
    public function getConfig($values = []): array
    {
        return [
            [
                'name' => 'api_base',
                'label' => 'Backend API Base URL',
                'type' => 'text',
                'required' => true,
                'description' => 'Example: http://192.168.1.5:8080'
            ],
            [
                'name' => 'api_key',
                'label' => 'Backend API Key',
                'type' => 'text',
                'required' => true
            ],
            [
                'name' => 'public_ip',
                'label' => 'Public IP',
                'type' => 'text',
                'required' => true,
                'description' => 'Public IP for site creation'
            ]
        ];
    }
}
```

**Configuration Fields:**
- `api_base`: Python backend URL
- `api_key`: Bearer token for authentication
- `public_ip`: Reverse proxy IP address

## Lifecycle Hooks

### Service Creation

```php
public function createServer(Service $service, $settings, $properties)
{
    $resp = $this->api()->createSite(
        (string) $service->id,
        $service->user->email,
        $service->plan->name ?? $service->product->name,
        (string) $this->config('public_ip')
    );
    
    return [
        'success' => (bool) ($resp['success'] ?? false),
        'data' => $resp
    ];
}
```

**API Call:**
```php
POST /create-site
{
    "service_id": "12345",
    "email": "customer@example.com",
    "plan": "basic",
    "public_ip": "203.0.113.10"
}
```

**What Happens:**
1. Container created on DRBD storage
2. DNS record created (subdomain.nodepoint.eu)
3. Reverse proxy configured with SSL
4. WordPress installed automatically
5. Initial snapshot created
6. Domain quota initialized

### Service Termination

```php
public function terminateServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->deleteSite((string) $service->id);
    } catch (Exception $e) {
        // Log but don't block termination
    }
    
    return true;
}
```

**API Call:**
```php
POST /delete-site
{"service_id": "12345"}
```

**Cleanup Actions:**
- Container stopped and deleted
- All snapshots removed
- HA resources deleted
- DNS records removed
- Proxy configuration removed
- Custom domains deleted
- Database entries cleaned

### Suspend/Unsuspend

```php
public function suspendServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->suspend((string) $service->id);
    } catch (Exception $e) {}
    return true;
}

public function unsuspendServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->unsuspend((string) $service->id);
    } catch (Exception $e) {}
    return true;
}
```

**Behavior:**
- Suspend: Container stopped, data preserved
- Unsuspend: Container started, services resume

### Plan Upgrade

```php
public function upgradeServer(Service $service, $settings, $properties)
{
    try {
        $plan = $service->plan->name ?? $service->product->name;
        $this->api()->upgradePlan((string) $service->id, (string) $plan);
    } catch (Exception $e) {
        return false;  // Upgrade failed
    }
    
    return true;
}
```

**API Call:**
```php
POST /wp/upgrade/
{
    "service_id": "12345",
    "plan": "premium"
}
```

**Resource Changes:**
- Container resized (RAM, disk)
- Backup quota updated
- Domain quota updated

## Customer Actions

### Control Buttons

```php
public function getActions(Service $service, $settings, $properties): array
{
    return [
        ['type' => 'view', 'name' => 'Overview', 'label' => 'Overview'],
        ['type' => 'button', 'label' => 'Start', 'function' => 'actionStart'],
        ['type' => 'button', 'label' => 'Stop', 'function' => 'actionStop'],
        ['type' => 'button', 'label' => 'Restart', 'function' => 'actionRestart'],
        ['type' => 'button', 'label' => 'Reset', 'function' => 'actionReset']
    ];
}

public function actionStart(Service $service, $settings, $properties)
{
    $this->api()->action($service->id, 'start', $service->status);
    return null;
}

public function actionStop(Service $service, $settings, $properties)
{
    $this->api()->action($service->id, 'stop', $service->status);
    return null;
}

public function actionRestart(Service $service, $settings, $properties)
{
    $this->api()->action($service->id, 'restart', $service->status);
    return null;
}

public function actionReset(Service $service, $settings, $properties)
{
    // Rollback to 'initial' snapshot
    $this->api()->action($service->id, 'reset', $service->status);
    return null;
}
```

**Backend Validation:**
- Checks service status (rejects if suspended)
- Verifies service exists
- Executes action via Proxmox API

### Client Area View

```php
public function getView(Service $service, $settings, $properties, $viewName)
{
    $serviceId = (string) $service->id;
    
    // Fetch status from backend
    try {
        $status = $this->api()->status($serviceId);
        $subdomain = $status['subdomain'] ?? 'unavailable';
        $state = ucfirst($status['status'] ?? 'Unknown');
    } catch (Exception $e) {
        $statusErr = $this->cleanError($e->getMessage());
    }
    
    // Handle form submissions (domains, backups)
    $action = request()->input('pwx_action', '');
    
    if ($action === 'add_domain') {
        $domain = request()->input('pwx_domain');
        $this->api()->domainAdd($serviceId, $domain);
        $notice = "Domain added: {$domain}";
    }
    
    // Fetch custom domains
    $customDomains = $this->api()->domainsList($serviceId);
    
    // Fetch backups
    $backups = $this->api()->getBackups($serviceId);
    
    return view()->file('details.blade.php', [
        'subdomain' => $subdomain,
        'state' => $state,
        'customDomains' => $customDomains,
        'backups' => $backups,
        'notice' => $notice
    ])->render();
}
```

## API Client Implementation

### HTTP Request Wrapper

```php
protected function request(string $path, string $method = 'GET', array $data = []): array
{
    $url = $this->base . $path;
    
    $req = Http::withHeaders([
        'Authorization' => 'Bearer ' . $this->key,
        'Accept' => 'application/json'
    ])->timeout(25);
    
    $resp = match (strtoupper($method)) {
        'POST' => $req->asJson()->post($url, $data),
        'PATCH' => $req->asJson()->patch($url, $data),
        'DELETE' => $req->asJson()->delete($url, $data),
        default => $req->get($url, $data)
    };
    
    if ($resp->failed()) {
        throw new Exception("API {$method} {$path} failed: HTTP " . $resp->status());
    }
    
    return $resp->json() ?? [];
}
```

**Features:**
- Bearer token authentication
- 25-second timeout
- JSON request/response
- HTTP method flexibility
- Error handling

### Domain Management

```php
public function domainsList(string $serviceId): array
{
    return $this->request('/action/getdomain', 'GET', ['service_id' => $serviceId]);
}

public function domainAdd(string $serviceId, string $domain): void
{
    $this->request('/action/setdomain', 'POST', [
        'service_id' => $serviceId,
        'domain' => $domain
    ]);
}

public function domainDel(string $serviceId, string $domain): void
{
    $this->request('/action/deldomain', 'POST', [
        'service_id' => $serviceId,
        'domain' => $domain
    ]);
}
```

**Backend Actions:**
- Verifies domain DNS points to correct IP
- Creates proxy host with SSL certificate
- Decrements domain quota
- Tracks in database

### Backup Management

```php
public function getBackups(string $serviceId): array
{
    return $this->request('/action/getbak', 'GET', ['service_id' => $serviceId]);
}

public function mkBackup(string $serviceId): void
{
    $this->request('/action/mkbak', 'POST', ['service_id' => $serviceId]);
}

public function delBackup(string $serviceId, string $backupId): void
{
    $this->request('/action/delbak', 'POST', [
        'service_id' => $serviceId,
        'bak' => $backupId
    ]);
}

public function rollbackBackup(string $serviceId, string $backupId): void
{
    $this->request('/action/rollback', 'POST', [
        'service_id' => $serviceId,
        'bak' => $backupId
    ]);
}
```

**Backup Naming:**
- Format: `BAK{DDMMYYYY}-{N}`
- Example: `BAK11122025-1`, `BAK11122025-2`
- Incremental numbering per day

## Error Handling

### Clean Error Messages

```php
private function cleanError(string $msg): string
{
    // Try JSON in message
    if (preg_match('/\{.*\}/s', $msg, $m)) {
        $json = json_decode($m[0], true);
        if (isset($json['error'])) {
            return trim($json['error']);
        }
    }
    
    // Try <p>...</p>
    if (preg_match('/<p[^>]*>(.*?)<\/p>/si', $msg, $m)) {
        $txt = strip_tags($m[1]);
        if ($txt !== '') {
            return trim($txt);
        }
    }
    
    // Strip API prefix
    $clean = preg_replace('/^API\s+\w+\s+\/[^\s]+\s+failed:\s*HTTP\s*\d+\s*/i', '', $msg);
    return trim($clean ?? $msg);
}
```

**User-Friendly Errors:**
- "Domain already exists" instead of "API POST /action/setdomain failed: HTTP 409 {error: 'Domain already exists'}"
- "Backup limit exceeded" instead of raw API error
- Clean HTML from error responses

### Graceful Failures

```php
try {
    $customDomains = $this->api()->domainsList($serviceId);
} catch (Exception $e) {
    // Don't block entire page if domain list fails
    $customDomains = [];
    if (!$notice) {
        $notice = 'Failed to load domains: ' . $this->cleanError($e->getMessage());
        $tone = 'error';
    }
}
```

**Principle:** Non-critical failures shouldn't break the entire interface.

## Toggle Feature

### Proxy Enable/Disable

```php
public function toggle(string $serviceId): void
{
    $this->request('/action/toggle', 'POST', [
        'service_id' => $serviceId
    ]);
}
```

**Backend Implementation:**
1. Lookup subdomain by service_id
2. Find proxy host in NPM
3. Toggle `enabled` field
4. Return new state

**Use Case:** Customer wants to temporarily disable their site without deleting it.

## Key Design Patterns

**API Adapter Pattern:**
- Extension translates Paymenter events to API calls
- Decouples billing system from infrastructure

**Idempotency:**
- Suspend/unsuspend can be called multiple times safely
- Domain add checks for duplicates

**Error Isolation:**
- API failures in one feature don't crash entire page
- Errors displayed to user without technical details

**Service ID Tracking:**
- Consistent identifier throughout system
- Links billing, infrastructure, and customer data

## Security Considerations

**Token Protection:**
- API key stored in extension config
- Never exposed to customer
- Validated on every backend request

**Input Validation:**
- Service ID format validation
- Email validation
- Domain sanitization

**Status Checks:**
- Suspended services rejected by backend
- Quota limits enforced server-side

## Cloud Translation

This pattern directly maps to cloud marketplaces:

**AWS Marketplace:**
```python
# Similar pattern
def create_subscription(customer_id, product_id):
    # Call CloudFormation to provision resources
    stack = cloudformation.create_stack(...)
```

**Azure Marketplace:**
```csharp
// Similar pattern
public async Task CreateSubscription(string customerId, string planId)
{
    // Call ARM template deployment
    await armClient.Deployments.CreateOrUpdateAsync(...);
}
```

## Key Takeaways

1. **Clean Abstraction** - Extension hides complexity from Paymenter core
2. **Event-Driven** - Hooks triggered by Paymenter lifecycle events
3. **RESTful Communication** - Standard HTTP API calls
4. **Error Resilience** - Graceful handling of failures
5. **Customer Empowerment** - Self-service features reduce support load

This implementation demonstrates:
- PHP API client development
- Webhook handling
- Error handling and UX
- Multi-service orchestration
- Production-grade integration patterns

---

## See Also

- **[Paymenter Integration Architecture](../docs/infrastructure/paymenter.md)** - Complete billing system overview
- **[Python Backend API](../docs/provisioning/python-backend.md)** - Backend endpoints this extension calls
- **[Automated Workflow](../docs/provisioning/automated_workflow.md)** - End-to-end provisioning flow

---

*This extension transforms infrastructure APIs into billing-integrated customer services, handling provisioning, management, and self-service through a clean PHP interface.*
