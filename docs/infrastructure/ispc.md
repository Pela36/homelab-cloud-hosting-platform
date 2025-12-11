# ISPConfig Infrastructure Services

## Overview

**Server**: ispc.nodepoint.eu  
**Management IP**: 192.168.1.56 (LAN)  
**ISPConfig Version**: 3.3.0p3  
**Operating System**: Ubuntu 24.04.3 LTS (Noble Numbat)  
**Uptime**: 1 week+ (highly stable)

ISPConfig serves as the centralized control panel for DNS, email, and web hosting services. It provides API-driven management of these services, enabling automated customer provisioning and domain management.

## Network Configuration

### Multi-IP Architecture

The ISPConfig server uses **5 separate network interfaces** with dedicated public IPs for service isolation:

| Interface | IP Address | Purpose | Services |
|-----------|-----------|---------|----------|
| **eth0** | 203.0.113.11 | Primary Web/FTP | HTTP, HTTPS, FTP |
| **eth1** | 192.168.1.56 | Management (LAN) | SSH, ISPConfig Panel |
| **eth2** | 203.0.113.12 | Mail Server | SMTP, IMAP, POP3 |
| **eth3** | 203.0.113.13 | DNS Primary | ns1.nodepoint.eu |
| **eth4** | 203.0.113.14 | DNS Secondary | ns2.nodepoint.eu |

**Gateway**: 203.0.113.1 (OpenWrt router DMZ interface)  
**DNS Resolvers**: 1.1.1.1 (Cloudflare), 192.168.1.1 (Local)

### Why Multiple IPs?

**Service Isolation:**
- Separate IPs for different services improves troubleshooting
- Mail server on dedicated IP improves email deliverability
- DNS servers on separate IPs ensures redundancy

**Traffic Monitoring:**
- Can track bandwidth per service
- Cloud provider limits (500GB/month per IP)
- Identify which service is consuming resources

**IP Reputation:**
- Mail server IP (203.0.113.12) maintains separate reputation
- If web server IP gets blacklisted, mail delivery unaffected
- DNS servers on clean IPs prevent DNS blocking

## DNS Services (BIND9)

### Configuration

**Service**: BIND9 (named)  
**Status**: Active and running (1 week+ uptime)  
**Nameservers**:
- ns1.nodepoint.eu → 203.0.113.13 (eth3)
- ns2.nodepoint.eu → 203.0.113.14 (eth4)

**Zone Management:**
- Zones stored in: `/etc/bind/`
- Configuration: `/etc/bind/named.conf.local`
- Managed via ISPConfig API

### Functionality

**Authoritative DNS:**
- Hosts DNS zones for customer domains
- A records, CNAME, MX, TXT records
- Subdomain management (*.nodepoint.eu)

**Automated Record Management:**
- Python backend creates DNS records via ISPConfig API
- Customer provisioning triggers DNS entry creation
- Custom domain support through API

**Example Zone:**
```
Zone: nodepoint.cloud
Type: master
File: /etc/bind/pri.nodepoint.cloud
```

### Integration with Platform

When a WordPress site is provisioned:
1. Python API calls ISPConfig to create DNS record
2. A record points subdomain.nodepoint.eu → NPM IP (203.0.113.10)
3. Customer domain records added via ISPConfig panel
4. DNS propagates within minutes

## Email Services (Postfix + Dovecot)

### Mail Server Configuration

**SMTP (Postfix)**: Outbound and inbound mail  
**IMAP/POP3 (Dovecot)**: Mailbox access  
**Hostname**: mail.nodepoint.eu  
**Domain**: nodepoint.eu  
**Mail IP**: 203.0.113.12 (eth2)

### Postfix Setup

**Key Configuration:**
```
myhostname = mail.nodepoint.eu
mydomain = nodepoint.eu
inet_interfaces = all (listens on all IPs)
mynetworks = 127.0.0.0/8 [::1]/128 (local only)
```

**Security Features:**
- SASL authentication required
- MySQL-backed virtual users
- Sender/recipient verification
- Greylisting for spam protection
- Quota enforcement
- TLS encryption

**Anti-Spam Measures:**
- Helo checks with blacklist
- Sender verification against database
- Recipient verification
- Greylisting policy
- Rate limiting

### Dovecot IMAP/POP3

**Service**: Active and running  
**Purpose**: Mailbox access for email clients  
**Protocols**: IMAP, IMAPS, POP3, POP3S

**Features:**
- Virtual mailboxes (MySQL backend)
- Secure authentication
- TLS/SSL support
- Quota management

### Email Deliverability

**SPF, DKIM, DMARC Configuration:**
- SPF records authorize mail server IP
- DKIM signs outgoing messages
- DMARC policy prevents spoofing
- PTR record: 203.0.113.12 → mail.nodepoint.eu

**Result:** Email reaches Gmail/Outlook inbox with high deliverability.

## FTP Services (Pure-FTPd)

**Service**: pure-ftpd-mysql  
**Status**: Active and running  
**Backend**: MySQL virtual users  
**Purpose**: Customer file access (optional service)

**Configuration:**
- MySQL-backed user authentication
- Virtual users (no system accounts)
- Chrooted users (restricted to home directory)
- TLS/SSL support

## ISPConfig Control Panel

### System Configuration

**Database:**
- Type: MySQL
- Host: localhost
- Database: dbispconfig
- User: ispconfig
- Charset: utf8

**Enabled Modules:**
- Dashboard: System overview
- Mail: Email management
- Sites: Web hosting and domains
- DNS: Zone and record management
- Tools: System utilities
- Help: Documentation

**Web Interface:**
- Accessible via: https://192.168.1.56:8081 (LAN only)
- Authentication: PAM or ISPConfig users
- API endpoint: /remote/

### API Integration

The platform's Python backend integrates with ISPConfig REST API for:

**DNS Operations:**
- Create/update/delete DNS zones
- Manage A, CNAME, MX, TXT records
- Subdomain automation

**Email Operations:**
- Create mailboxes for custom domains
- Manage email aliases
- Configure forwarding rules

**Domain Management:**
- Add customer domains
- Configure DNS zones
- SSL certificate management

## Service Architecture

### Service Distribution Across IPs

```
Customer Request Flow:

1. Web Traffic (HTTP/HTTPS):
   Customer → 203.0.113.10 (NPM) → WordPress Container
   
2. Email (SMTP):
   Sender → 203.0.113.12 (Postfix) → Deliver to mailbox
   
3. Email Access (IMAP):
   Email Client → 203.0.113.12 (Dovecot) → Fetch messages
   
4. DNS Query:
   Resolver → 203.0.113.13 or .14 (BIND9) → Return records
   
5. Management:
   Admin → 192.168.1.56 (ISPConfig panel) → Configure services
```

### Backend Communication

**Internal Services:**
- All services communicate via localhost or 192.168.1.56
- No direct external access to management interfaces
- ISPConfig panel accessible only from LAN

**External Services:**
- DNS (port 53): 203.0.113.13, 203.0.113.14
- SMTP (port 25, 587): 203.0.113.12
- IMAP/POP3 (ports 143, 993, 110, 995): 203.0.113.12
- Web/FTP: 203.0.113.11

## Operational Details

### Service Status

**Active Services:**
- ✅ BIND9 (named): Running, 1 week uptime
- ✅ Postfix: Running, processing mail
- ✅ Dovecot: Running, serving mailboxes
- ✅ Pure-FTPd: Running, MySQL backend
- ❌ Apache2: Disabled (not used for customer sites)

**Why Apache2 is Disabled:**
- Customer WordPress sites use their own Apache in LXC containers
- ISPConfig panel served via different mechanism
- Reduces attack surface on central server

### Database Backend

ISPConfig uses MySQL for:
- User authentication (virtual users)
- Email accounts and aliases
- DNS zones and records
- Configuration storage
- Logging and monitoring

### Logging

**Log Directory**: `/var/log/ispconfig/`  
**Log Priority**: Error level (2)  
**Max Message Length**: 32KB

## Automation Workflow

### Customer Provisioning Integration

When Python backend provisions a WordPress site:

**1. DNS Record Creation:**
```python
# API call to ISPConfig
ispconfig.dns_zone_create(
    server_id=1,
    origin='subdomain.nodepoint.eu',
    ns='ns1.nodepoint.eu',
    mbox='admin@nodepoint.eu'
)

ispconfig.dns_a_record_create(
    zone_id=zone_id,
    name='subdomain',
    type='A',
    data='203.0.113.10'  # NPM reverse proxy IP
)
```

**2. Custom Domain Setup:**
- Customer adds custom domain
- ISPConfig creates DNS zone
- Configures nameserver records
- Adds MX records if email needed

**3. Email Configuration:**
- Optionally create mailboxes for custom domains
- Configure DKIM keys
- Set up SPF/DMARC records

## Comparison to Cloud Services

| ISPConfig Feature | AWS Equivalent | Azure Equivalent |
|-------------------|----------------|------------------|
| BIND9 DNS | Route53 | Azure DNS |
| Postfix/Dovecot | Amazon SES + WorkMail | Exchange Online |
| ISPConfig API | AWS API Gateway | Azure API Management |
| Multi-IP setup | Elastic IPs | Public IP addresses |
| Virtual users (MySQL) | IAM users | Azure AD users |
| DNS zones | Hosted zones | DNS zones |

## Security Considerations

**Network Security:**
- Management interface (192.168.1.56) only accessible from LAN
- Public services on dedicated IPs
- Firewall rules restrict access to necessary ports

**Email Security:**
- SASL authentication required
- TLS encryption enforced
- SPF/DKIM/DMARC configured
- Greylisting reduces spam
- Rate limiting prevents abuse

**DNS Security:**
- DNSSEC not enabled (optional)
- Zone transfers restricted
- Query rate limiting

**Service Isolation:**
- Each service on separate IP
- Virtual users (no system accounts)
- Chrooted FTP users
- Database-backed authentication

## Monitoring and Maintenance

**Service Monitoring:**
- Systemctl status checks
- ISPConfig dashboard monitoring
- Mail queue monitoring
- DNS query logging

**Maintenance Tasks:**
- Regular backups of ISPConfig database
- Log rotation
- Security updates (Ubuntu)
- ISPConfig updates

## Key Takeaways

1. **Multi-IP Design**: 5 interfaces provide service isolation and traffic management
2. **Centralized Management**: ISPConfig provides unified control of DNS, email, FTP
3. **API-Driven**: Full REST API enables automation and integration
4. **Production Email**: Proper SPF/DKIM/DMARC achieves inbox deliverability
5. **Authoritative DNS**: ns1/ns2.nodepoint.eu serve customer domains
6. **MySQL Backend**: Virtual users and configuration stored in database
7. **Security Focus**: Network isolation, authentication, encryption

This infrastructure demonstrates:
- Multi-service management with single control panel
- API integration for automation
- Proper email infrastructure (not just relay)
- DNS server administration
- Network architecture (multiple IPs, service isolation)
- Understanding of mail delivery and reputation

---

*ISPConfig acts as the backbone for DNS and email services, providing API-driven management that integrates seamlessly with the automated provisioning system.*
