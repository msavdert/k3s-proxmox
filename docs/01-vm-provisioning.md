# Chapter 1: VM Provisioning

This chapter covers the creation of the Virtual Machines (VMs) on Proxmox VE using the command-line interface (`qm`). We will use the Ubuntu 24.04 (Noble Numbat) Cloud Image and configure it using **Cloud-Init** for an automated setup, including the automatic installation of the QEMU Guest Agent.

## 1. Prepare the Ubuntu Cloud Image

Log in to your Proxmox host via SSH. Choose your preferred Ubuntu version.

### Option A: Ubuntu 26.04 LTS (Recommended)
```bash
# Download the Ubuntu 26.04 Cloud Image
wget https://cloud-images.ubuntu.com/releases/26.04/release/ubuntu-26.04-server-cloudimg-amd64.img -O ubuntu-server-cloudimg.img
```

### Option B: Ubuntu 24.04 LTS (Alternative)
```bash
# Use this if you prefer the 24.04 release
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img -O ubuntu-server-cloudimg.img
```

## 2. Create a Cloud-Init Custom Snippet

We will use a **Snippet** to handle all configurations. This allows us to define a reusable base configuration for all nodes in the cluster.

> [!IMPORTANT]
> When using `--cicustom`, Proxmox's standard `--ciuser` and `--cipassword` commands are ignored. We must define everything inside the snippet.

First, generate a hashed password for security:
```bash
openssl passwd -6
```
Copy the output hash (it looks like `$6$rounds=...`).

Now create the configuration:
```bash
# Create the snippets directory if it doesn't exist
mkdir -p /var/lib/vz/snippets

# Create the custom configuration
cat << 'EOF' > /var/lib/vz/snippets/k3s-cloud-init.yaml
#cloud-config
user: k3sadmin
password: <YOUR_HASHED_PASSWORD>
chpasswd: { expire: False }
ssh_authorized_keys:
  - <YOUR_SSH_PUBLIC_KEY_STRING>

package_upgrade: true
packages:
  - qemu-guest-agent
  - nfs-common
  - open-iscsi
  - dmsetup
  - jq
  - curl

runcmd:
  - systemctl disable systemd-networkd-wait-online.service
  - systemctl mask systemd-networkd-wait-online.service
  - systemctl enable --now qemu-guest-agent
  - systemctl enable --now iscsid
  - ufw disable
  - swapoff -a
  - sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
EOF
```

> [!NOTE]
> Disabling and Masking `systemd-networkd-wait-online.service` is a specific requirement for **Ubuntu 26.04** on Proxmox. Previous LTS versions (like 22.04 and 24.04) did not suffer from this boot-time network synchronization hang. 

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
qm create 9000 --name ubuntu-template --memory 2048 --cores 2 --cpu host --net0 virtio,bridge=k3snet

# Enable the QEMU Guest Agent
qm set 9000 --agent enabled=1

# Import the disk into Proxmox storage
# REPLACE 'local-zfs' with the storage Name you found in the previous step
qm importdisk 9000 ubuntu-server-cloudimg.img local-zfs

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
rm ubuntu-server-cloudimg.img
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

# Add Tags for better organization
qm set $MASTER_ID --tags "k3s;master;ubuntu"
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

# Add Tags
qm set $WORKER1_ID --tags "k3s;worker;ubuntu"
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

# Add Tags
qm set $WORKER2_ID --tags "k3s;worker;ubuntu"
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

# Add Tags
qm set $WORKER3_ID --tags "k3s;worker;ubuntu"
```

## 6. Start and Initialize Identity

Once all nodes are provisioned, start them. We will then use the **QEMU Guest Agent** (`qm guest exec`) to set their hostnames automatically from the Proxmox host.

```bash
# Start all VMs
qm start $MASTER_ID
qm start $WORKER1_ID
qm start $WORKER2_ID
qm start $WORKER3_ID
```

> [!TIP]
> You can monitor the Cloud-Init progress by connecting to the serial console of the master:
> `qm terminal $MASTER_ID`

```bash
# Wait 10-20 seconds for the Guest Agent to start, then set hostnames:
qm guest exec $MASTER_ID -- hostnamectl set-hostname k3s-master-1
qm guest exec $WORKER1_ID -- hostnamectl set-hostname k3s-worker-1
qm guest exec $WORKER2_ID -- hostnamectl set-hostname k3s-worker-2
qm guest exec $WORKER3_ID -- hostnamectl set-hostname k3s-worker-3

# Reboot all VMs
for vmid in {105..108}; do
    echo "Rebooting VM $vmid..."
    qm reboot $vmid
done
```

> [!TIP]
> Using `qm guest exec` allows us to finalize the node identity without needing to SSH into each machine manually. This is perfect for the "Hard Way" automation.

---

> [!IMPORTANT]
> Ensure that the `k3snet` (SDN) is properly applied in Proxmox before starting the VMs, or they will not receive network connectivity.
