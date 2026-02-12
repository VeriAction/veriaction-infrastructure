# VeriAction Infrastructure Automation

Production-ready Ansible automation for deploying the complete VeriAction ecosystem with GDPR compliance, multi-region support, and zero-downtime deployments.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Deployment Guides](#deployment-guides)
- [GDPR Compliance](#gdpr-compliance)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

## 🎯 Overview

This repository contains Ansible automation for deploying and managing:

- **CockroachDB Clusters**: Multi-node distributed SQL database (EU + Global regions)
- **Kubernetes Clusters**: Self-hosted K8s clusters via kubeadm
- **HAProxy Load Balancers**: High-availability gRPC load balancing
- **GeoIP Service**: VeriAction's IP intelligence microservice
- **Monitoring Stack**: Prometheus + Grafana + Alertmanager on dedicated VMs
- **MaxMind Hub Service**: Central data distribution (future - Issue #13)

### Key Features

✅ **GDPR Compliant**: Separate EU and Global deployments with data isolation
✅ **Zero-Downtime**: Rolling updates with health checks and automated rollback
✅ **Production Hardened**: Security best practices, TLS everywhere, secret rotation
✅ **Fully Idempotent**: Safe to run multiple times
✅ **Multi-Region**: EU and Global infrastructure with proper data residency
✅ **Comprehensive Monitoring**: Pre-configured dashboards and alerting

## 🏗️ Architecture

### Regional Deployment Model

```
┌─────────────────────────────────────────────────────────────┐
│                      EU REGION (GDPR)                       │
├─────────────────────────────────────────────────────────────┤
│  HAProxy (HA) → Kubernetes → GeoIP Service (3-5 replicas)  │
│                          ↓                                   │
│                  CockroachDB Cluster                        │
│                   (3-5 nodes, EU only)                      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    GLOBAL REGION                            │
├─────────────────────────────────────────────────────────────┤
│  Cloud LB → Kubernetes → GeoIP Service (3-5 replicas)      │
│                          ↓                                   │
│                  CockroachDB Cluster                        │
│                   (3-5 nodes, global)                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              MONITORING (Dedicated VMs)                     │
├─────────────────────────────────────────────────────────────┤
│  Prometheus + Grafana + Alertmanager                        │
│  (Monitors all regions)                                     │
└─────────────────────────────────────────────────────────────┘
```

### Component Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Database** | CockroachDB 23.x | Distributed SQL, audit logs, config storage |
| **Orchestration** | Kubernetes 1.28+ | Container orchestration |
| **Load Balancer** | HAProxy 2.8+ / Cloud LB | gRPC load balancing, HA |
| **Application** | VeriAction GeoIP | IP intelligence service |
| **Monitoring** | Prometheus + Grafana | Metrics, dashboards, alerting |
| **Automation** | Ansible 2.15+ | Infrastructure provisioning |

## 📦 Prerequisites

### Control Node (Your Laptop/Bastion)

```bash
# Required software
ansible >= 2.15
python >= 3.9
kubectl >= 1.28
docker >= 24.0 (for image builds)
git

# Install Ansible
pip3 install ansible

# Install required Ansible collections
ansible-galaxy collection install kubernetes.core
ansible-galaxy collection install community.general
ansible-galaxy collection install community.crypto
```

See [docs/PREREQUISITES.md](docs/PREREQUISITES.md) for complete setup instructions.

## 🚀 Quick Start

```bash
# 1. Clone repository
git clone https://github.com/VeriAction/veriaction-infrastructure.git
cd veriaction-infrastructure

# 2. Configure inventory
cp inventory/production.example inventory/production
vim inventory/production

# 3. Configure secrets
ansible-vault create group_vars/vault.yml

# 4. Deploy everything
ansible-playbook playbooks/site.yml
```

See [Quick Start Guide](docs/QUICKSTART.md) for detailed instructions.

## 📚 Documentation

- **[CockroachDB Setup](docs/COCKROACHDB.md)** - Multi-node cluster deployment
- **[Kubernetes Setup](docs/KUBERNETES.md)** - Cluster bootstrapping
- **[HAProxy Setup](docs/HAPROXY.md)** - Load balancer configuration
- **[GeoIP Deployment](docs/DEPLOYMENT.md)** - Service deployment
- **[Monitoring Stack](docs/MONITORING.md)** - Prometheus + Grafana
- **[GDPR Compliance](docs/GDPR-COMPLIANCE.md)** - Regional isolation
- **[Troubleshooting](docs/TROUBLESHOOTING.md)** - Common issues

## 🔒 GDPR Compliance

Strict data residency enforcement:
- EU data never leaves EU region
- Separate CockroachDB clusters per region
- Regional Kubernetes clusters
- Compliance verification playbooks

See [GDPR Compliance Guide](docs/GDPR-COMPLIANCE.md).

## 📖 Directory Structure

```
veriaction-infrastructure/
├── ansible.cfg              # Ansible configuration
├── inventory/               # Server inventories
├── group_vars/             # Group variables
├── roles/                  # Ansible roles
├── playbooks/              # Deployment playbooks
├── docs/                   # Documentation
├── scripts/                # Helper scripts
└── templates/              # Configuration templates
```

## 📄 License

Internal VeriAction Infrastructure - Proprietary

---

**Maintained by**: VeriAction DevOps Team
