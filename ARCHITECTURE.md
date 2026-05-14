# K3s on Proxmox: Master Architecture Blueprint

This document defines the absolute source of truth for the entire K3s cluster infrastructure. AI agents working on this project must refer to this document to understand the desired state before generating documentation, scripts, or Kubernetes manifests.

## 1. Infrastructure Layer (Proxmox VE)
- **Hypervisor:** Proxmox VE 9.x
- **Network Boundary:** Dedicated SDN VNet (`k3snet`) within the `localnat` zone.
- **Subnet:** `10.0.1.0/24` (Gateway: `10.0.1.1`, SNAT enabled).
- **VM Provisioning:** Proxmox CLI (`qm create`, `qm clone`, Cloud-Init).
- **Operating System:** Ubuntu 24.04 Cloud Image (`qemu-guest-agent` installed and enabled).

## 2. Node Topology
The cluster consists of 1 Control Plane node and 3 Worker nodes.

| Node Name | Role | vCPU | RAM | Disk 1 (OS) | Disk 2 (Storage) | IP Address |
|-----------|------|------|-----|-------------|------------------|------------|
| `k3s-master-1` | Control Plane | 2 | 4 GB | 15 GB | - | `10.0.1.10` |
| `k3s-worker-1` | Worker | 2 | 8 GB | 20 GB | 100 GB (Longhorn) | `10.0.1.11` |
| `k3s-worker-2` | Worker | 2 | 8 GB | 20 GB | 100 GB (Longhorn) | `10.0.1.12` |
| `k3s-worker-3` | Worker | 2 | 8 GB | 20 GB | 100 GB (Longhorn) | `10.0.1.13` |

## 3. Kubernetes Layer (K3s)
- **Distribution:** K3s
- **Control Plane:** Single Master (SQLite datastore).
- **Disabled Default Components:** 
  - `--disable flannel` (Replaced by Cilium)
  - `--disable traefik` (Replaced by Cilium Gateway API)
  - `--disable servicelb` (Replaced by Cilium L2 Announcements)
  - `--disable local-storage` (Replaced by Longhorn)

## 4. Networking & Ingress (Cilium)
- **CNI:** Cilium.
- **Kube-Proxy Replacement:** Enabled (Strict mode).
- **Load Balancer:** Cilium L2 Announcements (allocating IP addresses from a predefined pool in the `10.0.1.0/24` subnet, e.g., `10.0.1.200 - 10.0.1.207`).
- **Ingress Controller:** Cilium Gateway API (replacing traditional Ingress controllers like NGINX or Traefik).

## 5. Persistent Storage (Longhorn)
- **CSI:** Longhorn.
- **Architecture:** Distributed block storage.
- **Disk Usage:** The dedicated 100GB secondary disks attached to each worker node will be exclusively formatted (e.g., ext4/xfs) and mounted (e.g., `/var/lib/longhorn`) for Longhorn. The OS disk (20GB) is protected from filling up with application data.

## 6. GitOps & Secrets
- **GitOps Engine:** ArgoCD (App-of-Apps pattern).
- **Secrets Management:** External Secrets Operator (ESO). Secrets will be safely pulled from a vault and synced into Kubernetes without storing plain text in Git.
- **Observability:** (Planned) Kube-Prometheus-Stack.

## AI Agent Instructions
If you are an AI agent generating manifests or documentation based on this plan:
1. Always implement the `qm` CLI commands for VM creation and disk attachment in `docs/01-vm-provisioning.md`.
2. Ensure K3s install commands use the exact `--disable` flags listed above in `docs/02-k3s-initialization.md`.
3. Configure Cilium with Helm using strict kube-proxy replacement, L2 announcements, and Gateway API enabled in `docs/05-networking-and-ingress.md`.
4. Provide instructions to partition, format, and mount the 100GB secondary disk in the worker nodes before installing Longhorn in `docs/06-storage-classes.md`.
