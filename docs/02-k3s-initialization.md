# Chapter 2: K3s Control-Plane Initialization

In this chapter, we will initialize the Kubernetes control plane on our primary master node (`10.0.1.10`). According to our architecture blueprint, we will disable several default K3s components to replace them with more advanced alternatives later in the guide.

## 1. Connect to the Master Node

SSH into the master node using the user we configured via Cloud-Init:

```bash
ssh k3sadmin@10.0.1.10
```

> [!NOTE]
> System prerequisites like disabling UFW and Swap are now automatically handled by our Cloud-Init configuration from Chapter 1.

## 2. Install K3s with Embedded etcd

We will use the official K3s installation script. By default, K3s uses **SQLite** for its datastore. However, to ensure our cluster is ready for **High Availability (HA)** in the future, we will initialize it with an embedded **etcd** datastore.

> [!IMPORTANT]
> The `--cluster-init` flag is crucial. It tells K3s to use embedded etcd. Without this flag, K3s would default to SQLite, which is stored in a local file and cannot be easily shared across multiple master nodes for high availability.

We are also explicitly disabling:
- **Flannel:** To be replaced by Cilium.
- **Traefik:** To be replaced by Cilium Gateway API.
- **ServiceLB:** To be replaced by Cilium L2 Announcements.
- **Local Storage:** To be replaced by Longhorn.

Run the following command:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --disable-cloud-controller \
  --disable flannel \
  --disable traefik \
  --disable servicelb \
  --disable local-storage \
  --flannel-backend=none \
  --node-ip=10.0.1.10 \
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
NAME           STATUS     ROLES           AGE   VERSION
k3s-master-1   NotReady   control-plane   12s   v1.35.4+k3s1
```

**Verify the datastore (etcd):**
You can confirm that etcd is being used by checking the K3s database directory:
```bash
sudo ls -F /var/lib/rancher/k3s/server/db/
```
If you see an `etcd/` directory, the initialization was successful.

## 4. Retrieve the Cluster Token

You will need the node token to join worker nodes in the next chapter. Save this value:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

---

> [!WARNING]
> Do not attempt to install any pods yet. The cluster will remain in a `NotReady` state until we install Cilium in Chapter 5.
