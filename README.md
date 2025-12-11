# Multi-Tenant Automated Cloud Hosting Platform

> **Live Infrastructure**: nodepoint.eu | **Contact**: matej.pelusic@nodepoint.eu

A production-grade homelab infrastructure demonstrating cloud engineering skills through real-world implementation. This project showcases high availability, distributed systems, automation, and infrastructure orchestration without requiring cloud provider accounts.

## 🎯 Project Overview

Fully automated WordPress hosting platform built on Proxmox infrastructure with:
- **Zero-touch provisioning** (payment → live website in 3-5 minutes)
- **High availability** (2-node cluster with automatic failover in ~2 minutes)
- **Distributed storage** (Linstor for real-time replication)
- **Self-hosted services** (DNS servers, mail server, reverse proxy)
- **API-driven management** (Python backend orchestrating all services)

## 🏗️ Architecture Highlights

```
Customer Payment (Paymenter)
    ↓
Python API Backend
    ↓
Proxmox Cluster (2 nodes + router)
    ├─→ LXC Container Creation (on Linstor distributed storage)
    ├─→ HA Configuration (automatic failover)
    ├─→ DNS Record (Cloudflare API)
    ├─→ Reverse Proxy (Nginx Proxy Manager)
    └─→ WordPress Installation
    ↓
Live Website: https://<subdomain>.nodepoint.eu
```

## 💡 Key Technical Achievements

### 1. High Availability with Fast Failover
- **Problem**: Standard HA requires volume transfers (10-30 min downtime)
- **Solution**: Linstor distributed storage (2-minute failover)
- **Result**: Tested with actual node failures, zero data loss

### 2. Network Architecture
- WireGuard VPN tunnel handling dynamic home IP
- Routed /27 public subnet
- Three-network design (WAN, Private, Public)
- Proper segmentation and firewall rules

### 3. Complete Automation
- Python API orchestrating 6+ services
- Async operations for concurrent provisioning
- Error handling with automatic rollback
- Full audit trail logging

### 4. Self-Hosted Infrastructure
- Authoritative DNS (ns1/ns2.nodepoint.eu)
- Mail server with inbox deliverability (SPF/DKIM/DMARC)
- Multi-IP traffic segregation (4 public IPs)

## 📚 Documentation Structure

### **Getting Started**
- **[QUICK_START.md](QUICK_START.md)** - For employers/recruiters (read this first!)
- **[README.md](README.md)** - This file (project overview)

### **Architecture** (`/docs/architecture/`)
- **[HA.md](docs/architecture/HA.md)** - 2-node cluster design and failover testing
- **[network_topology.md](docs/architecture/network_topology.md)** - VPN tunneling, routing, segmentation

### **Provisioning** (`/docs/provisioning/`)
- **[automated_workflow.md](docs/provisioning/automated_workflow.md)** - Complete zero-touch provisioning pipeline
- **[python-backend.md](docs/provisioning/python-backend.md)** - API architecture and orchestration

### **Infrastructure** (`/docs/infrastructure/`)
- **[proxmox.md](docs/infrastructure/proxmox.md)** - Proxmox cluster configuration and VM details
- **[ispc.md](docs/infrastructure/ispc.md)** - ISPConfig setup, DNS/email configuration
- **[paymenter.md](docs/infrastructure/paymenter.md)** - Billing system integration and customer portal

### **Problem Solving** (`/docs/problems-solved/`)
- **[storage-redundancy.md](docs/problems-solved/storage-redundancy.md)** - Case study: Solving HA storage bottleneck

### **Code Samples** (`/code-samples/`)
- **[lxc.md](code-samples/lxc.md)** - Production-quality container provisioning code
- **[extension-proxmoxwp.md](code-samples/extension-proxmoxwp.md)** - Paymenter extension for WordPress hosting
- **[extension-dns.md](code-samples/extension-dns.md)** - Paymenter extension for DNS management
- **[extension-ssupport.md](code-samples/extension-ssupport.md)** - Paymenter extension for support subscriptions

## 🔧 Technology Stack

**Infrastructure:**
- Proxmox VE (3 nodes: 1 router + 2 hosting)
- Linstor distributed storage (2x2TB mirrored per node)
- LXC containers with HA
- OpenWrt virtual router (PPPoE)

**Networking:**
- WireGuard VPN (Cloud Transit Provider)
- Public subnet: 203.0.113.0/27
- Private network: 192.168.1.0/24
- Nginx Proxy Manager

**Services:**
- ISPConfig (DNS + Email)
- Paymenter (billing/customer portal)
- Python Flask API backend
- Cloudflare API integration

**Automation:**
- Python asyncio for concurrent operations
- Proxmox API
- Cloudflare API
- Custom Paymenter extensions

## 🎓 Cloud Skills Demonstrated

| Homelab Technology | Cloud Equivalent | Skill Demonstrated |
|-------------------|------------------|-------------------|
| Linstor distributed storage | AWS EBS multi-attach, Azure Shared Disks | Distributed systems, data replication |
| Proxmox HA cluster | AWS Auto Scaling Groups, Azure Availability Sets | High availability, fault tolerance |
| LXC automation | AWS ECS/EKS, Azure Container Instances | Container orchestration |
| Python API backend | AWS Lambda + API Gateway, Azure Functions | Infrastructure as Code, automation |
| WireGuard VPN | AWS VPN Gateway, Azure VPN Gateway | Networking, tunneling protocols |
| Network segmentation | VPC design, Security Groups, NSGs | Network architecture, security |
| ISPConfig DNS | Route53, Azure DNS, Cloud DNS | DNS administration, API integration |
| Nginx Proxy Manager | Application Load Balancer, Azure App Gateway | Reverse proxy, SSL/TLS automation |
| Multi-tenant isolation | IAM policies, Resource tagging | Security boundaries, access control |

## 📊 Performance Metrics

- **Provisioning time**: 3-5 minutes (fully automated)
- **Failover time**: ~2 minutes (vs 30+ without Linstor)
- **Uptime**: 99.9%+ (tested with real node failures)
- **Data loss on failure**: Zero (real-time replication)
- **Write performance overhead**: ~15% (acceptable for reliability)
- **Read performance overhead**: 0% (reads are local)

## 🔍 What Makes This Different

### Not Just Following Tutorials
- Identified problems before building solutions
- Researched multiple approaches (Ceph, GlusterFS, DRBD)
- Made architectural decisions based on requirements
- Tested thoroughly with actual failures

### Production-Grade Thinking
- HA tested by literally pulling power plugs
- Monitoring and backup strategy
- Customer self-service portal
- Security boundaries and resource limits
- Error handling and rollback mechanisms

### Complete Stack Ownership
- Physical hardware setup
- Network configuration from scratch
- Built automation layer
- Integrated billing system
- Customer-facing portal

### Cost Consciousness
- ~$2000 total investment vs $500+/month cloud
- Evaluated solutions on cost/benefit
- Resource utilization optimization

## 💼 For Employers

This project demonstrates:

✅ **Distributed Systems** - Real-time storage replication, consistency guarantees  
✅ **High Availability** - Multi-node clusters, automatic failover  
✅ **Networking** - VPN tunneling, routing, segmentation, firewall rules  
✅ **Automation** - API-driven infrastructure, zero-touch provisioning  
✅ **Problem Solving** - Identified bottlenecks, researched solutions, validated with testing  
✅ **Production Operations** - Monitoring, backups, security, customer management  
✅ **Code Quality** - Error handling, async operations, rollback capability  
✅ **Documentation** - Technical writing, system explanation

**Interview talking points**: Every aspect has detailed documentation with concrete examples and testing results.

## 🚀 Additional Services

- **VPS Reselling**: Hetzner API integration
- **Remote Support Subscriptions**: Monthly technical assistance service
- **Custom Domain Management**: Full DNS control through customer portal
- **Automated Backups**: Snapshot-based rollback capability

## 📈 Future Development

- AI model integration for automated webapp generation
- Zabbix monitoring with Telegram bot notifications
- Enhanced analytics dashboard
- Additional service offerings

## 📞 Contact

- **Email**: matej.pelusic@nodepoint.eu
- **Website**: nodepoint.eu
- **DNS**: ns1.nodepoint.eu, ns2.nodepoint.eu
- **Location**: European Union

---

**Note on AI Assistance**: While AI tools assisted with code generation, I understand the architecture, can troubleshoot issues, and modify the code as needed. The system design, problem-solving, and infrastructure decisions are my own.

---

*This infrastructure represents 200+ hours of research, implementation, and testing. It demonstrates the thinking and skills that separate tutorial-followers from engineers who can design production systems.*
