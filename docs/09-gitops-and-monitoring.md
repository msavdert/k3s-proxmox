# Chapter 9: GitOps & Observability

In the final chapter of this guide, we establish the foundations for automated application delivery and cluster monitoring. We will install **ArgoCD** for GitOps and the **External Secrets Operator (ESO)** for secure secret management.

## 1. Install ArgoCD

ArgoCD will manage all applications in our cluster.

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Accessing ArgoCD UI

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Port-forward to access the UI:
```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

## 2. Install External Secrets Operator (ESO)

The External Secrets Operator allows you to fetch secrets from external providers (e.g., Infisical, 1Password, Vault) and sync them into Kubernetes.

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

## 3. (Planned) Observability Stack

A production cluster should be monitored. We recommend installing the **kube-prometheus-stack** via ArgoCD to get Prometheus, Grafana, and Alertmanager pre-configured for Kubernetes.

---

## 🏁 Conclusion

Congratulations! You have successfully built a "Hard Way" K3s cluster on Proxmox VE with:
- **Cilium** for eBPF networking and Gateway API ingress.
- **Longhorn** for distributed block storage.
- **ArgoCD** for GitOps automation.
- **ESO** for secure secret management.

This cluster is now ready to host production workloads in your homelab.
