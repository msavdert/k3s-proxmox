# Chapter 6: Storage Classes (Longhorn)

In this chapter, we will configure **Longhorn** as our distributed block storage provider. This ensures that application data is replicated across nodes, allowing pods to be rescheduled without data loss.

## 1. Prepare Secondary Disks (Workers only)

On each worker node (`10.0.0.11`, `10.0.0.12`, `10.0.0.13`), we need to format and mount the secondary 100GB disk (`scsi1`).

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
sudo mount -a
```

## 2. Install Required Dependencies

Longhorn requires `open-iscsi` and `nfs-common` to be installed on all nodes.

```bash
sudo apt update && sudo apt install -y open-iscsi nfs-common
```

## 3. Install Longhorn via Helm

Add the Longhorn repository and install it:

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace \
  --set defaultSettings.defaultDataPath="/var/lib/longhorn" \
  --set persistence.defaultClass=true
```

## 4. Verify Installation

Check the status of the Longhorn pods:

```bash
kubectl -n longhorn-system get pods
```

Verify that the `longhorn` storage class is now the default:

```bash
kubectl get sc
```

Output should show `longhorn (default)`.

---

> [!TIP]
> You can access the Longhorn UI by creating a temporary port-forward:
> `kubectl port-forward -n longhorn-system svc/longhorn-frontend 8080:80`
