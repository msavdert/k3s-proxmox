# Chapter 3: Joining K3s Worker Nodes

With our control plane initialized, we can now join our three worker nodes to the cluster.

## 1. Prerequisites

Before joining, ensure:
- The master node is running and accessible at `10.0.1.10`.
- You have the `K3S_TOKEN` retrieved in the previous chapter.

## 2. Join Worker Nodes

Execute the following on each worker node (`10.0.1.11`, `10.0.1.12`, `10.0.1.13`).

Replace `<K3S_TOKEN>` with the token from the master node.

### k3s-worker-1
```bash
ssh k3sadmin@10.0.1.11
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.1.10:6443 K3S_TOKEN=<K3S_TOKEN> sh -s - \
  --node-ip=10.0.1.11
```

### k3s-worker-2
```bash
ssh k3sadmin@10.0.1.12
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.1.10:6443 K3S_TOKEN=<K3S_TOKEN> sh -s - \
  --node-ip=10.0.1.12
```

### k3s-worker-3
```bash
ssh k3sadmin@10.0.1.13
curl -sfL https://get.k3s.io | K3S_URL=https://10.0.1.10:6443 K3S_TOKEN=<K3S_TOKEN> sh -s - \
  --node-ip=10.0.1.13
```

## 3. Verify Cluster Nodes

Return to the **master node** and check the node list:

```bash
kubectl get nodes
```

The output should show all four nodes in the `NotReady` state:

```text
NAME           STATUS     ROLES                  AGE     VERSION
k3s-master-1   NotReady   control-plane,master   10m     v1.30.x+k3s1
k3s-worker-1   NotReady   <none>                 2m      v1.30.x+k3s1
k3s-worker-2   NotReady   <none>                 1m      v1.30.x+k3s1
k3s-worker-3   NotReady   <none>                 30s     v1.30.x+k3s1
```

---

> [!NOTE]
> All nodes will remain `NotReady` until the Cilium CNI is installed in Chapter 5. This is normal and expected in a "Hard Way" installation.
