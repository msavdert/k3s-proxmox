# K3s on Proxmox: The Hard Way

This repository contains a comprehensive, step-by-step guide to deploying a production-ready K3s Kubernetes cluster on Proxmox VE. 
The guide is designed for homelab enthusiasts and professionals who want to understand the underlying mechanics of Kubernetes infrastructure provisioning, networking, and GitOps integration.

## 🚀 Roadmap and Guide Structure

- [Master Architecture Blueprint](ARCHITECTURE.md)
- [Chapter 0: Infrastructure Provisioning (Proxmox Setup)](docs/00-proxmox-setup.md)
- [Chapter 1: VM Provisioning (Cloud-Init & Networking)](docs/01-vm-provisioning.md)
- [Chapter 2: K3s Control-Plane Initialization](docs/02-k3s-initialization.md)
- [Chapter 3: Joining K3s Worker Nodes](docs/03-k3s-worker-nodes.md)
- [Chapter 4: Cluster Access and Configuration](docs/04-cluster-configuration.md)
- [Chapter 5: Networking & Ingress (Cilium & Gateway API)](docs/05-networking-and-ingress.md)
- [Chapter 6: Storage Classes (Longhorn)](docs/06-storage-classes.md)
- [Chapter 7: GitOps & Observability (ArgoCD & ESO)](docs/07-gitops-and-monitoring.md)
- [Chapter 99: Cleanup and Reset](docs/99-cleanup.md)

## 🛠️ Prerequisites

- A dedicated server or bare-metal machine (e.g., Hetzner).
- Minimum 4 cores, 16GB RAM for a decent cluster experience.
- Familiarity with Linux command line, SSH, and networking basics.

## 🤖 AI Agent Guidelines

This project is built and maintained with the help of AI agents. If you are an AI assistant working on this repository, please strictly follow the guidelines established in [`AGENTS.md`](AGENTS.md).

## 📄 License

MIT License. See [LICENSE](LICENSE) for more details.
