# Chapter 7: Storage Classes (Longhorn)

In this chapter, we will configure **Longhorn** as our distributed block storage provider. This ensures that application data is replicated across nodes, allowing pods to be rescheduled without data loss.

## 1. Prepare Secondary Disks (Workers only)

On each worker node (`10.0.1.11`, `10.0.1.12`, `10.0.1.13`), we need to format and mount the secondary 100GB disk (`scsi1`).

### Identify and Format the Disk

```bash
# Verify the disk name (usually /dev/sdb)
lsblk

# Create a filesystem (XFS is recommended for Longhorn)
sudo mkfs.xfs /dev/sdb
```

### Mount the Disk

We will mount the disk to `/var/lib/longhorn` and ensure it's persistent after reboots.

```bash
sudo mkdir -p /var/lib/longhorn
echo "/dev/sdb /var/lib/longhorn xfs defaults 0 0" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```

## 2. Verify and Prepare Required Dependencies

Longhorn requires several host-level utilities. We automated the installation via Cloud-Init in Chapter 1, but we must ensure the services are active on **all nodes**.

```bash
# Verify all dependencies are present (installed via Cloud-Init)
sudo dpkg -l | grep -E "open-iscsi|nfs-common|dmsetup|jq|curl"
```

## 3. Install Longhorn via Helm

We will install Longhorn in its own namespace. We are using specific flags for K3s compatibility and production stability.

### Add Helm Repository
```bash
helm repo add longhorn https://charts.longhorn.io --force-update

# Check for the latest versions
helm search repo longhorn/longhorn -l | head -n 5
```

### Advanced Installation
We ensure the default storage class is set and point Longhorn to our dedicated disk.

```bash
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version 1.11.2 \
  --set persistence.defaultClass=true \
  --set defaultSettings.defaultDataPath=/var/lib/longhorn \
  --set defaultSettings.backupTarget="" \
  --set defaultSettings.replicaSoftAntiAffinity=false \
  --set defaultSettings.storageOverProvisioningPercentage=200
```

## 4. Verify Installation
Check that all Longhorn pods are running:

```bash
kubectl -n longhorn-system get pods
```

Wait until all pods are in the `Running` state. This might take 2-3 minutes as it needs to deploy the CSI drivers to all nodes.

## 5. Access Longhorn UI
You can access the dashboard by port-forwarding the `longhorn-frontend` service:

```bash
kubectl port-forward -n longhorn-system svc/longhorn-frontend 13000:80
```

Open your browser and navigate to `http://localhost:13000`.

### Best Practice Tip: Configure Backup Target
In the Longhorn UI, go to **Settings > General** and configure a **Backup Target** (e.g., S3 or NFS) to ensure your volume snapshots are stored off-cluster.

---

> [!IMPORTANT]
> Longhorn volumes require at least **3 nodes** to satisfy the default replication policy. Since we have 3 worker nodes, you have full high availability.

Verify that the `longhorn` storage class is now the default:

```bash
kubectl get sc
```

Output should show `longhorn (default)`.
