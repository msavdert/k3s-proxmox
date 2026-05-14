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

# Check for the latest versions
helm search repo cilium/cilium -l | head -n 5
```

> [!NOTE]
> **Chart Version** is the version of the Helm package, while **App Version** is the Cilium software version. They are usually identical for Cilium.

## 2.1 Install Gateway API CRDs
The Kubernetes Gateway API is not installed by default. You must install the CRDs **before** installing Cilium:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/experimental-install.yaml --server-side=true
```

## 3. Install Cilium & Hubble

We will install Cilium with eBPF-based kube-proxy replacement and enable **Hubble** for network observability.

```bash
helm install cilium cilium/cilium --version 1.19.4 \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=10.0.1.10 \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes \
  --set l2announcements.enabled=true \
  --set gatewayAPI.enabled=true \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set operator.replicas=1
```

**Check Status (Wait for ALL pods to be Ready)**

```bash
kubectl -n kube-system get pods -l k8s-app=cilium

cilium status --wait
```

## 4. Configure L2 Announcements Pool

Now we define the IP pool that Cilium will use for LoadBalancer services. We will use a range from our `k3snet` subnet. Run this on your **local machine**:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cilium.io/v2
kind: CiliumLoadBalancerIPPool
metadata:
  name: local-pool
spec:
  blocks:
    - cidr: "10.0.1.240/28" # Range: 10.0.1.240 - 10.0.1.254 (Outside DHCP range)
---
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-policy
spec:
  interfaces:
    - ^eth[0-9]+
  loadBalancerIPs: true
EOF
```

## 5. Verify Networking

Check that all nodes are now in the `Ready` state:

```bash
kubectl get nodes
```

## 6. Accessing Hubble UI
Once Cilium is ready, you can access the Hubble UI to visualize your network traffic:

```bash
# Start the port-forward from your local machine
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Open your browser and navigate to `http://localhost:12000`.

---

> [!IMPORTANT]
> The Gateway API requires the CRDs to be installed. Cilium usually handles this, but you can verify them with `kubectl get crd | grep gateway.networking.k8s.io`.
