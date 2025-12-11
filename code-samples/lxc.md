# LXC Container Provisioning - Code Sample

This code demonstrates the core container provisioning logic used in the automated WordPress hosting platform. The implementation shows production-grade patterns for container lifecycle management, error handling, and infrastructure orchestration.

## Container Manager Implementation

### Core Provisioning Method

```python
class ContainerManager:
    """Manages LXC container lifecycle on Proxmox cluster"""
    
    def __init__(self):
        self.nodes = ["node1", "node2"]  # Cluster nodes
        self.bridge = "vmbr1"             # Private network bridge
        self.template = "local:vztmpl/ubuntu-24.04.tar.gz"
        self.home_node = "node2"          # Preferred creation node
    
    def create_container(
        self, 
        vmid: int, 
        subdomain: str, 
        private_ip: str, 
        memory: int, 
        disk: int
    ) -> dict:
        """
        Create new LXC container with WordPress configuration
        
        Args:
            vmid: Proxmox container ID
            subdomain: Hostname for container
            private_ip: Private network IP (192.168.1.x)
            memory: RAM in MB
            disk: Root disk size in GB
            
        Returns:
            Container creation result
        """
        print(f"[INFO] Creating container {vmid} with {memory}MB RAM, {disk}GB disk")
        
        # Create container via Proxmox API
        result = proxmox_client.create_container(
            node=self.home_node,
            vmid=vmid,
            template=self.template,
            hostname=subdomain,
            memory=memory,
            disk=disk,
            bridge=self.bridge,
            ip=private_ip
        )
        
        print(f"[SUCCESS] Container {vmid} created")
        return result
```

### Container Configuration

```python
def build_container_config(vmid, hostname, memory, disk, ip):
    """Build Proxmox container configuration"""
    return {
        "vmid": vmid,
        "hostname": hostname,
        "ostemplate": "local:vztmpl/ubuntu-24.04.tar.gz",
        "storage": "global-lvm",  # DRBD distributed storage
        "rootfs": f"global-lvm:{disk}",
        "memory": memory,
        "swap": 512,
        "cores": 2,
        "net0": f"name=eth0,bridge=vmbr1,ip={ip}/24,gw=192.168.1.1",
        "unprivileged": 1,  # Security: unprivileged container
        "onboot": 1,        # Auto-start on node boot
        "features": "nesting=1",  # Allow nested containers if needed
        "nameserver": "192.168.1.1 1.1.1.1",
        "searchdomain": "lan"
    }
```

## High Availability Configuration

### HA Resource Creation

```python
def add_ha_resource_and_rule(self, vmid: int) -> bool:
    """
    Configure High Availability for container
    
    Creates HA resource and migration rules to enable automatic
    failover between cluster nodes
    """
    print(f"[INFO] Configuring HA for CT {vmid}")
    
    # Create HA resource
    proxmox_client.create_ha_resource(
        sid=f"ct:{vmid}",
        state="started",
        max_restart=3,   # Restart attempts on same node
        max_relocate=2   # Migration attempts between nodes
    )
    
    # Create HA rule for node group
    proxmox_client.create_ha_rule(
        rule_name=f"rule-{vmid}",
        resources=f"ct:{vmid}",
        nodes="node1,node2"  # Eligible failover nodes
    )
    
    print(f"[SUCCESS] HA configured for CT {vmid}")
    return True
```

### HA Behavior

**Automatic Actions:**
- Service crash → Restart on same node (up to 3 times)
- Node failure → Migrate to alternate node (up to 2 times)
- Uses DRBD storage for instant access to disk on failover

**Failover Time:** ~2 minutes (vs 30+ without distributed storage)

## WordPress Installation

### Installation via SSH

```python
def install_wordpress(vmid: int, subdomain: str):
    """
    Install complete LAMP stack and WordPress in container
    
    Uses SSH to execute pct command on Proxmox node,
    streaming setup script into container
    """
    print(f"[INFO] Installing WordPress on CT{vmid}")
    
    # Find which node hosts the container
    node = proxmox_client.find_container_node(vmid)
    
    # Wait for container to be ready
    wait_for_container_ready(vmid, node, max_attempts=6)
    
    # Execute WordPress setup script
    script_path = "app/scripts/wordpress-setup.sh"
    exec_command = [
        "ssh", node,
        "pct", "exec", str(vmid), "--", 
        "bash", "-s", subdomain
    ]
    
    # Stream script content to container stdin
    with open(script_path, 'r') as script:
        result = subprocess.run(
            exec_command,
            input=script.read(),
            capture_output=True,
            text=True
        )
    
    if result.returncode != 0:
        raise Exception(f"WordPress install failed: {result.stderr}")
    
    # Verify installation
    verify_wordpress_installed(vmid, node)
    
    # Create initial snapshot
    proxmox_client.create_snapshot(node, vmid, "initial")
    
    print(f"[SUCCESS] WordPress installed on CT{vmid}")
```

### Container Readiness Check

```python
def wait_for_container_ready(vmid, node, max_attempts=6):
    """
    Poll container until it responds to commands
    
    Ensures network and services are available before
    attempting WordPress installation
    """
    for attempt in range(max_attempts):
        check_cmd = [
            "ssh", node,
            "pct", "exec", str(vmid), "--",
            "echo", "ready"
        ]
        
        result = subprocess.run(
            check_cmd,
            capture_output=True,
            timeout=5
        )
        
        if result.returncode == 0:
            print(f"[INFO] Container {vmid} ready")
            return True
        
        print(f"[INFO] Waiting for CT{vmid}... ({attempt + 1}/{max_attempts})")
        time.sleep(5)
    
    raise Exception(f"Container {vmid} did not become ready")
```

## Container Deletion

### Complete Cleanup Process

```python
def delete_container(self, vmid: int, subdomain: str) -> bool:
    """
    Delete container with full cleanup
    
    Handles stop, snapshot removal, HA resource deletion,
    and final container destruction
    """
    print(f"[INFO] Deleting container {vmid} ({subdomain})")
    
    # Find hosting node
    node = proxmox_client.find_container_node(vmid)
    if not node:
        raise Exception(f"Container {vmid} not found")
    
    # Stop container (graceful then forced)
    self._stop_container_safely(node, vmid)
    
    # Remove snapshots
    self._delete_snapshots(node, vmid)
    
    # Remove HA resources
    self._remove_ha_resource(vmid)
    
    # Delete container
    for attempt in range(3):
        try:
            proxmox_client.destroy_container(node, vmid)
            print(f"[SUCCESS] Container {vmid} deleted")
            return True
        except Exception as e:
            if attempt < 2:
                print(f"[INFO] Retry deletion in 5s...")
                time.sleep(5)
            else:
                raise
```

### Graceful Shutdown Logic

```python
def _stop_container_safely(self, node, vmid):
    """
    Attempt graceful shutdown before forcing stop
    """
    status = proxmox_client.get_container_status(node, vmid)
    
    if status.get("status") != "running":
        return  # Already stopped
    
    # Try graceful shutdown
    try:
        print(f"[INFO] Graceful shutdown CT{vmid}")
        proxmox_client.shutdown_container(node, vmid)
        
        # Wait for shutdown
        for _ in range(6):
            time.sleep(5)
            status = proxmox_client.get_container_status(node, vmid)
            if status.get("status") != "running":
                print(f"[SUCCESS] Container {vmid} stopped")
                return
    except Exception as e:
        print(f"[WARN] Graceful shutdown failed: {e}")
    
    # Force stop if graceful failed
    print(f"[INFO] Force stopping CT{vmid}")
    proxmox_client.stop_container(node, vmid)
    time.sleep(5)
```

## IP Allocation

### Next Available IP

```python
def next_available_ip() -> str:
    """
    Find next available IP in private network range
    
    Scans 192.168.1.100-253 and returns first unused IP
    """
    used_ips = set()
    
    # Get all containers across cluster
    for node in ["node1", "node2"]:
        containers = proxmox_client.get_container_list(node)
        for container in containers:
            config = proxmox_client.get_container_config(node, container["vmid"])
            if "net0" in config:
                # Extract IP from net0 config
                ip_match = re.search(r'ip=(\d+\.\d+\.\d+\.\d+)', config["net0"])
                if ip_match:
                    used_ips.add(ip_match.group(1))
    
    # Find first available IP in range
    for i in range(100, 254):
        candidate = f"192.168.1.{i}"
        if candidate not in used_ips:
            return candidate
    
    raise Exception("No available IPs in range")
```

## Error Handling Patterns

### Retry with Exponential Backoff

```python
def retry_operation(func, max_attempts=3, backoff=2):
    """Generic retry wrapper for API operations"""
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception as e:
            if attempt < max_attempts - 1:
                wait = backoff ** attempt
                print(f"[INFO] Retry in {wait}s...")
                time.sleep(wait)
            else:
                raise
```

### Resource Cleanup on Failure

```python
def provision_with_rollback(service_id, email, plan):
    """
    Provision container with automatic rollback on failure
    """
    created_resources = []
    
    try:
        # Database entry
        add_site(service_id, email, plan, ...)
        created_resources.append(("db", service_id))
        
        # Container
        vmid = create_container(...)
        created_resources.append(("container", vmid))
        
        # DNS
        dns_id = create_dns_record(...)
        created_resources.append(("dns", dns_id))
        
        # Proxy
        proxy_id = create_proxy(...)
        created_resources.append(("proxy", proxy_id))
        
        return {"success": True, "vmid": vmid}
        
    except Exception as e:
        # Rollback in reverse order
        print(f"[ERROR] Provisioning failed: {e}")
        for resource_type, resource_id in reversed(created_resources):
            cleanup_resource(resource_type, resource_id)
        raise
```

## Key Design Patterns

**Separation of Concerns:**
- Container manager handles lifecycle
- Proxmox client handles API communication
- Database tracks state
- Functions handle specific operations

**Idempotency:**
- Operations can be safely retried
- Duplicate executions have no effect
- State checks before destructive actions

**Error Visibility:**
- Detailed logging at each step
- Stack traces for exceptions
- Success confirmation messages

**Resource Safety:**
- Unprivileged containers
- Resource limits enforced
- Network isolation
- Cleanup on deletion

## Cloud Translation

This pattern directly translates to cloud container services:

**AWS:**
```python
# Similar pattern with ECS/Fargate
ecs_client.create_task_definition(...)
ecs_client.run_task(...)
```

**Azure:**
```python
# Similar pattern with Container Instances
container_client.create_container_group(...)
```

**Key Difference:** Cloud services abstract away the host node, but orchestration logic remains identical.

## Key Takeaways

1. **Production-Grade Error Handling** - Retry logic, rollback capability
2. **Infrastructure as Code** - Containers defined programmatically
3. **HA-Aware** - Automatic failover configuration
4. **Security-Focused** - Unprivileged containers, network isolation
5. **Observable** - Comprehensive logging and status tracking

This implementation demonstrates:
- Container lifecycle management
- API-driven infrastructure provisioning
- Error handling and resilience patterns
- Production operations mindset
- Cloud-equivalent design patterns

---

## See Also

- **[Python Backend Architecture](../docs/provisioning/python-backend.md)** - Complete API design
- **[Automated Workflow](../docs/provisioning/automated_workflow.md)** - How this code fits in the pipeline
- **[Proxmox Infrastructure](../docs/infrastructure/proxmox.md)** - Cluster configuration details

---

*This code represents real production infrastructure management, not tutorial examples—handling failures, cleanup, and edge cases that arise in live environments.*
