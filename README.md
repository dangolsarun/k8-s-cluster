# RKE2 High Availability Cluster Deployment

This Ansible project automates the deployment of a highly available, CIS-hardened RKE2 (Rancher Kubernetes Engine 2) cluster with Cilium CNI.

## Architecture

- **Control Plane**: 3 Master Nodes (Stacked Etcd + API + Controller)
- **Data Plane**: 4 Worker Nodes
- **VIP**: `kube-vip` provides a floating IP for the HA Control Plane.
- **Networking**: Cilium CNI (replacing default Canal).
- **Security**: Hardened to `cis-1.23` profile by default.

## Prerequisites

- **Ansible 2.10+** installed on the controller machine.
- **SSH Access**: Passwordless SSH root/sudo access to all 7 nodes.
- **Operating System**: Ubuntu 20.04/22.04 LTS, Rocky Linux 9, or RHEL 8/9.
- **Resources**:
  - Masters: 2 vCPU, 4GB RAM minimum (4 vCPU, 8GB recommended).
  - Workers: Depends on workload.

## Configuration

### 1. Inventory (`inventory.ini`)
Update the IP addresses for your masters and workers.
```ini
[rke2_servers]
master-1 ansible_host=10.0.0.10
master-2 ansible_host=10.0.0.11
master-3 ansible_host=10.0.0.12

[rke2_agents]
worker-1 ansible_host=10.0.0.20
...
```

### 2. Global Variables (`group_vars/all.yml`)
- `cluster_vip`: Set this to a free IP in your subnet. This is crucial for HA.
- `rke2_token`: Set a strong shared secret.
- `rke2_cni`: Default is `cilium`.

## Usage

Test connectivity to your nodes:
```bash
ansible -m ping all
```

Run the deployment playbook:
```bash
ansible-playbook site.yml
```

## Verification

After deployment, log into `master-1`:

1. **Check Nodes**:
   ```bash
   kubectl get nodes
   ```
2. **Check Pods**:
   ```bash
   kubectl get pods -A
   ```
3. **Verify Cilium**:
   ```bash
   cilium status
   ```
   (Requires `cilium-cli` installed manually, or check pods in `kube-system`).

## Security (CIS)

The cluster is deployed with `profile: "cis-1.23"`. This enforces:
- Strict pod security policies (if enabled).
- Kernel parameter tuning (handled by `common` role).
- Etcd encryption and user restrictions.

## Troubleshooting

- **API Server won't start**: Check `journalctl -u rke2-server`.
- **VIP not assigned**: Check `kubectl -n kube-system logs -l name=kube-vip`. Ensure the IP is not in use.
