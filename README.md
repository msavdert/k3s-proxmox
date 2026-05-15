# K3s on Proxmox: The Hard Way

This repository contains a comprehensive, step-by-step guide to deploying a production-ready K3s Kubernetes cluster on Proxmox VE. 
The guide is designed for homelab enthusiasts and professionals who want to understand the underlying mechanics of Kubernetes infrastructure provisioning, networking, and GitOps integration.

## 🛠 Technology Stack (May 2026)

| Component | Version | Purpose |
| :--- | :--- | :--- |
| **Hypervisor** | [Proxmox VE 9.1+](https://www.proxmox.com) | Bare-metal virtualization platform. |
| **Operating System** | [Ubuntu 26.04 LTS](https://ubuntu.com) | Lightweight, modern base OS with Cloud-Init support. |
| **Kubernetes Distro** | [K3s v1.35.x](https://k3s.io) | Production-grade, lightweight Kubernetes distribution. |
| **Networking (CNI)** | [Cilium 1.19.x](https://cilium.io) | eBPF-based networking, security, and observability (Hubble). |
| **Storage (CSI)** | [Longhorn 1.11.x](https://longhorn.io) | Distributed, resilient block storage for stateful applications. |
| **Certificates** | [cert-manager v1.20.x](https://cert-manager.io) | Automated TLS certificate issuance and renewal. |
| **GitOps** | [ArgoCD v2.14.x](https://argoproj.github.io/argo-cd) | Declarative, continuous delivery for Kubernetes. |
| **Secrets Management** | [External Secrets](https://external-secrets.io) | Syncing secrets from external providers (ESO). |
| **Database Operator** | [CloudNativePG v1.29.x](https://cloudnative-pg.io) | Cloud-native, HA PostgreSQL management and backups. |
| **Automation** | Cloud-Init & `qm` CLI | Automated VM provisioning and system preparation. |
| **Ingress/LB** | Cilium Gateway API | Modern L7 traffic management and L2 Service LB. |

## 🚀 Roadmap and Guide Structure

- [Master Architecture Blueprint](ARCHITECTURE.md)
- [Chapter 0: Infrastructure Provisioning (Proxmox Setup)](docs/00-proxmox-setup.md)
- [Chapter 1: VM Provisioning (Cloud-Init & Networking)](docs/01-vm-provisioning.md)
- [Chapter 2: K3s Control-Plane Initialization](docs/02-k3s-initialization.md)
- [Chapter 3: Joining K3s Worker Nodes](docs/03-k3s-worker-nodes.md)
- [Chapter 4: Cluster Access and Configuration](docs/04-cluster-configuration.md)
- [Chapter 5: Networking & Ingress (Cilium & Gateway API)](docs/05-networking-and-ingress.md)
- [Chapter 6: Certificate Management (cert-manager)](docs/06-certificate-management.md)
- [Chapter 7: Storage Classes (Longhorn)](docs/07-storage-classes.md)
- [Chapter 8: Exposing Services (Cilium Gateway API)](docs/08-exposing-services.md)
- [Chapter 9: GitOps & Observability (ArgoCD & ESO)](docs/09-gitops-and-monitoring.md)
- [Chapter 10: Database Layer (CloudNativePG)](docs/10-cloudnative-pg.md)
- [Chapter 99: Cleanup and Reset](docs/99-cleanup.md)

## 🛠️ Prerequisites

- A dedicated server or bare-metal machine (e.g., Hetzner).
- Minimum 4 cores, 16GB RAM for a decent cluster experience.
- Familiarity with Linux command line, SSH, and networking basics.

## 📄 License

MIT License. See [LICENSE](LICENSE) for more details.
