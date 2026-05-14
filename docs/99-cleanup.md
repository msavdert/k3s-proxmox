# Chapter 99: Cleanup and Reset

This chapter provides instructions for completely removing the K3s cluster and all associated Proxmox resources. Use this if you want to start from scratch or permanently decommission the environment.

> [!CAUTION]
> These actions are destructive and cannot be undone. Ensure you have backed up any important data before proceeding.

## 1. Stop and Destroy Virtual Machines

First, stop and remove all cluster nodes (Master and Workers). Replace the variables with your actual VM IDs if you didn't use the dynamic ones in Chapter 1.

```bash
# Stop the VMs
# Replace with your actual IDs (e.g., 100, 101, etc.)
qm stop <MASTER_ID>
qm stop <WORKER1_ID>
qm stop <WORKER2_ID>
qm stop <WORKER3_ID>

# Destroy the VMs
qm destroy <MASTER_ID> --purge
qm destroy <WORKER1_ID> --purge
qm destroy <WORKER2_ID> --purge
qm destroy <WORKER3_ID> --purge
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

# Remove SSH public key (if still exists)
rm k3s.pub

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
