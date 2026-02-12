# CockroachDB Cluster Deployment Guide

## Overview

This guide covers deploying production-grade, multi-node CockroachDB clusters for the VeriAction platform. CockroachDB stores audit logs, user settings, organization configurations, and operational metadata with GDPR compliance.

## Architecture

### Regional Deployment

- **EU Region**: Dedicated 3-5 node cluster (data never leaves EU)
- **Global Region**: Separate 3-5 node cluster for non-EU data
- **Replication Factor**: 3x (survives 1 node failure)
- **Data Isolation**: Zero cross-region replication

### Hardware Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8 cores |
| RAM | 16 GB | 32 GB |
| Storage | 500 GB SSD | 1 TB NVMe SSD |
| Network | 1 Gbps | 10 Gbps |

## Prerequisites

### Operating System
- Ubuntu 22.04 LTS (recommended)
- Debian 11/12
- Rocky Linux 9 / AlmaLinux 9

### Network
- Static IP addresses for all nodes
- Ports 26257 (CockroachDB) and 8080 (HTTP) open between cluster nodes
- DNS resolution between nodes (or /etc/hosts entries)

### User Setup
```bash
# Run on each CockroachDB node
sudo useradd -m -s /bin/bash cockroach
sudo mkdir -p /var/lib/cockroach
sudo chown cockroach:cockroach /var/lib/cockroach
```

## Deployment Steps

### 1. Configure Inventory

Edit `inventory/production` and add your CockroachDB nodes:

```ini
[eu_cockroachdb]
eu-crdb-01 ansible_host=10.0.1.10 cockroachdb_node_id=1
eu-crdb-02 ansible_host=10.0.1.11 cockroachdb_node_id=2
eu-crdb-03 ansible_host=10.0.1.12 cockroachdb_node_id=3

[global_cockroachdb]
global-crdb-01 ansible_host=10.0.2.10 cockroachdb_node_id=1
global-crdb-02 ansible_host=10.0.2.11 cockroachdb_node_id=2
global-crdb-03 ansible_host=10.0.2.12 cockroachdb_node_id=3
```

### 2. Configure Variables

Review and customize `group_vars/cockroachdb.yml`:

```yaml
cockroachdb_version: "23.2.3"
cockroachdb_cache_size: "25%"
cockroachdb_max_sql_memory: "25%"
cockroachdb_default_replication_factor: 3
```

### 3. Set Passwords in Vault

```bash
ansible-vault edit group_vars/vault.yml
```

Add:
```yaml
vault_cockroachdb_app_password: "your-strong-password"
vault_cockroachdb_readonly_password: "your-strong-password"
vault_cockroachdb_backup_password: "your-strong-password"
```

### 4. Deploy EU Cluster

```bash
# Deploy EU CockroachDB cluster
ansible-playbook playbooks/cockroachdb-cluster.yml --limit eu_cockroachdb
```

### 5. Deploy Global Cluster

```bash
# Deploy Global CockroachDB cluster
ansible-playbook playbooks/cockroachdb-cluster.yml --limit global_cockroachdb
```

### 6. Verify Deployment

```bash
# Check cluster status (run on any node)
cockroach node status --certs-dir=/var/lib/cockroach/certs --host=localhost:26257
```

Expected output:
```
  id |    address     |   sql_address  |  build  |    started_at    | updated_at | locality | is_available | is_live
-----+----------------+----------------+---------+------------------+------------+----------+--------------+---------
   1 | eu-crdb-01:... | eu-crdb-01:... | v23.2.3 | 2026-02-12 ...   | 2026-02...  | ...      | true         | true
   2 | eu-crdb-02:... | eu-crdb-02:... | v23.2.3 | 2026-02-12 ...   | 2026-02...  | ...      | true         | true
   3 | eu-crdb-03:... | eu-crdb-03:... | v23.2.3 | 2026-02-12 ...   | 2026-02...  | ...      | true         | true
```

## Post-Deployment Configuration

### Create Databases

Databases are automatically created by the playbook:
- `veriaction` - Main application database
- `audit` - Audit logs (GDPR compliant)
- `config` - Configuration storage

### Create Users

Users are automatically created with appropriate permissions:
- `veriaction_app` - Full access (read/write)
- `veriaction_readonly` - Read-only access
- `backup_user` - Backup privileges only

### Access Database

```bash
# Connect as root
cockroach sql --certs-dir=/var/lib/cockroach/certs --host=localhost:26257

# Connect as app user
cockroach sql --certs-dir=/var/lib/cockroach/certs --host=localhost:26257 --user=veriaction_app --database=veriaction
```

## Backup and Restore

### Automated Backups

Backups are automatically configured:
- **Full backup**: Weekly (Sundays at 2 AM)
- **Incremental backup**: Daily (Mon-Sat at 2 AM)
- **Retention**: 7 days
- **Destination**: S3-compatible storage

### Manual Backup

```bash
# Full backup
cockroach sql --certs-dir=/var/lib/cockroach/certs --execute="BACKUP INTO 's3://your-bucket/manual-backup?AWS_ACCESS_KEY_ID=xxx&AWS_SECRET_ACCESS_KEY=yyy';"
```

### Restore from Backup

```bash
# List available backups
cockroach sql --certs-dir=/var/lib/cockroach/certs --execute="SHOW BACKUPS IN 's3://your-bucket/full';"

# Restore latest backup
cockroach sql --certs-dir=/var/lib/cockroach/certs --execute="RESTORE FROM LATEST IN 's3://your-bucket/full';"
```

## Monitoring

### Admin UI

Access the CockroachDB Admin UI:
```
https://<any-node-ip>:8080
```

### Prometheus Metrics

Metrics are exposed at:
```
http://<node-ip>:8080/_status/vars
```

Prometheus automatically scrapes these endpoints.

### Key Metrics to Monitor

- **Node availability**: `is_live` and `is_available`
- **Replication lag**: `ranges.unavailable`
- **Query latency**: `sql.exec.latency`
- **Disk usage**: `capacity.available`
- **Connection count**: `sql.conns`

## Maintenance Operations

### Rolling Restart

```bash
# Drain node before restart
ansible-playbook playbooks/maintenance/cockroachdb-drain-node.yml --limit eu-crdb-01

# Restart node
ansible-playbook playbooks/maintenance/cockroachdb-restart-node.yml --limit eu-crdb-01
```

### Scale Cluster

To add a 4th node:

1. Add to inventory:
```ini
eu-crdb-04 ansible_host=10.0.1.13 cockroachdb_node_id=4
```

2. Deploy to new node:
```bash
ansible-playbook playbooks/cockroachdb-cluster.yml --limit eu-crdb-04
```

3. Verify new node joined:
```bash
cockroach node status --certs-dir=/var/lib/cockroach/certs --host=localhost:26257
```

### Decommission Node

```bash
# Mark node for decommissioning
cockroach node decommission 4 --certs-dir=/var/lib/cockroach/certs --host=localhost:26257

# Monitor decommissioning progress
cockroach node status --certs-dir=/var/lib/cockroach/certs --host=localhost:26257
```

## Security

### TLS Certificates

Certificates are automatically generated and managed by Ansible:
- **CA Certificate**: `/var/lib/cockroach/certs/ca.crt`
- **Node Certificate**: `/var/lib/cockroach/certs/node.crt`
- **Client Certificate**: `/var/lib/cockroach/certs/client.root.crt`

### Certificate Rotation

```bash
# Rotate certificates annually
ansible-playbook playbooks/maintenance/rotate-certificates.yml --limit cockroachdb
```

### Firewall Rules

Required ports:
- **26257**: CockroachDB inter-node communication and SQL
- **8080**: HTTP API and Admin UI

```bash
# UFW example
sudo ufw allow from 10.0.1.0/24 to any port 26257
sudo ufw allow from 10.0.1.0/24 to any port 8080
```

## Troubleshooting

### Node Not Joining Cluster

1. Check connectivity:
```bash
telnet <other-node-ip> 26257
```

2. Check logs:
```bash
sudo journalctl -u cockroachdb -f
```

3. Verify certificates:
```bash
ls -la /var/lib/cockroach/certs/
```

### Cluster Stuck

If cluster won't initialize:
```bash
# On first node only
cockroach init --certs-dir=/var/lib/cockroach/certs --host=localhost:26257
```

### Performance Issues

1. Check ranges:
```bash
cockroach sql --execute="SELECT * FROM crdb_internal.ranges_no_leases WHERE database_name='veriaction';"
```

2. Check slow queries:
```bash
cockroach sql --execute="SELECT * FROM crdb_internal.node_statement_statistics ORDER BY service_lat DESC LIMIT 10;"
```

## GDPR Compliance

### Data Residency

- EU cluster stores ONLY EU customer data
- Global cluster stores non-EU customer data
- NO replication between regions
- Audit logs stored in same region as source data

### Verification

```bash
# Verify zone constraints
cockroach sql --execute="SHOW ZONE CONFIGURATION FOR DATABASE veriaction;"
```

### Data Subject Access

```bash
# Export user data for GDPR request
cockroach sql --execute="SELECT * FROM veriaction.user_data WHERE user_id='<user-id>';" --format=csv > user_data.csv
```

## Performance Tuning

### Recommended Settings

```sql
-- Optimize for SSDs
SET CLUSTER SETTING kv.range_merge.queue_enabled = true;

-- Connection pooling
SET CLUSTER SETTING server.max_connections_per_gateway = 500;

-- Query timeouts
SET CLUSTER SETTING sql.defaults.statement_timeout = '60s';
SET CLUSTER SETTING sql.defaults.idle_in_transaction_session_timeout = '10m';
```

### Benchmarking

```bash
# Run TPC-C benchmark
cockroach workload init tpcc 'postgresql://veriaction_app@localhost:26257/veriaction?sslmode=verify-full&sslrootcert=/var/lib/cockroach/certs/ca.crt'

cockroach workload run tpcc --duration=5m 'postgresql://veriaction_app@localhost:26257/veriaction?sslmode=verify-full&sslrootcert=/var/lib/cockroach/certs/ca.crt'
```

## Upgrade Procedure

```bash
# Step 1: Upgrade Ansible variable
vim group_vars/cockroachdb.yml
# Change: cockroachdb_version: "23.2.4"

# Step 2: Rolling upgrade
ansible-playbook playbooks/cockroachdb-cluster.yml --tags upgrade --serial=1

# Step 3: Finalize upgrade
cockroach sql --execute="SET CLUSTER SETTING version = '23.2';"
```

## Support

- **Logs**: `/var/lib/cockroach/logs/`
- **Metrics**: `http://<node>:8080/_status/vars`
- **Health endpoint**: `http://<node>:8080/health?ready=1`
- **CockroachDB Docs**: https://www.cockroachlabs.com/docs/

## See Also

- [Troubleshooting Guide](TROUBLESHOOTING.md)
- [GDPR Compliance](GDPR-COMPLIANCE.md)
- [Monitoring Guide](MONITORING.md)
