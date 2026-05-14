# Chapter 2: K3s Control-Plane Initialization

In this chapter, we will initialize the Kubernetes control plane on our primary master node (`10.0.0.10`). According to our architecture blueprint, we will disable several default K3s components to replace them with more advanced alternatives later in the guide.

## 1. Connect to the Master Node

SSH into the master node using the user we configured via Cloud-Init:

```bash
ssh k3sadmin@10.0.0.10
```

## 2. Install K3s (Single Master)

We will use the official K3s installation script. We are explicitly disabling:
- **Flannel:** To be replaced by Cilium.
- **Traefik:** To be replaced by Cilium Gateway API.
- **ServiceLB:** To be replaced by Cilium L2 Announcements.
- **Local Storage:** To be replaced by Longhorn.

Run the following command:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --disable-cloud-controller \
  --disable flannel \
  --disable traefik \
  --disable servicelb \
  --disable local-storage \
  --flannel-backend=none \
  --node-ip=10.0.0.10 \
  --write-kubeconfig-mode 644
```

> [!NOTE]
> We use `--flannel-backend=none` because we are disabling Flannel and will install Cilium manually. This prevents K3s from trying to manage any CNI.

## 3. Verify the Installation

Check the status of the K3s service:

```bash
sudo systemctl status k3s
```

Verify that the node is registered but in a `NotReady` state (this is expected since no CNI is installed yet):

```bash
kubectl get nodes
```

Output should look like:
```text
NAME           STATUS     ROLES                  AGE   VERSION
k3s-master-1   NotReady   control-plane,master   10s   v1.30.x+k3s1
```

## 4. Retrieve the Cluster Token

You will need the node token to join worker nodes in the next chapter. Save this value:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

---

> [!WARNING]
> Do not attempt to install any pods yet. The cluster will remain in a `NotReady` state until we install Cilium in Chapter 5.
