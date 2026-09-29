# Production RKE2 Cluster Deployment & Operations Guide

This document provides a complete, end-to-end operational and deployment guide for building, managing, and expanding a production-grade **RKE2 (Rancher Kubernetes Engine 2)** cluster with **Cilium CNI**, **H/A Control Plane (Kube-VIP)**, **MetalLB Load Balancer**, **Dual NGINX Ingress Controllers**, **Rook-Ceph Storage**, **Centralized Wildcard TLS Certificates**, **Prometheus/Grafana/Loki Observability Stack**, and a **DevOps Platform (Harbor Container Registry & Argo CD GitOps)**.

---

## 1. Executive Summary & Architecture Overview

The automation in this repository provisions an enterprise-grade Kubernetes platform designed for high availability, network isolation, persistent distributed storage, and GitOps CI/CD workflows.

```
                         [ Internal Clients / Developers ]       [ External Internet Traffic ]
                                         │                                      │
                                         ▼                                      ▼
                        ┌─────────────────────────────────┐   ┌──────────────────────────────────┐
                        │  MetalLB VIP: 192.168.28.125    │   │   MetalLB VIP: 192.168.28.127    │
                        │ Internal Ingress (nginx-internal) │   │ External Ingress (nginx-external)│
                        └────────────────┬────────────────┘   └────────────────┬─────────────────┘
                                         │                                     │
           ┌─────────────────────────────┼─────────────────────────────┐       │
           │                             │                             │       │
           ▼                             ▼                             ▼       ▼
┌──────────────────────┐      ┌──────────────────────┐      ┌──────────────────────┐
│  Grafana / Prometheus│      │  Harbor Registry     │      │   Argo CD GitOps     │
│ grafana.dishhome...  │      │ harbor.dishhome...   │      │ argocd.dishhome...   │
└──────────────────────┘      └──────────────────────┘      └──────────────────────┘
           │                             │                             │
           └─────────────────────────────┼─────────────────────────────┘
                                         ▼
                 ┌─────────────────────────────────────────────────┐
                 │ Rook-Ceph Block Storage Class (`ceph-block`)     │
                 └─────────────────────────────────────────────────┘
```

### Core Components Summary

| Component | Technology | Description / Justification |
| :--- | :--- | :--- |
| **Control Plane HA** | Kube-VIP (`192.168.28.129`) | Virtual Floating IP across master nodes for seamless API server failover. |
| **Networking & CNI** | Cilium (v1.15.5) | eBPF-based high performance CNI replacing Canal. |
| **Load Balancer** | MetalLB (`192.168.28.125-128`) | Bare-metal Layer-2 ARP LoadBalancer service allocator. |
| **Dual Ingress** | NGINX (`nginx-internal` / `nginx-external`) | Strict physical traffic separation between internal admin tools and public web apps. |
| **Persistent Storage** | Rook-Ceph (`ceph-block`) | Distributed block & file storage leveraging raw disks on dedicated storage nodes. |
| **TLS Automation** | Wildcard SSL (`*.dishhome.com.np`) | Auto-generates or syncs wildcard TLS secret into all application namespaces. |
| **Observability** | kube-prometheus-stack + Loki | Full metric monitoring (Prometheus + Grafana) and log aggregation (Loki + Promtail). |
| **DevOps Platform** | Harbor (v1.14.0) + Argo CD (v6.7.1) | Enterprise image registry with security scanning + GitOps continuous deployment. |
| **External CI/CD** | GitLab (`git.dishhome.com.np`) | Dedicated external VM running GitLab & GitLab Runners. |

---

## 2. Infrastructure & Inventory Configuration

### 2.1 Hardware Requirements

- **3 Master Nodes (Control Plane)**: `192.168.28.120` – `192.168.28.122` (Min 4 vCPU, 8GB RAM, 50GB Disk)
- **3 Worker Nodes (Data Plane)**: `192.168.28.130` – `192.168.28.132` (Min 4 vCPU, 16GB RAM)
- **3 Storage Nodes (Ceph)**: Co-located on workers or dedicated nodes with raw, unformatted disk drives (`/dev/sdb`, `/dev/nvme1n1`, etc.).

### 2.2 Ansible Inventory (`inventory.ini`)

```ini
[rke2_servers]
master1 ansible_host=192.168.28.120
master2 ansible_host=192.168.28.121
master3 ansible_host=192.168.28.122

[rke2_agents]
worker1 ansible_host=192.168.28.130
worker2 ansible_host=192.168.28.131
worker3 ansible_host=192.168.28.132

[ceph_agents]
ceph1 ansible_host=192.168.28.130
ceph2 ansible_host=192.168.28.131
ceph3 ansible_host=192.168.28.132

[k8s_cluster:children]
rke2_servers
rke2_agents
ceph_agents
```

### 2.3 Global Variables (`group_vars/all.yml`)

The `group_vars/all.yml` file acts as the single source of truth for global cluster parameters:

```yaml
---
# RKE2 Version
rke2_version: "v1.28.10+rke2r1"

# Cluster VIP for HA Control Plane
cluster_vip: "192.168.28.129"
cluster_domain: "k8s.dishhome.com.np"

# Networking
rke2_cni: "cilium"
cilium_version: "1.15.5"

# RKE2 Config
rke2_config_dir: "/etc/rancher/rke2"
rke2_token: "{{ lookup('password', 'credentials/node-token length=32 chars=ascii_letters,digits') }}"

# OS Settings
disable_firewalld: true
enable_iscsi: true

# TLS & Certificate Management
tls_secret_name: "dishhome-wildcard-tls"
tls_namespaces:
  - "rook-ceph"
  - "default"
  - "kube-system"
  - "ingress-internal"
  - "ingress-external"
  - "monitoring"
  - "devops-harbor"
  - "devops-argocd"

# DevOps Platform Configuration
harbor_domain: "harbor.dishhome.com.np"
harbor_namespace: "devops-harbor"
harbor_chart_version: "1.14.0"
harbor_storage_class: "ceph-block"
harbor_admin_password: "DishHomeAdmin123!"

argocd_domain: "argocd.dishhome.com.np"
argocd_namespace: "devops-argocd"
argocd_chart_version: "6.7.1"
argocd_storage_class: "ceph-block"

gitlab_url: "https://git.dishhome.com.np"

# Global Ingress Classes
ingress_internal_class: "nginx-internal"
```

---

## 3. Initial Bootstrap & Deployment

### 3.1 Step 1: Bootstrap Administrative User

Run `bootstrap.yml` to create a dedicated user `dictator` with passwordless `sudo` rights and deploy your SSH key:

```bash
ansible-playbook bootstrap.yml -u ubuntu -k -K
```

### 3.2 Step 2: Full End-to-End Cluster Deployment

Execute the entire deployment workflow in sequence using the `dictator` user:

```bash
ansible-playbook site.yml -u dictator
```

### 3.3 Granular Deployment Phases (By Tags)

If deploying step-by-step or troubleshooting specific components, execute using Ansible tags:

| Phase | Description | Command |
| :--- | :--- | :--- |
| **Phase 1** | System preparation, packages, iSCSI, firewall | `ansible-playbook site.yml -u dictator --tags common` |
| **Phase 2** | Initialize first Master | `ansible-playbook site.yml -u dictator --tags init` |
| **Phase 3** | Deploy Kube-VIP (HA Control Plane) | `ansible-playbook site.yml -u dictator --tags vip` |
| **Phase 4** | Join remaining Master nodes | `ansible-playbook site.yml -u dictator --tags join_server` |
| **Phase 5** | Join Worker and Ceph storage nodes | `ansible-playbook site.yml -u dictator --tags join_agent,join_ceph` |
| **Phase 6** | MetalLB & Dual Ingress Controllers | `ansible-playbook site.yml -u dictator --tags networking,dual_ingress` |
| **Phase 7** | Centralized Wildcard TLS Secrets | `ansible-playbook site.yml -u dictator --tags tls_certs` |
| **Phase 8** | Rook-Ceph Storage Cluster | `ansible-playbook site.yml -u dictator --tags storage` |
| **Phase 9** | Observability (Prometheus, Grafana, Loki) | `ansible-playbook site.yml -u dictator --tags monitoring` |
| **Phase 10**| DevOps Platform (Harbor & Argo CD) | `ansible-playbook site.yml -u dictator --tags devops` |

---

## 4. Component Deep-Dive & Key Technical Decisions

### 4.1 Dual NGINX Ingress Controllers
To ensure strict security boundaries, two NGINX ingress controllers are deployed:
- **Internal Ingress (`nginx-internal`)**:
  - Bound to MetalLB IP `192.168.28.125`.
  - Exposes internal operational dashboards (Ceph Dashboard, Grafana, Prometheus, Harbor Registry, Argo CD UI).
  - Keeps sensitive tools invisible from the public internet.
- **External Ingress (`nginx-external`)**:
  - Bound to MetalLB IP `192.168.28.127`.
  - Dedicated exclusively for public user-facing applications.

### 4.2 Rook-Ceph Distributed Storage Class (`ceph-block`)
- Provisions resilient block storage across nodes labeled with `node-role.kubernetes.io/ceph=true`.
- Dynamically creates the `ceph-block` StorageClass and sets it as default (`is-default-class: "true"`).
- Automatically handles disk detection, Ceph OSD lifecycle, and Ceph Dashboard exposure on `https://ceph.dishhome.com.np`.

### 4.3 Centralized TLS Certificate Sync
- Looks for custom Wildcard SSL certificates at `~/certificates/tls.crt` and `~/certificates/tls.key` on the Ansible controller.
- If certificates are missing, automatically creates a self-signed fallback certificate for `*.dishhome.com.np`.
- Automatically syncs the `dishhome-wildcard-tls` secret into all operational namespaces (`kube-system`, `ingress-internal`, `ingress-external`, `monitoring`, `rook-ceph`, `devops-harbor`, `devops-argocd`).

### 4.4 DevOps Platform (Harbor & Argo CD)
- **Harbor Container Registry (`harbor.dishhome.com.np`)**:
  - Utilizes `ceph-block` storage for PostgreSQL, Redis, Trivy vulnerability DB, and image blob persistence.
  - Ingress configured with `proxy-body-size: "0"` for unlimited image upload layer size.
- **Argo CD GitOps Engine (`argocd.dishhome.com.np`)**:
  - Configured with `redis.persistence.enabled: false` (in-memory caching). Because Argo CD's source of truth is stored in Kubernetes CRDs (`etcd`), non-persistent Redis eliminates unnecessary PVC overhead, speeds up Pod recovery, and avoids disk attach delays.
- **External GitLab (`git.dishhome.com.np`)**:
  - Hosted on a dedicated external VM to isolate CPU/memory heavy CI/CD docker builds from the Kubernetes production cluster nodes.

---

## 5. Observability & Monitoring Stack

The monitoring stack is deployed via `kube-prometheus-stack` and `loki-stack`:

### Exposed Endpoints (Internal Ingress: `192.168.28.125`)
- **Grafana**: `https://grafana.dishhome.com.np`
- **Prometheus**: `https://prometheus.dishhome.com.np`
- **AlertManager**: `https://alertmanager.dishhome.com.np`

### Credentials & Access

```bash
# Retrieve Grafana Admin Password
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 --decode; echo
```

---

## 6. Cluster Lifecycle & Operational Playbooks

This repository includes dedicated playbooks in the `updatecluster/` directory for scaling, node maintenance, and cluster teardown.

### 6.1 Scaling Out: Adding Nodes

#### Add a New Master Node (`add_master.yml`)
1. Add the new host to `[rke2_servers]` in `inventory.ini`.
2. Run:
   ```bash
   ansible-playbook updatecluster/add_master.yml --limit master4
   ```

#### Add a New Worker Node (`add_worker.yml`)
1. Add the host to `[rke2_agents]` in `inventory.ini`.
2. Run:
   ```bash
   ansible-playbook updatecluster/add_worker.yml --limit worker4
   ```

#### Add a New Ceph Storage Node (`add_ceph.yml`)
1. Add the host to `[ceph_agents]` in `inventory.ini`.
2. Run:
   ```bash
   ansible-playbook updatecluster/add_ceph.yml --limit ceph4
   ```

---

### 6.2 Scaling In: Removing Nodes Gracefully

#### Remove a Master Node (`remove_master.yml`)
Drains the node, cordons it, stops the RKE2 service, removes it from the etcd cluster, and cleans up binaries:
```bash
ansible-playbook updatecluster/remove_master.yml --limit master3
```

#### Remove a Worker Node (`remove_worker.yml`)
Evacuates running workloads safely before uninstalling RKE2 binaries:
```bash
ansible-playbook updatecluster/remove_worker.yml --limit worker3
```

#### Remove a Ceph Storage Node (`remove_ceph.yml`)
Safely drains OSDs, waits for Ceph HEALTH_OK status, wipes LVM/Ceph disk metadata, and uninstalls RKE2:
```bash
ansible-playbook updatecluster/remove_ceph.yml --limit ceph3
```

---

### 6.3 Complete Cluster Teardown (`reset.yml`)

To completely wipe and reset all nodes in the cluster back to clean OS state:

```bash
ansible-playbook reset.yml -u dictator
```

> [!CAUTION]
> Running `reset.yml` will permanently destroy all RKE2 data, Rook-Ceph storage pools, network interfaces, container volumes, and etcd cluster state.

---

## 7. Verification & Post-Deployment Health Check

Log into `master1` (`ssh dictator@192.168.28.120`):

```bash
# 1. Check All Nodes and Roles
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -o wide --show-labels

# 2. Check All Pods across Namespaces
sudo /var/lib/rancher/rke2/bin/kubectl get pods -A

# 3. Check LoadBalancer Ingress IPs (MetalLB)
sudo /var/lib/rancher/rke2/bin/kubectl get svc -A | grep LoadBalancer

# 4. Verify Rook-Ceph Storage Class Status
sudo /var/lib/rancher/rke2/bin/kubectl get sc
sudo /var/lib/rancher/rke2/bin/kubectl get storagecluster -n rook-ceph

# 5. Verify Argo CD Admin Initial Secret
sudo /var/lib/rancher/rke2/bin/kubectl -n devops-argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

---

## 8. Network DNS Requirements

Add the following DNS records to your enterprise DNS server (or local `/etc/hosts` for testing) pointing to the **Internal Ingress IP (`192.168.28.125`)**:

```text
192.168.28.125 ceph.dishhome.com.np
192.168.28.125 grafana.dishhome.com.np
192.168.28.125 prometheus.dishhome.com.np
192.168.28.125 alertmanager.dishhome.com.np
192.168.28.125 harbor.dishhome.com.np
192.168.28.125 argocd.dishhome.com.np
```
