# Chapter 5: Networking & Ingress (Cilium)

In this chapter, we install and configure **Cilium** as our CNI (Container Network Interface). We will leverage Cilium's advanced features, including strict kube-proxy replacement, L2 announcements for LoadBalancer services, and the Gateway API for ingress management.

## 1. Install Helm

We will use Helm to manage Cilium and other cluster components. Install it on your **local machine**:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## 2. Add Cilium Helm Repository

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
```

## 3. Install Cilium

We will install Cilium with the following configuration:
- `kubeProxyReplacement=true`: Replaces `kube-proxy` with eBPF for better performance and security.
- `l2announcements.enabled=true`: Allows Cilium to announce LoadBalancer IPs via ARP.
- `gatewayAPI.enabled=true`: Enables the Kubernetes Gateway API support.

```bash
helm install cilium cilium/cilium --version 1.16.x \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=10.0.1.10 \
  --set k8sServicePort=6443 \
  --set l2announcements.enabled=true \
  --set gatewayAPI.enabled=true \
  --set operator.replicas=1
```

## 4. Configure L2 Announcements Pool

Now we define the IP pool that Cilium will use for LoadBalancer services. We will use a range from our `k3snet` subnet.

Create a file named `cilium-l2-pool.yaml`:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: local-pool
spec:
  blocks:
    - cidr: "10.0.1.200/29" # Range: 10.0.1.200 - 10.0.1.207
---
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-policy
spec:
  interfaces:
    - ^eth0$
  loadBalancerIPs: true
```

Apply the configuration:
```bash
kubectl apply -f cilium-l2-pool.yaml
```

## 5. Verify Networking

Wait for all Cilium pods to be ready:

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
```

Check that all nodes are now in the `Ready` state:

```bash
kubectl get nodes
```

---

> [!IMPORTANT]
> The Gateway API requires the CRDs to be installed. Cilium usually handles this, but you can verify them with `kubectl get crd | grep gateway.networking.k8s.io`.
