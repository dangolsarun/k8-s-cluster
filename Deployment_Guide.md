# Production RKE2 Cluster Deployment Guide

This document provides a complete, step-by-step guide to deploying a production-grade RKE2 cluster with Cilium CNI, HA Control Plane, and CIS Hardening using the Ansible automation in this repository.

## 1. Environment Preparation

### 1.1 Infrastructure
You need 7 Virtual Machines (Ubuntu/Rocky/RHEL):
- **3 Masters (Servers)**: 192.168.28.120 - 122
- **3 Workers (Agents)**: 192.168.28.130 - 132
- **1 Worker (Spare)**: 192.168.28.133 (Currently disabled in inventory)

### 1.2 Networking
- **Cluster VIP**: `192.168.1.100` (Defined in `group_vars/all.yml`). Ideally, this should be a reserved IP in the same subnet as your masters.
- **Firewall**: The `common` role will disable `ufw`/`firewalld` to let CNI handle traffic.

### 1.3 Ansible Controller Setup
On your deployment machine:
1.  **Install Ansible**: `sudo apt install ansible` (or via pip).
2.  **Generate SSH Key** (You just did this):
    ```bash
    ssh-keygen -t ed25519 -C "Sarun to k8's Cluster"
    ```

## 2. Configuration

### 2.1 Inventory (`inventory.ini`)
Verify your nodes are listed correctly.
```ini
[rke2_servers]
master1 ansible_host=192.168.28.120
...
[rke2_agents]
worker1 ansible_host=192.168.28.130
# worker4 ... (commented out for now)
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

```bash
ansible-playbook site.yml -u dictator
```

**What happens?**
1.  **Common**: Disables swap, loads kernel modules, installs packages.
2.  **Server**:
    -   Installs RKE2 on Master 1.
    -   Starts `kube-vip` (HA).
    -   Installs RKE2 on Master 2 & 3 and joins them.
    -   Configures `cis-1.23` profile.
3.  **Agent**: Installs RKE2 on Workers and joins them to the cluster.

## 5. Verification

Log into `master1` (`ssh dictator@192.168.28.120`):

```bash
# Check Nodes
sudo /var/lib/rancher/rke2/bin/kubectl get nodes -o wide

# Check Pods (Cilium, CoreDNS)
sudo /var/lib/rancher/rke2/bin/kubectl get pods -A

# Check Local Kubeconfig
ls -l ~/.kube/config
```

## 6. Maintenance

- **Adding Worker 4**: Uncomment it in `inventory.ini` and re-run `site.yml`.
- **Upgrading**: Change `rke2_version` in `group_vars/all.yml` and re-run `site.yml`.
