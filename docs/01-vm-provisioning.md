# Chapter 1: VM Provisioning

This chapter covers the creation of the Virtual Machines (VMs) on Proxmox VE using the command-line interface (`qm`). We will use the Ubuntu 24.04 (Noble Numbat) Cloud Image as our base and configure it using Cloud-Init for a seamless, automated setup.

## 1. Prepare the Ubuntu Cloud Image

First, we need to download the official Ubuntu 24.04 Cloud Image and prepare it for use as a Proxmox template.

Log in to your Proxmox host via SSH and execute the following:

```bash
# Install required tools
apt update && apt install -y libguestfs-tools

# Download the Ubuntu 24.04 Cloud Image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# (Optional) Customize the image - install qemu-guest-agent
# This ensures Proxmox can communicate with the VM for IP reporting and graceful shutdowns
virt-customize -a noble-server-cloudimg-amd64.img --install qemu-guest-agent
```

## 2. Create the VM Template

We will create a base VM template (ID `9000`) that all our cluster nodes will be cloned from.

```bash
# Create the VM
qm create 9000 --name ubuntu-2404-template --memory 2048 --cores 2 --net0 virtio,bridge=vnet0

# Import the disk to Proxmox storage (replace 'local-zfs' with your storage name)
qm importdisk 9000 noble-server-cloudimg-amd64.img local-zfs

# Attach the disk to the VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-zfs:vm-9000-disk-0

# Add Cloud-Init drive
qm set 9000 --ide2 local-zfs:cloudinit

# Configure boot order
qm set 9000 --boot c --bootdisk scsi0

# Enable Serial Console (important for Cloud-Init debugging)
qm set 9000 --serial0 socket --vga serial0

# Convert to template
qm template 9000
```

## 3. Provision the Control Plane Node

Now we clone the template to create our single master node.

```bash
# Clone the template
qm clone 9000 100 --name k3s-master-1 --full

# Configure resources (Master: 2 vCPU, 4GB RAM, 15GB Disk)
qm set 100 --memory 4096 --cores 2
qm disk resize 100 scsi0 15G

# Configure Cloud-Init
# Replace <YOUR_PUBLIC_SSH_KEY> with your actual public key
qm set 100 --ciuser k3sadmin --cipassword <YOUR_SECURE_PASSWORD> --sshkeys <YOUR_PUBLIC_SSH_KEY_FILE_OR_STRING>
qm set 100 --ipconfig0 ip=10.0.0.10/24,gw=10.0.0.1
```

## 4. Provision the Worker Nodes

We will create 3 worker nodes. Each worker will have a secondary 100GB disk for Longhorn.

### k3s-worker-1 (ID 101)
```bash
qm clone 9000 101 --name k3s-worker-1 --full
qm set 101 --memory 8192 --cores 2
qm disk resize 101 scsi0 20G
# Attach secondary 100GB disk for Longhorn
# Replace 'local-zfs' with your target storage
pvesm alloc local-zfs 101 vm-101-disk-1 100G
qm set 101 --scsi1 local-zfs:vm-101-disk-1
qm set 101 --ipconfig0 ip=10.0.0.11/24,gw=10.0.0.1
qm set 101 --ciuser k3sadmin --cipassword <YOUR_SECURE_PASSWORD> --sshkeys <YOUR_PUBLIC_SSH_KEY>
```

### k3s-worker-2 (ID 102)
```bash
qm clone 9000 102 --name k3s-worker-2 --full
qm set 102 --memory 8192 --cores 2
qm disk resize 102 scsi0 20G
pvesm alloc local-zfs 102 vm-102-disk-1 100G
qm set 102 --scsi1 local-zfs:vm-102-disk-1
qm set 102 --ipconfig0 ip=10.0.0.12/24,gw=10.0.0.1
qm set 102 --ciuser k3sadmin --cipassword <YOUR_SECURE_PASSWORD> --sshkeys <YOUR_PUBLIC_SSH_KEY>
```

### k3s-worker-3 (ID 103)
```bash
qm clone 9000 103 --name k3s-worker-3 --full
qm set 103 --memory 8192 --cores 2
qm disk resize 103 scsi0 20G
pvesm alloc local-zfs 103 vm-103-disk-1 100G
qm set 103 --scsi1 local-zfs:vm-103-disk-1
qm set 103 --ipconfig0 ip=10.0.0.13/24,gw=10.0.0.1
qm set 103 --ciuser k3sadmin --cipassword <YOUR_SECURE_PASSWORD> --sshkeys <YOUR_PUBLIC_SSH_KEY>
```

## 5. Start the VMs

Once all nodes are configured, start them up:

```bash
qm start 100
qm start 101
qm start 102
qm start 103
```

> [!TIP]
> You can monitor the Cloud-Init progress by connecting to the serial console:
> `qm terminal 100`

---

> [!IMPORTANT]
> Ensure that the `vnet0` (SDN) is properly applied in Proxmox before starting the VMs, or they will not receive network connectivity.
