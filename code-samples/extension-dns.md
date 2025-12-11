# DNS Extension - Code Sample

This extension provides customer self-service DNS management through Paymenter, integrating with ISPConfig via the Python backend. Customers can create zones, add records, and manage their domains without admin intervention.

## Extension Structure

### Route-Based Architecture

```php
Route::middleware(['web', 'auth'])->group(function () {
    // Zone management
    Route::get('/dns', 'listZones')->name('dns.index');
    Route::post('/dns/add-zone', 'addZone')->name('dns.add-zone');
    Route::post('/dns/{zone}/delete', 'deleteZone')->name('dns.zone.delete');
    
    // Record management
    Route::get('/dns/{zone}', 'viewZone')->name('dns.zone');
    Route::post('/dns/{zone}/records/add', 'addRecord')->name('dns.record.add');
    Route::post('/dns/{zone}/records/{record}/delete', 'deleteRecord')->name('dns.record.delete');
});
```

**Pattern:** Laravel routes instead of Paymenter server extension (allows more flexibility)

### Environment Configuration

```php
$backendBase = env('DNS_BACKEND_URL', 'http://192.168.1.5:8080/customdomain');
```

**Single Configuration:** Backend URL points to ISPConfig integration endpoint.

## Zone Management

### List All Zones

```php
Route::get('/dns', function () {
    $userId = (string) auth()->id();
    
    $response = Http::timeout(10)->post($backendBase . '/getdata', [
        'user_id' => $userId
    ]);
    
    if ($response->successful()) {
        $data = $response->json();
        $zones = collect($data['zones'] ?? [])
            ->map(function ($item) {
                $z = $item['zone'] ?? $item;
                
                // Extract nameservers from NS records
                $nameservers = collect($z['records'] ?? [])
                    ->filter(fn($r) => $r['type'] === 'NS')
                    ->map(fn($r) => $r['data'])
                    ->values()
                    ->all();
                
                return [
                    'id' => $z['id'],
                    'name' => $z['name'],
                    'status' => 'active',
                    'records_count' => count($z['records'] ?? []),
                    'nameservers' => $nameservers
                ];
            })->all();
    }
    
    return view('dns::index', ['zones' => $zones, 'error' => $error]);
});
```

**Backend Call:**
```json
POST /customdomain/getdata
{
    "user_id": "123"
}

Response:
{
    "zones": [
        {
            "zone": {
                "id": "5",
                "name": "example.com",
                "records": {
                    "12": {"type": "A", "name": "example.com.", "data": "203.0.113.10"},
                    "13": {"type": "NS", "name": "example.com.", "data": "ns1.nodepoint.eu."}
                }
            }
        }
    ]
}
```

### Create Zone

```php
Route::post('/dns/add-zone', function (Request $request) {
    $request->validate([
        'domain' => ['required', 'string']
    ]);
    
    $domain = trim($request->input('domain'));
    
    $response = Http::timeout(20)->post($backendBase . '/add/zone', [
        'user_id' => (string) auth()->id(),
        'domain' => $domain
    ]);
    
    $body = trim((string) $response->body());
    
    if ($response->successful() && stripos($body, 'success') === 0) {
        session()->flash('success', "Zone '{$domain}' successfully added.");
    } else {
        session()->flash('error', "Failed to add zone: {$body}");
    }
    
    return redirect()->route('dns.index');
});
```

**Backend Validation:**
1. Verifies domain nameservers point to ns1/ns2.nodepoint.eu
2. Creates ISPConfig DNS zone
3. Adds default NS records for apex
4. Adds default A records (apex + www)

**Nameserver Verification:**
```python
# Backend checks DNS for domain
answers = resolver.resolve(domain, 'NS')
found_ns = {str(rdata.target).lower().rstrip('.') for rdata in answers}

if not expected_ns.issubset(found_ns):
    raise RuntimeError("Domain not configured with correct nameservers")
```

**Result:** Zone only created if domain is properly delegated.

### Delete Zone

```php
Route::post('/dns/{zone}/delete', function (string $zone) {
    $response = Http::timeout(20)->post($backendBase . '/delete/zone', [
        'domain' => $zone
    ]);
    
    $body = trim((string) $response->body());
    
    if ($response->successful() && stripos($body, 'success') === 0) {
        session()->flash('success', "Zone '{$zone}' deleted.");
    } else {
        session()->flash('error', "Failed to delete zone: {$body}");
    }
    
    return redirect()->route('dns.index');
});
```

**Backend Action:**
- Deletes zone from ISPConfig
- Automatically removes all associated records
- Returns simple text response

## Record Management

### View Zone Records

```php
Route::get('/dns/{zone}', function (string $zone) {
    $userId = (string) auth()->id();
    
    $response = Http::timeout(10)->post($backendBase . '/getdata', [
        'user_id' => $userId
    ]);
    
    if ($response->successful()) {
        $data = $response->json();
        
        // Find zone
        $zoneData = collect($data['zones'] ?? [])
            ->map(fn($item) => $item['zone'] ?? $item)
            ->first(fn($z) => $z['name'] === $zone);
        
        if ($zoneData) {
            $records = collect($zoneData['records'] ?? [])
                ->reject(fn($r) => $r['type'] === 'NS')  // Hide NS records
                ->map(function ($r) {
                    // Map PROXYSRV alias
                    $content = $r['data'];
                    if ($content === '203.0.113.10') {
                        $content = 'PROXYSRV';
                    }
                    
                    return [
                        'id' => $r['id'],
                        'type' => $r['type'],
                        'name' => $r['name'],
                        'content' => $content,
                        'ttl' => $r['ttl'],
                        'priority' => $r['priority'] ?? null,
                        'active' => $r['active'] === 'y'
                    ];
                })->values()->all();
        }
    }
    
    return view('dns::zone', [
        'zone' => $zoneData,
        'records' => $records,
        'error' => $error
    ]);
});
```

**PROXYSRV Alias:**
- Display `PROXYSRV` instead of actual proxy IP
- Simplifies customer experience
- Translated back to real IP on record creation

### Add Record

```php
Route::post('/dns/{zone}/records/add', function (Request $request, string $zone) {
    $request->validate([
        'type' => ['required', 'string'],
        'name' => ['required', 'string'],
        'content' => ['nullable', 'string'],
        'ttl' => ['required', 'integer']
    ]);
    
    $type = strtolower($request->input('type'));
    $name = trim($request->input('name'));
    $content = trim((string) $request->input('content', ''));
    $ttl = (int) $request->input('ttl', 3600);
    $priority = $request->input('priority');
    
    // Translate PROXYSRV → actual IP
    if (strcasecmp($content, 'PROXYSRV') === 0) {
        $content = '203.0.113.10';
    }
    
    $payload = [
        'type' => $type,
        'name' => $name,
        'ttl' => $ttl
    ];
    
    // Map by record type
    switch ($type) {
        case 'a':
        case 'aaaa':
            $payload['ip'] = $content;
            break;
        
        case 'cname':
            $payload['target'] = $content;
            break;
        
        case 'mx':
            $payload['exchange'] = $content;
            $payload['priority'] = (int) ($priority ?: 10);
            break;
        
        case 'txt':
            $payload['txt'] = $content;
            break;
        
        case 'ns':
            $payload['ns'] = $content;
            break;
        
        default:
            session()->flash('error', "Unsupported record type: {$type}");
            return redirect()->route('dns.zone', $zone);
    }
    
    $response = Http::timeout(20)->post($backendBase . '/add/record', $payload);
    
    if ($response->successful()) {
        session()->flash('success', 'DNS record added.');
    } else {
        session()->flash('error', 'Failed to add record.');
    }
    
    return redirect()->route('dns.zone', $zone);
});
```

**Backend Processing:**
1. Auto-detects zone from FQDN if zone_id not provided
2. Validates record data
3. Calls appropriate ISPConfig API (dns_a_add, dns_cname_add, etc.)
4. Returns success/failure

### Delete Record

```php
Route::post('/dns/{zone}/records/{record}/delete', function (Request $request, string $zone, string $record) {
    $zoneId = $request->input('zone_id');
    
    if (empty($zoneId)) {
        session()->flash('error', 'Missing zone_id.');
        return redirect()->route('dns.zone', $zone);
    }
    
    $response = Http::timeout(20)->post($backendBase . '/delete/record', [
        'zone_id' => (int) $zoneId,
        'rc_id' => (int) $record
    ]);
    
    $body = trim((string) $response->body());
    
    if ($response->successful() && stripos($body, 'success') === 0) {
        session()->flash('success', 'DNS record deleted.');
    } else {
        session()->flash('error', "Failed to delete record: {$body}");
    }
    
    return redirect()->route('dns.zone', $zone);
});
```

**Backend Action:**
- Verifies record belongs to specified zone
- Determines record type (A, CNAME, MX, etc.)
- Calls appropriate ISPConfig delete API

## ISPConfig Integration

### Backend API Wrapper

The Python backend wraps ISPConfig SOAP/REST API:

**Login Flow:**
```python
def _login():
    return _rest("login", {
        "username": ISPC_USER,
        "password": ISPC_PASS
    })

def _rest(method, payload):
    url = f"{ISPC_URL}/remote/json.php?{method}"
    response = session.post(url, json=payload)
    data = response.json()
    
    if data.get("code") != "ok":
        raise RuntimeError(data.get("message"))
    
    return data.get("response")
```

**Session Management:**
- Login before operations
- Logout after completion
- Session ID passed in all requests

### Zone Creation

```python
# Create zone
zone_params = {
    "server_id": ISPC_SERVER_ID,
    "origin": domain + ".",
    "ns": "ns1.nodepoint.eu.",
    "ns1": "ns1.nodepoint.eu.",
    "ns2": "ns2.nodepoint.eu.",
    "mbox": f"hostmaster.{domain}.",
    "serial": int(time.strftime("%Y%m%d01")),
    "refresh": 86400,
    "retry": 7200,
    "expire": 3600000,
    "minimum": 3600,
    "ttl": 3600,
    "active": "y"
}

zone_id = _rest("dns_zone_add", {
    "session_id": sid,
    "client_id": client_id,
    "params": zone_params
})

# Add default NS records
for ns in ["ns1.nodepoint.eu", "ns2.nodepoint.eu"]:
    _rest("dns_ns_add", {
        "session_id": sid,
        "client_id": client_id,
        "params": {
            "server_id": ISPC_SERVER_ID,
            "zone": zone_id,
            "name": domain + ".",
            "type": "ns",
            "data": ns + ".",
            "ttl": 3600,
            "active": "y"
        }
    })

# Add default A records (apex + www)
for host in [domain + ".", "www"]:
    _rest("dns_a_add", {
        "session_id": sid,
        "client_id": client_id,
        "params": {
            "server_id": ISPC_SERVER_ID,
            "zone": zone_id,
            "name": host,
            "type": "a",
            "data": TARGET_IP,
            "ttl": 3600,
            "active": "y"
        }
    })
```

### Record Creation

```python
# Example: A record
_rest("dns_a_add", {
    "session_id": sid,
    "client_id": client_id,
    "params": {
        "server_id": ISPC_SERVER_ID,
        "zone": zone_id,
        "name": name,
        "type": "a",
        "data": ip_address,
        "ttl": ttl,
        "active": "y"
    }
})

# Example: MX record
_rest("dns_mx_add", {
    "session_id": sid,
    "client_id": client_id,
    "params": {
        "server_id": ISPC_SERVER_ID,
        "zone": zone_id,
        "name": name,
        "type": "mx",
        "data": mail_server,
        "aux": priority,  # MX priority
        "ttl": ttl,
        "active": "y"
    }
})
```

## Supported Record Types

### A Record (IPv4)
```php
type: 'a'
name: 'subdomain'
content: '203.0.113.10' or 'PROXYSRV'
ttl: 3600
```

### AAAA Record (IPv6)
```php
type: 'aaaa'
name: 'subdomain'
content: '2001:db8::1'
ttl: 3600
```

### CNAME Record (Alias)
```php
type: 'cname'
name: 'www'
content: 'example.com.'
ttl: 3600
```

### MX Record (Mail)
```php
type: 'mx'
name: '@' or 'example.com.'
content: 'mail.example.com.'
priority: 10
ttl: 3600
```

### TXT Record (Text)
```php
type: 'txt'
name: '@'
content: 'v=spf1 mx -all'
ttl: 3600
```

### NS Record (Nameserver)
```php
type: 'ns'
name: 'subdomain'
content: 'ns1.provider.com.'
ttl: 3600
```

## User Experience Features

### Flash Messages

```php
// Success
session()->flash('success', 'Zone created successfully.');

// Error
session()->flash('error', 'Failed to add record: Domain not delegated.');

// Warning
session()->flash('warning', 'Zone exists but may not be propagated yet.');
```

**Display:** Bootstrap alerts in Blade templates

### Record Filtering

```php
// Hide NS records from display (managed automatically)
$records = collect($records)
    ->reject(fn($r) => $r['type'] === 'NS')
    ->values()
    ->all();
```

**Reason:** NS records managed by system, shouldn't be user-editable.

### Content Translation

```php
// Display
if ($content === '203.0.113.10') {
    $content = 'PROXYSRV';
}

// Submit
if (strcasecmp($content, 'PROXYSRV') === 0) {
    $content = '203.0.113.10';
}
```

**User-Friendly:** Customer doesn't need to remember proxy IP.

## Security Considerations

### Authentication

```php
Route::middleware(['web', 'auth'])->group(function () {
    // Only authenticated users can access
});
```

**Paymenter Auth:** Uses existing Paymenter user session.

### User Isolation

```php
$userId = (string) auth()->id();

// Backend filters zones by customer_no derived from user_id
$cust_no = "P" + user_id;
$client = _client_by_customer_no(sid, cust_no);
```

**Result:** Customers only see their own zones.

### DNS Validation

**Backend Checks:**
- Nameserver verification before zone creation
- Domain ownership via DNS lookup
- Record type validation
- TTL range validation
- Content format validation

## Error Handling

### Backend Errors

```php
try {
    $response = Http::timeout(20)->post($backendBase . '/add/zone', $data);
    
    if ($response->successful()) {
        // Success
    } else {
        session()->flash('error', "Failed: {$response->body()}");
    }
} catch (\Throwable $e) {
    session()->flash('error', "Error: {$e->getMessage()}");
}
```

### User-Friendly Messages

Backend returns simple text:
- `"Success"` → Flash success message
- `"Zona već postoji"` → Flash error with message
- `"Neispravni nameserveri"` → Explain delegation issue

## Key Design Patterns

**Route-Based Extension:**
- More flexible than server extension
- Can use full Laravel features
- Custom UI/UX

**Session Flash Messages:**
- User feedback without JavaScript
- Standard Laravel pattern
- Bootstrap integration

**Backend API Abstraction:**
- Extension doesn't talk to ISPConfig directly
- Backend handles complexity
- Clean separation of concerns

**PROXYSRV Alias:**
- Simplifies customer experience
- Hides infrastructure details
- Bidirectional translation

## Cloud Comparison

| Feature | This Extension | Route 53 (AWS) | Azure DNS |
|---------|----------------|----------------|-----------|
| Zone creation | Via Paymenter UI | Console/API | Portal/API |
| Record types | A, AAAA, CNAME, MX, TXT, NS | All | All |
| Nameserver verification | Required | Automatic | Automatic |
| User isolation | Customer No | AWS Account | Subscription |
| API | ISPConfig + Python | Route 53 API | Azure DNS API |

## Key Takeaways

1. **Customer Empowerment** - Self-service DNS without admin
2. **Integration Layer** - Backend abstracts ISPConfig complexity
3. **Validation First** - Nameserver check prevents misconfiguration
4. **User-Friendly** - PROXYSRV alias, flash messages, clean UI
5. **Secure Isolation** - Customers only access their zones

This implementation demonstrates:
- Laravel route development
- API integration patterns
- User experience design
- DNS management at scale
- ISPConfig automation

---

## See Also

- **[Paymenter Integration Architecture](../docs/infrastructure/paymenter.md)** - Complete billing system overview
- **[ISPConfig Infrastructure](../docs/infrastructure/ispc.md)** - DNS server configuration details
- **[Python Backend API](../docs/provisioning/python-backend.md)** - Backend ISPConfig integration

---

*This extension transforms ISPConfig DNS management into a customer-friendly self-service portal, handling zone creation, record management, and validation through a clean PHP interface.*
