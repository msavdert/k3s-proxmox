# Chapter 99: Cleanup and Reset

This chapter provides instructions for completely removing the K3s cluster and all associated Proxmox resources. Use this if you want to start from scratch or permanently decommission the environment.

> [!CAUTION]
> These actions are destructive and cannot be undone. Ensure you have backed up any important data before proceeding.

## 1. Stop and Destroy Virtual Machines

First, stop and remove all cluster nodes (Master and Workers). Replace the variables with your actual VM IDs if you didn't use the dynamic ones in Chapter 1.

```bash
# Option A: Manual cleanup (one by one)
# Replace with your actual IDs
qm stop <ID> && qm destroy <ID> --purge

# Option B: Efficient cleanup using a loop
# Example: If your VMs are IDs 105, 106, 107, and 108
for vmid in {105..108}; do
    echo "Cleaning up VM $vmid..."
    qm stop $vmid && qm destroy $vmid --purge
done
```

## 2. Remove the VM Template

Delete the base Ubuntu template:

```bash
qm destroy 9000 --purge
```

## 3. Clean Up Files and Snippets

Remove the Cloud-Init configuration files and temporary SSH keys from the Proxmox host:

```bash
# Remove Cloud-Init snippet
rm /var/lib/vz/snippets/k3s-cloud-init.yaml

# Remove downloaded Cloud Image (if still exists)
rm noble-server-cloudimg-amd64.img
```

## 4. Decommission the SDN (Network)

> [!WARNING]
> Only perform these steps if you are not using this SDN for other clusters.

1. Navigate to **Datacenter > SDN > VNets**.
2. Select `k3snet` and click **Remove**.
3. (Optional) If you want to remove the zone, go to **Datacenter > SDN > Zones**, select `localnat` and click **Remove**.
4. Click **Apply** in the SDN section to propagate the changes.

## 5. Verify Cleanup

Ensure no remnants of the cluster are left on the host:

```bash
# Verify no VMs are running
qm list

# Verify no custom bridges exist
ip a show k3snet
```

Your Proxmox environment is now clean and ready for a fresh start!
