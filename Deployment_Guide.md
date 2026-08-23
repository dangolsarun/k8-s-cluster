# Production RKE2 Cluster Deployment Guide

This document provides a complete, step-by-step guide to deploying a production-grade RKE2 cluster with Cilium CNI, HA Control Plane, CIS Hardening, and Rook-Ceph storage using the Ansible automation in this repository.

## 1. Environment Preparation

### 1.1 Infrastructure
You need virtual machines (Ubuntu/Rocky/RHEL):
- **3 Masters (Servers)**: `192.168.28.120` - `192.168.28.122`
- **3 Workers (Agents)**: `192.168.28.130` - `192.168.28.132`
- **3 Ceph Storage Nodes**: `192.168.28.130` - `192.168.28.132` (Co-located or dedicated)

### 1.2 Networking
- **Cluster VIP**: `192.168.28.129` (Defined in `group_vars/all.yml`). Reserved IP in the same subnet as your masters.
- **Firewall**: The `common` role will disable `ufw`/`firewalld` to let CNI handle traffic.

### 1.3 Ansible Controller Setup
On your deployment machine:
1. **Install Ansible**: `sudo apt install ansible` (or via pip).
2. **Generate SSH Key**:
   ```bash
   ssh-keygen -t ed25519 -C "Sarun to k8's Cluster"
   ```

## 2. Configuration

### 2.1 Inventory (`inventory.ini`)
Verify your nodes are listed correctly under their respective groups. Worker nodes (`rke2_agents`) and Ceph storage nodes (`ceph_agents`) can be distinct dedicated hosts or co-located:
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

### 2.2 Global Config (`group_vars/all.yml`)
- **VIP**: Ensure `cluster_vip` matches your plan.
- **Token**: The token is auto-generated and saved to `credentials/node-token` on first run.

## 3. Bootstrap (User Setup)

Run the **Bootstrap Playbook** to configure the `dictator` user with passwordless sudo and your SSH key.

**Prerequisite**: You must have initial access (e.g., as `ubuntu` or `root`) with a password.

```bash
# Run verifying with a known user (e.g., ubuntu)
ansible-playbook bootstrap.yml -u ubuntu -k -K
```

- `-u ubuntu`: Connect as `ubuntu` initially.
- `-k`: Ask for SSH password.
- `-K`: Ask for Sudo password.

**Result**: A user `dictator` is created on all nodes with your ED25519 key and `NOPASSWD` sudo rights.

## 4. Cluster Deployment

Now deployment runs fully automated using the `dictator` user.

### 4.1 Granular Deployment (Recommended for Learning)
You can run steps individually using tags:
1. **Common**: `ansible-playbook site.yml -u dictator --tags common`
2. **Masters**: `ansible-playbook site.yml -u dictator --tags init,vip,join_server`
3. **Workers**: `ansible-playbook site.yml -u dictator --tags join_agent`
4. **Ceph Storage Nodes**: `ansible-playbook site.yml -u dictator --tags join_ceph`

### 4.2 Full Deployment (All at once)
```bash
ansible-playbook site.yml -u dictator
```

**What happens?**
1. **Common**: Disables swap, loads kernel modules, installs packages across all hosts.
2. **Server**:
   - Installs RKE2 on Master 1.
   - Starts `kube-vip` (HA).
   - Installs RKE2 on Master 2 & 3 and joins them.
   - Configures `cis-1.23` profile.
3. **Agents**:
   - Installs RKE2 on Worker nodes (`rke2_agents`) and labels them with `node-role.kubernetes.io/worker=true`.
   - Installs RKE2 on Ceph storage nodes (`ceph_agents`) and labels them with `node-role.kubernetes.io/ceph=true`.
4. **Networking**: 
   - Deploys `metallb` referencing the address pool configured in `group_vars/all.yml` (e.g., `192.168.28.125-192.168.28.128`).
   - Deploys **Dual Ingress Controllers**:
     - **Internal Ingress** (`nginx-internal`): Binds to `192.168.28.125` inside `ingress-internal` namespace.
     - **External Ingress** (`nginx-external`): Binds to `192.168.28.127` inside `ingress-external` namespace.
5. **TLS Certificates**:
   - Checks your Ansible controller for `~/certificates/tls.crt` and `~/certificates/tls.key`.
   - If these files **exist locally**, it copies them securely to the cluster.
   - If these files **do not exist**, it automatically generates a valid Self-Signed Wildcard Certificate for `*.dishhome.com.np` directly on the server to prevent deployment failures.
   - Deploys the `dishhome-wildcard-tls` Secret into multiple predefined namespaces (e.g., `kube-system`, `ingress-internal`, `ingress-external`, `monitoring`, `rook-ceph`).
6. **Storage**:
   - Installs the **Rook-Ceph** Operator.
   - Provisions storage using all available raw devices on `ceph_agents` nodes (`node-role.kubernetes.io/ceph=true`).
   - Automatically clones the wildcard certificate into the `rook-ceph` namespace.
   - Exposes the Ceph Dashboard at `ceph.dishhome.com.np` utilizing the Internal Ingress controller via HTTPS.

## 5. Verification

Log into `master1` (`ssh dictator@192.168.28.120`):

```bash
# Check Nodes and Labels
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -o wide --show-labels

# Check Ceph Storage Nodes specifically
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -l node-role.kubernetes.io/ceph=true

# Check Pods (Cilium, CoreDNS, Rook-Ceph)
sudo /var/lib/rancher/rke2/bin/kubectl get pods -A

# Check Local Kubeconfig
ls -l ~/.kube/config

# Verify Ingress IPs allocated via MetalLB
sudo /var/lib/rancher/rke2/bin/kubectl get svc -A | grep LoadBalancer

# Verify TLS Secrets
sudo /var/lib/rancher/rke2/bin/kubectl get secrets -A | grep tls
```

## 6. Maintenance

### 6.1 Certificate Management
If you need to update your Wildcard Certificate:
1. Place the new `tls.crt` and `tls.key` in your Ansible host user's `~/certificates/` directory.
2. Run the TLS certs tag again:
   ```bash
   ansible-playbook site.yml -u dictator --tags tls_certs
   ```
This will forcefully recreate the secrets in all necessary namespaces across the cluster.

### 6.2 Adding a New Worker Node
To add a new worker (e.g., `worker4`):
1. **Update Inventory**: Add `worker4` under `[rke2_agents]` in `inventory.ini`.
2. **Add Worker Node**:
   ```bash
   ansible-playbook updatecluster/add_worker.yml --limit worker4
   ```

### 6.3 Adding / Removing a Ceph Storage Node
To add a dedicated or new Ceph node (e.g., `ceph4`):
1. **Update Inventory**: Add `ceph4` under `[ceph_agents]` in `inventory.ini`.
2. **Add Ceph Node**:
   ```bash
   ansible-playbook updatecluster/add_ceph.yml --limit ceph4
   ```
To safely drain, delete, and wipe a Ceph node:
```bash
ansible-playbook updatecluster/remove_ceph.yml --limit ceph4
```

### 6.4 Upgrading
Change `rke2_version` in `group_vars/all.yml` and re-run `site.yml`.

---

## 7. Component Architecture & Justification

To ensure a highly available, secure, and production-ready environment, this deployment utilizes several industry-standard CNCF components. Here is a detailed breakdown of why each component is used and what it does.

### 7.1 Kube-VIP (Control Plane High Availability)
- **What it does**: Provides a Virtual IP (`192.168.28.129`) that floats between the Master nodes.
- **Why it's used**: By default, worker nodes and external clients connect directly to a single Master node's IP. If that Master dies, the cluster API becomes unreachable. Kube-VIP ensures that if `master1` fails, the VIP instantly fails over to `master2` or `master3`, guaranteeing zero-downtime access to the Kubernetes API server.

### 7.2 Cilium (Container Network Interface - CNI)
- **What it does**: Manages all pod-to-pod and node-to-node networking using eBPF (Extended Berkeley Packet Filter) technology natively within the Linux kernel.
- **Why it's used**: It replaces the default RKE2 Canal CNI because Cilium is significantly faster, highly scalable, and provides advanced features like strict NetworkPolicies, transparent encryption, and deep network observability (Hubble) without the overhead of traditional iptables routing.

### 7.3 MetalLB (Network Load Balancer)
- **What it does**: Acts as a bare-metal LoadBalancer implementation. It allocates IPs from a reserved pool (e.g., `192.168.28.125-128`) to Kubernetes `Service` objects of type `LoadBalancer`.
- **Why it's used**: In cloud environments (AWS/GCP), the cloud provider natively provisions LoadBalancers. In bare-metal or on-premise VM environments, Kubernetes cannot automatically provision an external IP. MetalLB bridges this gap by broadcasting these IPs via Layer 2 ARP to your local network router, making your Ingress controllers reachable.

### 7.4 Dual NGINX Ingress (Internal & External Traffic Isolation)
- **What it does**: We deploy *two* separate NGINX Ingress Controllers instead of the default one. 
  - **Internal Ingress**: Bound to `192.168.28.125`. Routes traffic for internal admin dashboards (like Ceph) and private company apps.
  - **External Ingress**: Bound to `192.168.28.127`. Dedicated strictly for public-facing internet traffic.
- **Why it's used**: Security and isolation. It prevents internal dashboards from accidentally being exposed to the internet. We can easily apply aggressive Web Application Firewalls (WAF) or rate-limiting on the External ingress while keeping the Internal ingress unrestricted for developers.

### 7.5 Rook-Ceph (Distributed Persistent Storage)
- **What it does**: Turns raw, unformatted disk drives attached to designated Ceph nodes (`ceph_agents`) into a highly available, distributed storage cluster. It provides `ReadWriteOnce` Block storage (RBD) and `ReadWriteMany` File storage (CephFS) natively to your Pods.
- **Why it's used**: Pods are ephemeral; if they die, local data is lost. Rook-Ceph replicates data across multiple nodes (`ceph1`, `ceph2`, `ceph3`). If one node's hard drive fails, the data is safely preserved on the remaining Ceph nodes. The Operator manages the entire lifecycle, self-healing, and dashboarding automatically.

### 7.6 Automated TLS Certificate Management
- **What it does**: The Ansible `tls_certs` role orchestrates the secure distribution of your organization's Wildcard TLS Certificate (`*.dishhome.com.np`). 
- **Why it's used**: Copying certificates manually into multiple namespaces is error-prone. The automation checks your local machine for `~/certificates/tls.crt`, securely injects it into Kubernetes as a `Secret`, and loops through necessary namespaces (`kube-system`, `ingress-internal`, `ingress-external`, `rook-ceph`). If it detects a missing certificate during a fresh install, it dynamically generates a valid self-signed fallback to prevent deployment crashes.
