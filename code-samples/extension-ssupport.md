# SSupport Extension - Code Sample

This extension demonstrates a simple support subscription management system integrated with Paymenter. It shows how to create custom service types beyond infrastructure provisioning, with MySQL backend for tracking subscriptions.

## Extension Structure

### Basic Configuration

```php
class SSupport extends Server
{
    public string $name = 'SSupport';
    public string $description = 'Manages support subscriptions through a remote API backend';
    public string $version = '1.0.0';
    
    protected function api(): SSupportApi
    {
        return new SSupportApi(
            (string) $this->config('api_base'),
            (string) $this->config('api_key')
        );
    }
    
    public function getConfig($values = []): array
    {
        return [
            [
                'name' => 'api_base',
                'label' => 'Support API Base URL',
                'type' => 'text',
                'required' => true,
                'description' => 'Example: http://192.168.1.5:8080'
            ],
            [
                'name' => 'api_key',
                'label' => 'Support API Key',
                'type' => 'text',
                'required' => true,
                'description' => 'Bearer token for API authentication'
            ]
        ];
    }
}
```

**Configuration:**
- Minimal setup (only API URL and key)
- No product-specific configuration needed
- Reusable across multiple support plans

## Lifecycle Management

### Create Support Subscription

```php
public function createServer(Service $service, $settings, $properties)
{
    $resp = $this->api()->create(
        (string) $service->id,
        $service->user->email,
        $service->user->first_name ?? 'Customer',
        $service->user->last_name ?? 'User'
    );
    
    // Store support key as property for display
    if (isset($resp['support_key'])) {
        $service->properties()->updateOrCreate(
            ['key' => 'support_key'],
            [
                'name' => 'Support Key',
                'value' => $resp['support_key']
            ]
        );
    }
    
    return [
        'success' => (bool) ($resp['success'] ?? false),
        'data' => $resp
    ];
}
```

**API Call:**
```php
POST /support/create
{
    "service_id": "12345",
    "email": "customer@example.com",
    "firstname": "John",
    "lastname": "Doe"
}

Response:
{
    "success": true,
    "service_id": "12345",
    "support_key": "SUP-ABC123XYZ789",
    "status": "ACTIVE"
}
```

**What Happens:**
1. Backend generates unique support key
2. Creates MySQL record with customer details
3. Returns support key to Paymenter
4. Key stored as service property (visible to customer)

### Support Key Generation

```python
# Backend implementation
import secrets
import string

def generate_support_key():
    """Generate unique support key: SUP-{16 random chars}"""
    chars = string.ascii_uppercase + string.digits
    random_part = ''.join(secrets.choice(chars) for _ in range(16))
    return f"SUP-{random_part}"
```

**Format:** `SUP-ABC123XYZ789DEF456`

**Usage:** Customer provides this key when submitting support tickets.

### Suspend Subscription

```php
public function suspendServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->suspend((string) $service->id);
    } catch (Exception $e) {
        // Log but don't block operation
    }
    
    return true;
}
```

**API Call:**
```php
POST /support/suspend
{"service_id": "12345"}
```

**Backend Action:**
```python
# Update status to SUSPENDED
cursor.execute("""
    UPDATE customers 
    SET status = 'SUSPENDED' 
    WHERE service_id = %s
""", (service_id,))
```

**Effect:** Customer cannot use support key until unsuspended.

### Unsuspend Subscription

```php
public function unsuspendServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->unsuspend((string) $service->id);
    } catch (Exception $e) {}
    
    return true;
}
```

**Backend Action:**
```python
# Update status to ACTIVE
cursor.execute("""
    UPDATE customers 
    SET status = 'ACTIVE' 
    WHERE service_id = %s
""", (service_id,))
```

### Terminate Subscription

```php
public function terminateServer(Service $service, $settings, $properties)
{
    try {
        $this->api()->terminate((string) $service->id);
        
        // Clean up stored property
        $service->properties()->where('key', 'support_key')->delete();
    } catch (Exception $e) {}
    
    return true;
}
```

**Backend Action:**
```python
# Update status to INACTIVE (soft delete)
cursor.execute("""
    UPDATE customers 
    SET status = 'INACTIVE' 
    WHERE service_id = %s
""", (service_id,))
```

**Note:** Record preserved for audit trail, not hard-deleted.

## API Client Implementation

### Request Wrapper

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

### API Methods

```php
public function create(string $serviceId, string $email, string $firstname, string $lastname): array
{
    return $this->request('/support/create', 'POST', [
        'service_id' => $serviceId,
        'email' => $email,
        'firstname' => $firstname,
        'lastname' => $lastname
    ]);
}

public function suspend(string $serviceId): void
{
    $this->request('/support/suspend', 'POST', ['service_id' => $serviceId]);
}

public function unsuspend(string $serviceId): void
{
    $this->request('/support/unsuspend', 'POST', ['service_id' => $serviceId]);
}

public function terminate(string $serviceId): void
{
    $this->request('/support/terminate', 'POST', ['service_id' => $serviceId]);
}
```

**Simple Interface:** All methods follow same pattern (service_id → status change)

## Backend Implementation

### Database Schema

```python
CREATE TABLE customers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id VARCHAR(255) UNIQUE NOT NULL,
    firstname VARCHAR(100) NOT NULL,
    lastname VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    supportkey VARCHAR(50) UNIQUE NOT NULL,
    status ENUM('ACTIVE', 'SUSPENDED', 'INACTIVE') DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE INDEX idx_service_id ON customers(service_id);
CREATE INDEX idx_supportkey ON customers(supportkey);
CREATE INDEX idx_status ON customers(status);
```

**Key Fields:**
- `service_id`: Links to Paymenter service
- `supportkey`: Unique identifier for tickets
- `status`: Current subscription state
- Timestamps for audit trail

### Create Endpoint

```python
@support_bp.route("/support/create", methods=["POST"])
def create_support():
    data = request.get_json(force=True)
    
    # Extract and validate fields
    email = data.get("email", "").strip()
    service_id = data.get("service_id", "").strip()
    firstname = data.get("firstname", "").strip()
    lastname = data.get("lastname", "").strip()
    
    if not all([email, service_id, firstname, lastname]):
        return jsonify({
            "success": False,
            "error": "Missing required fields"
        }), 400
    
    # Validate email format
    if not validate_email(email):
        return jsonify({
            "success": False,
            "error": "Invalid email format"
        }), 400
    
    # Generate support key
    support_key = generate_support_key()
    
    # Insert into database
    try:
        conn = get_conn()
        cursor = conn.cursor()
        
        cursor.execute("""
            INSERT INTO customers 
            (service_id, firstname, lastname, email, supportkey, status)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (service_id, firstname, lastname, email, support_key, "ACTIVE"))
        
        conn.commit()
        conn.close()
        
        return jsonify({
            "success": True,
            "service_id": service_id,
            "support_key": support_key,
            "status": "ACTIVE"
        }), 200
        
    except pymysql.IntegrityError as e:
        # Handle duplicate service_id
        if "duplicate" in str(e).lower():
            return jsonify({
                "success": False,
                "error": f"Service ID '{service_id}' already exists"
            }), 409
        raise
```

### Status Update Helper

```python
def _update_support_status(action: str):
    """
    Generic status updater for suspend/unsuspend/terminate
    """
    data = request.get_json(force=True)
    service_id = data.get("service_id", "").strip()
    
    if not validate_service_id(service_id):
        return jsonify({
            "success": False,
            "error": "Invalid service_id"
        }), 400
    
    # Map action to status
    status_mapping = {
        "suspend": "SUSPENDED",
        "unsuspend": "ACTIVE",
        "terminate": "INACTIVE"
    }
    
    target_status = status_mapping[action]
    
    conn = get_conn()
    cursor = conn.cursor()
    
    # Check if exists
    cursor.execute("""
        SELECT service_id FROM customers WHERE service_id = %s
    """, (service_id,))
    
    if not cursor.fetchone():
        conn.close()
        return jsonify({
            "success": False,
            "error": f"Service ID '{service_id}' not found"
        }), 404
    
    # Update status
    cursor.execute("""
        UPDATE customers SET status = %s WHERE service_id = %s
    """, (target_status, service_id))
    
    conn.commit()
    conn.close()
    
    return jsonify({
        "success": True,
        "service_id": service_id,
        "status": target_status
    }), 200
```

**Idempotent:** Calling suspend multiple times is safe (no error if already suspended).

## Client Area View

### Display Support Key

```php
public function getActions(Service $service, $settings, $properties): array
{
    return [
        ['type' => 'view', 'name' => 'Overview', 'label' => 'Overview']
    ];
}

public function getView(Service $service, $settings, $properties, $viewName)
{
    $supportKey = $service->properties()
        ->where('key', 'support_key')
        ->first()?->value ?? 'N/A';
    
    $status = ucfirst(strtolower((string) $service->status));
    $plan = $service->plan->name ?? $service->product->name;
    $email = $service->user->email;
    
    return view()->file(__DIR__ . '/resources/views/details.blade.php', [
        'supportKey' => $supportKey,
        'status' => $status,
        'plan' => $plan,
        'email' => $email,
        'service' => $service
    ])->render();
}
```

**Blade Template:**
```blade
<div class="card">
    <div class="card-header">
        <h3>Support Subscription</h3>
    </div>
    <div class="card-body">
        <dl class="row">
            <dt class="col-sm-3">Support Key</dt>
            <dd class="col-sm-9">
                <code>{{ $supportKey }}</code>
            </dd>
            
            <dt class="col-sm-3">Status</dt>
            <dd class="col-sm-9">
                <span class="badge badge-{{ $status === 'Active' ? 'success' : 'warning' }}">
                    {{ $status }}
                </span>
            </dd>
            
            <dt class="col-sm-3">Plan</dt>
            <dd class="col-sm-9">{{ $plan }}</dd>
            
            <dt class="col-sm-3">Email</dt>
            <dd class="col-sm-9">{{ $email }}</dd>
        </dl>
        
        <div class="alert alert-info">
            <strong>How to use:</strong> Provide your support key when submitting tickets for priority assistance.
        </div>
    </div>
</div>
```

## Validation Functions

### Email Validation

```python
import re

def validate_email(email: str) -> bool:
    """Validate email format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))
```

### Service ID Validation

```python
def validate_service_id(service_id: str) -> bool:
    """Validate service_id format"""
    if not service_id or not isinstance(service_id, str):
        return False
    
    service_id = service_id.strip()
    
    if len(service_id) < 1 or len(service_id) > 255:
        return False
    
    return True
```

## Error Handling

### Duplicate Prevention

```python
try:
    cursor.execute("""
        INSERT INTO customers (service_id, ...) VALUES (%s, ...)
    """, (service_id, ...))
except pymysql.IntegrityError as e:
    if "duplicate" in str(e).lower():
        return jsonify({
            "success": False,
            "error": "Service already exists"
        }), 409
```

**HTTP 409 Conflict:** Indicates duplicate service_id.

### Not Found Handling

```python
cursor.execute("SELECT * FROM customers WHERE service_id = %s", (service_id,))

if not cursor.fetchone():
    return jsonify({
        "success": False,
        "error": "Service not found"
    }), 404
```

**HTTP 404:** Service doesn't exist in database.

## Use Cases

### Priority Support

**Tier System:**
- Free tier: No support key, community forums only
- Basic tier: Support key, email support within 48 hours
- Premium tier: Support key, priority email + live chat within 24 hours
- Enterprise tier: Support key, dedicated support manager

**Verification:**
Support ticket system checks `supportkey` against database:
```python
def verify_support_key(key: str) -> dict:
    cursor.execute("""
        SELECT service_id, firstname, lastname, email, status
        FROM customers
        WHERE supportkey = %s AND status = 'ACTIVE'
    """, (key,))
    
    customer = cursor.fetchone()
    return customer if customer else None
```

### License Management

**Alternative Use:** Support keys can serve as software license keys
- Customer enters key to activate software
- Backend verifies key is active
- Software checks periodically (online activation)

## Security Considerations

### Support Key Security

**Generation:**
- Cryptographically secure random generator
- 16 characters (uppercase + digits)
- Collision probability: 1 in 36^16 (extremely low)

**Storage:**
- Plain text in database (not sensitive like passwords)
- Indexed for fast lookup
- Unique constraint prevents duplicates

### API Authentication

**Bearer Token:**
- Required on all endpoints
- Validated before processing
- Prevents unauthorized access

## Key Design Patterns

**Simplicity:**
- Minimal configuration
- Single-purpose extension
- Clear API contract

**Idempotency:**
- Status updates safe to repeat
- No side effects from duplicate calls

**Soft Deletion:**
- Terminate sets status to INACTIVE
- Record preserved for audit
- Can be reactivated if needed

**Property Storage:**
- Support key stored as Paymenter property
- Accessible to customer
- Persists across sessions

## Cloud Comparison

| Feature | This Extension | AWS Support Plans | Azure Support |
|---------|----------------|-------------------|---------------|
| Key generation | Custom SUP-XXX | Case numbers | Ticket IDs |
| Status tracking | MySQL | Internal | Internal |
| Integration | Paymenter API | AWS API | Azure Portal |
| Lifecycle | Create/Suspend/Terminate | Plan changes | Subscription changes |

## Key Takeaways

1. **Service Type Flexibility** - Extensions aren't just for infrastructure
2. **Simple State Management** - MySQL status tracking
3. **Unique Identifiers** - Support keys as customer-facing IDs
4. **Lifecycle Integration** - Suspend/unsuspend/terminate patterns
5. **Property Storage** - Paymenter properties for customer-visible data

This implementation demonstrates:
- Non-infrastructure service automation
- Database-backed state tracking
- Unique identifier generation
- Customer-facing property display
- Simple API design

---

## See Also

- **[Paymenter Integration Architecture](../docs/infrastructure/paymenter.md)** - Complete billing system overview
- **[Python Backend API](../docs/provisioning/python-backend.md)** - Backend API patterns
- **[ProxmoxWP Extension](extension-proxmoxwp.md)** - Infrastructure provisioning example

---

*This extension shows how Paymenter can manage any subscription-based service, not just infrastructure, by abstracting lifecycle management into clean API calls.*
