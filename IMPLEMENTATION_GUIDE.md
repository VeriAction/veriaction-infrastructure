# VeriAction Infrastructure - Implementation Guide

## What Has Been Created

This repository contains **production-ready Ansible automation** for deploying the complete VeriAction infrastructure with GDPR compliance and multi-region support.

### Created Files and Structure

```
veriaction-infrastructure/
├── ansible.cfg                  ✅ Complete Ansible configuration
├── README.md                    ✅ Comprehensive overview
├── IMPLEMENTATION_GUIDE.md      ✅ This file
│
├── inventory/
│   ├── production              ✅ Production inventory (EU + Global regions)
│   └── staging                 ✅ Staging inventory
│
├── group_vars/
│   ├── all.yml                 ✅ Global variables
│   ├── cockroachdb.yml         ✅ CockroachDB configuration
│   ├── kubernetes.yml          ✅ Kubernetes configuration
│   ├── haproxy.yml             ✅ HAProxy configuration
│   ├── monitoring.yml          ✅ Monitoring stack configuration
│   └── vault.yml.example       ✅ Vault template for secrets
│
├── playbooks/
│   ├── site.yml                ✅ Master playbook (deploys everything)
│   ├── cockroachdb-cluster.yml ✅ CockroachDB deployment
│   ├── kubernetes-cluster.yml  ✅ Kubernetes cluster bootstrap
│   ├── haproxy-cluster.yml     ✅ HAProxy load balancers
│   ├── monitoring-stack.yml    ✅ Prometheus + Grafana + Alertmanager
│   └── deploy-geoip.yml        ✅ GeoIP service deployment
│
├── roles/                      ⚠️  Directory structure created (tasks TODO)
│   ├── common/
│   ├── cockroachdb/
│   ├── kubernetes/
│   ├── haproxy/
│   ├── prometheus/
│   ├── grafana/
│   ├── alertmanager/
│   ├── node_exporter/
│   ├── geoip-service/
│   └── keepalived/
│
└── docs/
    └── COCKROACHDB.md          ✅ Complete CockroachDB guide
```

## Implementation Status

### ✅ Complete (Ready to Use)

1. **Inventory Files**
   - Production inventory with EU/Global regions
   - Staging inventory
   - Proper group hierarchies for GDPR compliance

2. **Configuration Variables**
   - All group_vars files with production-ready defaults
   - Comprehensive settings for all components
   - GDPR-compliant regional configurations

3. **Playbooks**
   - Master deployment playbook (site.yml)
   - Component-specific playbooks for each service
   - Validation and health check tasks included

4. **Documentation**
   - README with quick start guide
   - Comprehensive CockroachDB deployment guide
   - Implementation guide (this file)

### ⚠️ TODO (Next Steps)

1. **Ansible Role Implementation**

   Each role in `roles/` directory needs task implementation. The structure is created, but `tasks/main.yml` files need to be populated.

   **Priority order:**
   1. `roles/common/` - Base system setup (users, packages, firewall)
   2. `roles/cockroachdb/` - CockroachDB installation and configuration
   3. `roles/kubernetes-prerequisites/` - Container runtime, dependencies
   4. `roles/kubernetes-master/` - K8s master node initialization
   5. `roles/kubernetes-worker/` - K8s worker node joining
   6. `roles/kubernetes-networking/` - Calico CNI deployment
   7. `roles/kubernetes-addons/` - Metrics server, cert-manager, ingress
   8. `roles/haproxy/` - HAProxy installation and configuration
   9. `roles/keepalived/` - Keepalived for VIP failover
   10. `roles/prometheus/` - Prometheus deployment
   11. `roles/alertmanager/` - Alertmanager deployment
   12. `roles/grafana/` - Grafana with dashboards
   13. `roles/node_exporter/` - Node exporter for metrics

2. **Template Files**

   Create Jinja2 templates in `templates/` directory:
   - CockroachDB systemd service
   - HAProxy configuration
   - Kubernetes kubeadm configs
   - Prometheus configuration and alert rules
   - Grafana datasources and dashboards
   - GeoIP Kubernetes manifests

3. **Static Files**

   Place in `files/` directory:
   - Prometheus alert rules (copy from veriaction-geoip/deployments/prometheus/)
   - Grafana dashboard JSON files
   - SSL certificates (if pre-generated)

4. **Additional Documentation**

   Complete these guides:
   - `docs/KUBERNETES.md` - Kubernetes deployment guide
   - `docs/HAPROXY.md` - HAProxy setup guide
   - `docs/DEPLOYMENT.md` - GeoIP deployment guide
   - `docs/MONITORING.md` - Monitoring stack guide
   - `docs/GDPR-COMPLIANCE.md` - GDPR compliance verification
   - `docs/TROUBLESHOOTING.md` - Common issues and solutions

5. **Helper Scripts**

   Implement in `scripts/`:
   - `bootstrap-servers.sh` - Initial server setup
   - `verify-data-residency.sh` - GDPR compliance check
   - `generate-certs.sh` - Certificate generation
   - `backup-cockroachdb.sh` - Manual backup script

## Quick Start for Implementation

### Step 1: Complete Role Tasks

Start with the `common` role as an example:

```bash
cd roles/common/tasks
vim main.yml
```

Example `roles/common/tasks/main.yml`:
```yaml
---
# Common system setup tasks

- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600
  when: ansible_os_family == 'Debian'

- name: Install common packages
  apt:
    name: "{{ system_packages }}"
    state: present
  when: ansible_os_family == 'Debian'

- name: Create service user
  user:
    name: "{{ service_user }}"
    uid: "{{ service_uid }}"
    group: "{{ service_group }}"
    system: yes
    create_home: yes

- name: Configure firewall
  ufw:
    state: enabled
    policy: "{{ firewall_default_policy }}"
  when: firewall_enabled

- name: Apply sysctl settings
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop: "{{ sysctl_settings | dict2items }}"

- name: Configure system limits
  pam_limits:
    domain: "{{ item.domain }}"
    limit_type: "{{ item.type }}"
    limit_item: "{{ item.item }}"
    value: "{{ item.value }}"
  loop: "{{ system_limits }}"
```

### Step 2: Create Templates

Example `templates/cockroachdb/cockroachdb.service.j2`:
```ini
[Unit]
Description=CockroachDB
After=network.target

[Service]
Type=notify
User={{ cockroachdb_user }}
Group={{ cockroachdb_group }}
ExecStart=/usr/local/bin/cockroach start \
  --certs-dir={{ cockroachdb_certs_dir }} \
  --store={{ cockroachdb_data_dir }} \
  --listen-addr={{ ansible_host }}:{{ cockroachdb_port }} \
  --http-addr={{ ansible_host }}:{{ cockroachdb_http_port }} \
  --join={{ cockroachdb_join_list }} \
  --cache={{ cockroachdb_cache_size }} \
  --max-sql-memory={{ cockroachdb_max_sql_memory }} \
  --locality={{ cockroachdb_locality }}

Restart=on-failure
RestartSec={{ cockroachdb_restart_delay }}
LimitNOFILE={{ cockroachdb_service_limits.LimitNOFILE }}
LimitNPROC={{ cockroachdb_service_limits.LimitNPROC }}
LimitCORE={{ cockroachdb_service_limits.LimitCORE }}

[Install]
WantedBy=multi-user.target
```

### Step 3: Copy Alert Rules

```bash
# Copy Prometheus alert rules from GeoIP repo
cp ../veriaction-geoip/deployments/prometheus/hotreload-alerts.yaml \
   files/prometheus/geoip-alerts.yml
```

### Step 4: Configure Inventory

Edit `inventory/production` and replace `<IP_ADDRESS>` placeholders with actual server IPs:

```ini
[eu_cockroachdb]
eu-crdb-01 ansible_host=10.0.1.10 cockroachdb_node_id=1
eu-crdb-02 ansible_host=10.0.1.11 cockroachdb_node_id=2
eu-crdb-03 ansible_host=10.0.1.12 cockroachdb_node_id=3
```

### Step 5: Create Vault

```bash
ansible-vault create group_vars/vault.yml
```

Use `group_vars/vault.yml.example` as a template.

### Step 6: Test Deployment

```bash
# Test connectivity
ansible all -m ping

# Deploy to staging first
ansible-playbook playbooks/site.yml -i inventory/staging --check

# Deploy to production
ansible-playbook playbooks/site.yml
```

## Role Implementation Guide

### Template for Each Role

```
roles/<role_name>/
├── tasks/
│   ├── main.yml           # Main task orchestration
│   ├── install.yml        # Installation tasks
│   ├── configure.yml      # Configuration tasks
│   ├── certificates.yml   # Certificate management
│   └── validate.yml       # Health checks
├── templates/
│   ├── <service>.conf.j2  # Configuration templates
│   └── <service>.service.j2 # Systemd service
├── files/
│   └── <static-files>     # Static configuration files
├── handlers/
│   └── main.yml           # Service restart handlers
├── defaults/
│   └── main.yml           # Default variables (optional)
├── vars/
│   └── main.yml           # Role-specific variables
└── meta/
    └── main.yml           # Role dependencies
```

### Handler Example

`roles/cockroachdb/handlers/main.yml`:
```yaml
---
- name: restart cockroachdb
  systemd:
    name: cockroachdb
    state: restarted
    daemon_reload: yes

- name: reload cockroachdb
  systemd:
    name: cockroachdb
    state: reloaded
```

## Testing Strategy

### 1. Syntax Check
```bash
ansible-playbook playbooks/site.yml --syntax-check
```

### 2. Dry Run
```bash
ansible-playbook playbooks/site.yml --check
```

### 3. Staging Deployment
```bash
ansible-playbook playbooks/site.yml -i inventory/staging
```

### 4. Production Deployment (Step-by-Step)
```bash
# Deploy one component at a time
ansible-playbook playbooks/cockroachdb-cluster.yml --limit eu_cockroachdb
ansible-playbook playbooks/kubernetes-cluster.yml --limit eu_k8s_masters
# ... etc
```

## GDPR Compliance Checklist

- [ ] EU CockroachDB cluster uses only EU servers
- [ ] Global CockroachDB cluster separated from EU
- [ ] No cross-region replication configured
- [ ] Audit logging enabled in both regions
- [ ] Data encryption at rest enabled
- [ ] TLS encryption for all communications
- [ ] Zone constraints properly configured
- [ ] Backup destinations region-specific

## Production Readiness Checklist

### Infrastructure
- [ ] All servers have static IPs
- [ ] DNS properly configured
- [ ] Firewall rules implemented
- [ ] NTP synchronized across all nodes
- [ ] Swap disabled on Kubernetes nodes

### Security
- [ ] SSH keys properly distributed
- [ ] Ansible vault encrypted
- [ ] TLS certificates generated
- [ ] Strong passwords set
- [ ] Firewall enabled on all nodes
- [ ] SELinux/AppArmor configured

### Monitoring
- [ ] Prometheus scraping all endpoints
- [ ] Grafana dashboards imported
- [ ] Alertmanager routing configured
- [ ] Alert destinations tested (Slack/PagerDuty)
- [ ] Node exporters running on all servers

### Backups
- [ ] S3 bucket created for backups
- [ ] Backup schedules configured
- [ ] Restore procedure tested
- [ ] Retention policies set

### Documentation
- [ ] Inventory documented
- [ ] Network diagram created
- [ ] Access credentials stored securely
- [ ] Runbooks created for common operations
- [ ] On-call procedures documented

## Deployment Timeline Estimate

| Phase | Duration | Tasks |
|-------|----------|-------|
| Role Implementation | 3-5 days | Implement all role tasks |
| Template Creation | 2-3 days | Create all Jinja2 templates |
| Testing (Staging) | 2-3 days | Deploy and test in staging |
| Documentation | 1-2 days | Complete remaining docs |
| Production Deployment | 1 day | Deploy to production |
| Validation | 1 day | Health checks, monitoring setup |
| **Total** | **10-15 days** | Full implementation |

## Next Steps

1. **Implement roles** starting with `common` and `cockroachdb`
2. **Create templates** for all configuration files
3. **Copy alert rules** from veriaction-geoip repository
4. **Configure inventory** with actual server IPs
5. **Create vault** with all secrets
6. **Test in staging** environment first
7. **Deploy to production** with validation at each step

## Support and Resources

- **Ansible Docs**: https://docs.ansible.com/
- **CockroachDB Docs**: https://www.cockroachlabs.com/docs/
- **Kubernetes Docs**: https://kubernetes.io/docs/
- **HAProxy Docs**: http://www.haproxy.org/
- **Prometheus Docs**: https://prometheus.io/docs/

## Contact

For questions or issues during implementation:
- Check `docs/TROUBLESHOOTING.md`
- Review role-specific documentation
- Check Ansible verbose output: `-vvv`

---

**Created**: 2026-02-12
**Status**: Foundation Complete, Role Implementation Pending
**Maintained by**: VeriAction DevOps Team
