# RKE2 Homelab Deployment Summary

## Configuration Overview

### Infrastructure

- **Operating System**: Ubuntu
- **Master Nodes**: 3 nodes (10.10.1.30, 10.10.1.31, 10.10.1.32)
- **Worker Nodes**: 3 nodes (10.10.1.33, 10.10.1.34, 10.10.1.35)
- **Total Nodes**: 6

### High Availability Configuration

- **HA Mode**: Enabled
- **VIP Management**: Keepalived
- **VIP Address**: 10.10.1.30
- **API Port**: 6443

### Network Configuration

- **CNI**: Cilium (with kube-proxy replacement)
- **Pod Network (Cluster CIDR)**: 10.10.1.0/16
- **Service Network (Service CIDR)**: 172.16.0.0/16
- **Default CNI Disabled**: rke2-canal

### RKE2 Configuration

- **Version**: v1.34.2+rke2r1
- **Architecture**: amd64
- **Cloud Controller**: Disabled (external)
- **SELinux**: Disabled
- **CIS Profile**: Disabled

### Cilium Features

- **Kube-proxy Replacement**: Strict
- **Hubble**: Enabled (with relay and UI)
- **Local Redirect Policy**: Enabled
- **Auto Direct Node Routes**: Enabled
- **CNI Chaining Mode**: None

### Authentication

- **SSH User**: rykelley
- **SSH Key**: ~/workspace/Mac Computer/sk/ssh/rkelley-prikey
- **Sudo**: Enabled with password

## Deployment Steps

### 1. Test Connectivity

```bash
ansible -i inventory/hosts.ini all -m ping
```

### 2. Run Baseline Configuration

This will:
- Install required packages (open-iscsi, nfs-common, cryptsetup, curl, wget)
- Start iSCSI service
- Configure DNS settings
- Restart NetworkManager

```bash
ansible-playbook -i inventory/hosts.ini baseline.yml
```

### 3. Deploy RKE2 Cluster

This will:
- Install keepalived on master nodes
- Configure VIP (10.10.1.30) with automatic failover
- Install RKE2 v1.34.2+rke2r1
- Configure Cilium CNI
- Setup HA etcd cluster
- Download kubeconfig to /tmp/rke2.yaml

```bash
ansible-playbook -i inventory/hosts.ini homelab.yaml
```

### 4. Access Your Cluster

```bash
export KUBECONFIG=/tmp/rke2.yaml
kubectl get nodes
kubectl get pods -A
```

## Post-Deployment Verification

### Check Cluster Health

```bash
kubectl get nodes
kubectl get pods -n kube-system
```

### Verify Cilium

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium
```

### Check Hubble

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=hubble
```

### Verify VIP Failover

```bash
# On master nodes, check keepalived status
ssh rykelley@10.10.1.30 "sudo systemctl status keepalived"
ssh rykelley@10.10.1.31 "sudo systemctl status keepalived"
ssh rykelley@10.10.1.32 "sudo systemctl status keepalived"

# Check which node has the VIP
ssh rykelley@10.10.1.30 "ip addr show | grep 10.10.1.30"
ssh rykelley@10.10.1.31 "ip addr show | grep 10.10.1.30"
ssh rykelley@10.10.1.32 "ip addr show | grep 10.10.1.30"
```

## Important Files

- **Inventory**: `inventory/hosts.ini`
- **Credentials**: `inventory/group_vars/all.yml`
- **Tokens**: `inventory/homelab/group_vars/all.yaml`
- **Playbooks**: `baseline.yml`, `homelab.yaml`
- **Cilium Config**: `manifests/rke2-cilium-config.yaml`
- **Ansible Config**: `ansible.cfg`

## Network Requirements

### Required Ports (must be open between nodes)

**Master Nodes:**
- 6443/tcp - Kubernetes API
- 9345/tcp - RKE2 supervisor API
- 10250/tcp - Kubelet
- 2379-2380/tcp - etcd
- 112/tcp - VRRP (for keepalived)

**Worker Nodes:**
- 10250/tcp - Kubelet
- 8472/udp - Cilium VXLAN (if using tunnel mode)

**All Nodes:**
- 4240/tcp - Hubble server
- 4244/tcp - Hubble relay

## Troubleshooting

### Check RKE2 Service Status

```bash
# On master nodes
sudo systemctl status rke2-server

# On worker nodes
sudo systemctl status rke2-agent
```

### View RKE2 Logs

```bash
# Master nodes
sudo journalctl -u rke2-server -f

# Worker nodes
sudo journalctl -u rke2-agent -f
```

### Check Keepalived Logs

```bash
sudo journalctl -u keepalived -f
```

### Verify Network Connectivity

```bash
# From any master to other masters
nc -zv 10.10.1.31 6443
nc -zv 10.10.1.32 6443
```

## Upgrade Process

To upgrade RKE2:
1. Update `rke2_version` in `homelab.yaml`
2. Re-run the playbook: `ansible-playbook -i inventory/hosts.ini homelab.yaml`
3. The role will perform a rolling upgrade

## Security Notes

⚠️ **Important**: The credentials in `inventory/group_vars/all.yml` are stored in plain text. Consider encrypting them with Ansible Vault:

```bash
ansible-vault encrypt inventory/group_vars/all.yml
```

When running playbooks with encrypted files:
```bash
ansible-playbook -i inventory/hosts.ini homelab.yaml --ask-vault-pass
```
