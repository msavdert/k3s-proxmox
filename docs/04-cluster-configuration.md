# Chapter 4: Cluster Access and Configuration

To manage your cluster efficiently, you should configure access from your local machine using `kubectl`.

## 1. Retrieve the Kubeconfig

The `kubeconfig` file is located on the master node at `/etc/rancher/k3s/k3s.yaml`.

From your **local machine**, run the following command to download the file:

```bash
scp k3sadmin@10.0.1.10:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-proxmox.yaml
```

## 2. Update the Server Address

By default, the `kubeconfig` file points to `127.0.0.1`. You need to update it to the master node's IP address (`10.0.1.10`).

Open the file on your local machine and replace:
`server: https://127.0.0.1:6443`
with:
`server: https://10.0.1.10:6443`

Or use `sed`:
```bash
# On Linux:
sed -i 's/127.0.0.1/10.0.1.10/g' ~/.kube/k3s-proxmox.yaml

# On macOS:
sed -i '' 's/127.0.0.1/10.0.1.10/g' ~/.kube/k3s-proxmox.yaml
```

## 3. Configure Local Environment

Set the `KUBECONFIG` environment variable to use this new configuration:

```bash
export KUBECONFIG=~/.kube/k3s-proxmox.yaml

# To make it persistent, add it to your shell profile (~/.zshrc or ~/.bashrc)
echo "export KUBECONFIG=~/.kube/k3s-proxmox.yaml" >> ~/.zshrc
```

## 4. Test Access

Verify that you can communicate with the cluster from your local machine:

```bash
kubectl get nodes
```

---

> [!TIP]
> If you are using Tailscale to access your Proxmox network, ensure that your Tailscale client is connected and the routes to your networks (e.g., `10.0.0.0/24,10.0.1.0/24`) are accepted via the admin console and the `--advertise-routes` flag.
