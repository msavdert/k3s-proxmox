# Chapter 1: VM Provisioning

This chapter covers the creation of the Virtual Machines (VMs) on Proxmox VE using the command-line interface (`qm`). We will use the Ubuntu 24.04 (Noble Numbat) Cloud Image and configure it using **Cloud-Init** for an automated setup, including the automatic installation of the QEMU Guest Agent.

## 1. Prepare the Ubuntu Cloud Image

Log in to your Proxmox host via SSH. We will download the Ubuntu Cloud Image to our temporary workspace (`/root`). 

> [!NOTE]
> Unlike an ISO file (which stays in `/var/lib/vz/template/iso/`), a Cloud Image is a pre-installed disk. We download it temporarily, import it into Proxmox's storage, and then delete the source file.

```bash
# Download the Ubuntu 24.04 Cloud Image
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

## 2. Prepare SSH Credentials

Before configuring the template, you need an SSH key pair. If you don't have one, generate a modern **Ed25519** key:

```bash
# Generate a new SSH key pair (Run this on your LOCAL machine or Proxmox host)
ssh-keygen -t ed25519 -C "k3s-proxmox-key" -f id_ed25519_k3s
```
Your public key will be in `id_ed25519_k3s.pub`. You will need its content in the next step.

## 3. Create a Cloud-Init Custom Snippet

We will use a **Snippet**. Snippets are custom configuration files stored in `/var/lib/vz/snippets/`. They allow us to extend Cloud-Init beyond the basic GUI options, such as installing packages on the first boot.

```bash
# Create the snippets directory if it doesn't exist
mkdir -p /var/lib/vz/snippets

# Create the custom configuration
cat << EOF > /var/lib/vz/snippets/k3s-cloud-init.yaml
#cloud-config
package_upgrade: true
packages:
  - qemu-guest-agent
  - nfs-common
  - open-iscsi
runcmd:
  - systemctl enable --now qemu-guest-agent
EOF
```

## 3. Create the VM Template

Before creating the template, you must identify where Proxmox will store the VM disks. 

### Discover Your Storage ID
Run the following command to see which storage supports VM **images**:

**Manual Method:**
```bash
pvesh get /cluster/resources --type storage --output yaml | grep images -A 9 | grep 'storage:'
```

**Automated Method (One-liner):**
```bash
pvesh get /cluster/resources --type storage --output-format json | python3 -c "import sys, json; print('\n'.join([s['storage'] for s in json.load(sys.stdin) if 'images' in s['content']]))"
```

Look for the storage ID that includes `images` in its content. Note this name (e.g., `local-zfs`).

### Initialize the Template
We will create a base VM template (ID `9000`). Once the disk is imported, the source `.img` file in `/root` is no longer needed.

```bash
# Create the VM
qm create 9000 --name ubuntu-2404-template --memory 2048 --cores 2 --net0 virtio,bridge=k3snet

# Enable the QEMU Guest Agent
qm set 9000 --agent enabled=1

# Configure Shared Cloud-Init Settings (Inherited by all clones)
# Use the public key we generated (or your existing one)
# If you generated a new one: cat id_ed25519_k3s.pub > k3s.pub
# If you have one: echo "ssh-ed25519 ..." > k3s.pub
qm set 9000 --ciuser k3sadmin --cipassword <YOUR_SECURE_PASSWORD> --sshkeys k3s.pub
rm k3s.pub

# Import the disk into Proxmox storage
# REPLACE 'local-zfs' with the storage Name you found in the previous step
qm importdisk 9000 noble-server-cloudimg-amd64.img local-zfs

# Attach the imported disk to the VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-zfs:vm-9000-disk-0

# Add Cloud-Init drive
qm set 9000 --ide2 local-zfs:cloudinit

# Attach the custom Cloud-Init snippet
qm set 9000 --cicustom "user=local:snippets/k3s-cloud-init.yaml"

# Configure boot order
qm set 9000 --boot c --bootdisk scsi0

# Enable Serial Console
qm set 9000 --serial0 socket --vga serial0

# Convert to template
qm template 9000

# CLEANUP: Remove the temporary image file
rm noble-server-cloudimg-amd64.img
```

## 4. Provision the Control Plane Node

We will now clone the template. It will inherit the user, password, and SSH keys automatically.

```bash
# Get the next available VM ID
MASTER_ID=$(pvesh get /cluster/nextid)

# Clone the template
qm clone 9000 $MASTER_ID --name k3s-master-1 --full

# Configure node-specific resources
qm set $MASTER_ID --memory 4096 --cores 2
qm disk resize $MASTER_ID scsi0 15G

# Configure node-specific Network (IP must be unique)
qm set $MASTER_ID --ipconfig0 ip=10.0.1.10/24,gw=10.0.1.1
```

## 5. Provision the Worker Nodes

We will create 3 worker nodes. We will use the same `k3s.pub` file for all nodes.

### k3s-worker-1
```bash
# Get next ID
WORKER1_ID=$(pvesh get /cluster/nextid)

qm clone 9000 $WORKER1_ID --name k3s-worker-1 --full
qm set $WORKER1_ID --memory 8192 --cores 2
qm disk resize $WORKER1_ID scsi0 20G
# Attach secondary 100GB disk
pvesm alloc local-zfs $WORKER1_ID vm-$WORKER1_ID-disk-1 100G
qm set $WORKER1_ID --scsi1 local-zfs:vm-$WORKER1_ID-disk-1
qm set $WORKER1_ID --ipconfig0 ip=10.0.1.11/24,gw=10.0.1.1
```

### k3s-worker-2
```bash
# Get next ID
WORKER2_ID=$(pvesh get /cluster/nextid)

qm clone 9000 $WORKER2_ID --name k3s-worker-2 --full
qm set $WORKER2_ID --memory 8192 --cores 2
qm disk resize $WORKER2_ID scsi0 20G
pvesm alloc local-zfs $WORKER2_ID vm-$WORKER2_ID-disk-1 100G
qm set $WORKER2_ID --scsi1 local-zfs:vm-$WORKER2_ID-disk-1
qm set $WORKER2_ID --ipconfig0 ip=10.0.1.12/24,gw=10.0.1.1
```

### k3s-worker-3
```bash
# Get next ID
WORKER3_ID=$(pvesh get /cluster/nextid)

qm clone 9000 $WORKER3_ID --name k3s-worker-3 --full
qm set $WORKER3_ID --memory 8192 --cores 2
qm disk resize $WORKER3_ID scsi0 20G
pvesm alloc local-zfs $WORKER3_ID vm-$WORKER3_ID-disk-1 100G
qm set $WORKER3_ID --scsi1 local-zfs:vm-$WORKER3_ID-disk-1
qm set $WORKER3_ID --ipconfig0 ip=10.0.1.13/24,gw=10.0.1.1
```

## 6. Start the VMs

Once all nodes are provisioned, start them using their respective variables:

```bash
qm start $MASTER_ID
qm start $WORKER1_ID
qm start $WORKER2_ID
qm start $WORKER3_ID
```

# CLEANUP: Remove the public key file
rm k3s.pub

> [!TIP]
> You can monitor the Cloud-Init progress by connecting to the serial console of the master:
> `qm terminal $MASTER_ID`

---

> [!IMPORTANT]
> Ensure that the `k3snet` (SDN) is properly applied in Proxmox before starting the VMs, or they will not receive network connectivity.
