# Chapter 3: Joining K3s Worker Nodes

With our control plane initialized, we can now join our three worker nodes to the cluster.

## 1. Prerequisites

Before joining, ensure:
- The master node is running and accessible at `10.0.1.10`.
- You have the `K3S_TOKEN` retrieved in the previous chapter.

## 2. Join Worker Nodes

### 2.1 System Preparation (Run on all Workers)

Before joining the cluster, perform the same system preparation as on the master node:

```bash
# Run on k3s-worker-1, k3s-worker-2, and k3s-worker-3
sudo ufw disable
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

## 3. Joining the Nodes

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
k3s-master-1   NotReady   control-plane,master   10m     v1.35.4+k3s1
k3s-worker-1   NotReady   <none>                 2m      v1.35.4+k3s1
k3s-worker-2   NotReady   <none>                 1m      v1.35.4+k3s1
k3s-worker-3   NotReady   <none>                 30s     v1.35.4+k3s1
```

---

## 4. Cleanup and Reset (Troubleshooting)

If you encounter issues during the join process and need to start over on a worker node, follow these steps to completely remove the K3s agent:

```bash
# 1. Run the uninstall script
sudo /usr/local/bin/k3s-agent-uninstall.sh

# 2. (Optional) Clean up remaining K3s data directories
sudo rm -rf /var/lib/rancher/k3s
sudo rm -rf /etc/rancher/k3s

# 3. Reboot the node to ensure all network interfaces (like flannel/cilium) are cleared
sudo reboot
```

After rebooting, you can re-run the `curl` command in **Step 3** to try joining again.

---

> [!NOTE]
> All nodes will remain `NotReady` until the Cilium CNI is installed in Chapter 5. This is normal and expected in a "Hard Way" installation.
