# Installing Proxmox VE 9.x

## Downloading Proxmox VE

```sh
ISO_VERSION=$(curl -s 'http://download.proxmox.com/iso/' | grep -oP 'proxmox-ve_(\d+.\d+-\d).iso' | sort -V | tail -n1)
ISO_URL="http://download.proxmox.com/iso/$ISO_VERSION"
curl $ISO_URL -o /tmp/proxmox-ve.iso

echo $ISO_VERSION
proxmox-ve_9.1-1.iso
```

## Acquire Network Configuration

```sh
INTERFACE_NAME=$(udevadm info -q property /sys/class/net/eth0 | grep "ID_NET_NAME_PATH=" | cut -d'=' -f2)
IP_CIDR=$(ip addr show eth0 | grep "inet\b" | awk '{print $2}')
GATEWAY=$(ip route | grep default | awk '{print $3}')
IP_ADDRESS=$(echo "$IP_CIDR" | cut -d'/' -f1)
CIDR=$(echo "$IP_CIDR" | cut -d'/' -f2)

echo $INTERFACE_NAME
enp0s31f6

echo $IP_CIDR
<PUBLIC_IP>/<CIDR_PREFIX>

echo $GATEWAY
<PUBLIC_GATEWAY_IP>

echo $IP_ADDRESS
<PUBLIC_IP>

echo $CIDR
<CIDR_PREFIX>
```

## Initiate QEMU and start the Proxmox installation

**UEFI**

```sh
apt-get install -y ovmf
# Get the primary and secondary disks
PRIMARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 1p | awk '{print $1}')
SECONDARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 2p | awk '{print $1}')
# Kick off QEMU with CDROM
qemu-system-x86_64 -daemonize -enable-kvm -m 10240 -k en-us \
-drive file=/dev/$PRIMARY_DISK,format=raw,media=disk,if=virtio,id=$PRIMARY_DISK \
-drive file=/dev/$SECONDARY_DISK,format=raw,media=disk,if=virtio,id=$SECONDARY_DISK \
-drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,readonly=on \
-drive file=/usr/share/OVMF/OVMF_VARS.fd,if=pflash,format=raw \
-cdrom /tmp/proxmox-ve.iso -boot d \
-vnc :0,password=on -monitor telnet:127.0.0.1:4444,server,nowait

# Both (UEFI & Legacy)
echo "change vnc password <YOUR_VNC_PASSWORD>" | nc -q 1 127.0.0.1 4444
```

Macos > Finder > CMD+k

```sh
vnc://<PUBLIC_IP>:5900
```

## Installing Proxmox via the GUI

Once connected select Install Proxmox VE (Graphical), you will see a message saying “No support for hardware-accelerated KVM virtualization detected” this is safe to ignore as it’s because we are running it via QEMU at this time. Click OK to continue.

Target > Options > ZFS RAID0 (to use both disk)

Country: United States
Time zone: UTC
Keyboard: U.S. English
Password: <YOUR_SECURE_PASSWORD>
Email: <YOUR_EMAIL>
Hostname: pve.lan

Untick “Automatically reboot after successful installation” and press Install
Once installation is complete, do not press Reboot, instead go back to the SSH terminal and enter the following command to close QEMU

```sh
# Stop QEMU
printf "quit\n" | nc 127.0.0.1 4444
```

Now we need to configure QEMU again but this time to boot from the internal drive and not from our Proxmox ISO

**UEFI**

```sh
# Kick off QEMU without CDROM
qemu-system-x86_64 -daemonize -enable-kvm -m 10240 -k en-us \
-drive file=/dev/$PRIMARY_DISK,format=raw,media=disk,if=virtio,id=$PRIMARY_DISK \
-drive file=/dev/$SECONDARY_DISK,format=raw,media=disk,if=virtio,id=$SECONDARY_DISK \
-drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,readonly=on \
-drive file=/usr/share/OVMF/OVMF_VARS.fd,if=pflash,format=raw \
-vnc :0,password=on -monitor telnet:127.0.0.1:4444,server,nowait \
-net user,hostfwd=tcp::2222-:22 -net nic

# Both (UEFI & Legacy)
echo "change vnc password <YOUR_VNC_PASSWORD>" | nc -q 1 127.0.0.1 4444
```

Macos > Finder > CMD+k

```sh
vnc://<PUBLIC_IP>:5900
```

Once Proxmox has booted return to the SSH client and install sshpass

```sh
apt-get -y install sshpass
```

Next we need to configure the network interfaces to work correctly once you reboot to baremetal Proxmox.

Enter the following commands replacing `<ROOT_PASSWORD>` with the password you set for root during the Proxmox GUI installation

```sh
cat > /tmp/proxmox_network_config << EOF
auto lo
iface lo inet loopback
iface $INTERFACE_NAME inet manual
auto vmbr0
iface vmbr0 inet static
  address $IP_ADDRESS/$CIDR
  gateway $GATEWAY
  bridge_ports $INTERFACE_NAME
  bridge_stp off
  bridge_fd 0
EOF
# transfer the network configuration file to Proxmox VE system
sshpass -p "<ROOT_PASSWORD>" scp -o StrictHostKeyChecking=no -P 2222 /tmp/proxmox_network_config root@localhost:/etc/network/interfaces
# update the nameserver
sshpass -p "<ROOT_PASSWORD>" ssh -o StrictHostKeyChecking=no -p 2222 root@localhost "sed -i 's/nameserver.*/nameserver 1.1.1.1/' /etc/resolv.conf"
```

Now we can inform QEMU to gracefully shutdown the VM with the following commands

```sh
printf "system_powerdown\n" | nc 127.0.0.1 4444
```

We are now ready to boot directly to Proxmox, restart the server so it exits the Rescue System and continue to the next section of this tutorial.

```sh
shutdown -r now
# SSH back into the server using your new <ROOT_PASSWORD>
```

# Configuring Proxmox

Now you should be able to access Proxmox from your browser using your https://<PUBLIC_IP>:8006

## Running the community Proxmox VE Post Install script

[PVE Post Install | Proxmox VE Helper Scripts](https://community-scripts.org/scripts/post-pve-install?id=post-pve-install)

From the web UI select your host on the left and click “Shell” at the top right.

```sh
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Say “disable” to disabling the “pve-enterprise” repository (unless you have a license)
Say “disable” to correcting the ‘ceph package sources (unless you have a license)
Say “yes” to enabling the “pve-no-subscription” repository (unless you have a license)
Say “yes” or “no” to adding the “pvetest” repository (user decision)
Say “yes” to disabling the subscription nag (unless you have a license)
Say “no” to disabling high availability unless you don’t require it
Say “yes” to updating Proxmox
Say “yes” to rebooting Proxmox (not always needed, but you may aswell)

Say “yes” to correcting Proxmox VE sources

## Preparing for SDN functionality

[Software-Defined Network](https://pve.proxmox.com/pve-docs/chapter-pvesdn.html)

### Crucial Prerequisite: Enable Kernel IP Forwarding

By default, Hetzner's Debian images disable packet forwarding for security reasons. Even if you configure SNAT in Proxmox, the kernel will drop the packets, preventing VMs from reaching the internet. You must enable IP forwarding at the OS level first.

Execute the following via SSH:

```bash
cat << EOF > /etc/sysctl.d/99-ip-forwarding.conf
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF

# Apply the changes immediately
sysctl -p /etc/sysctl.d/99-ip-forwarding.conf
```

```sh
apt update && apt install -y dnsmasq
systemctl disable --now dnsmasq
```

```sh
vi /etc/network/interfaces
```

```
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback
iface enp0s31f6 inet manual
auto vmbr0
...
..
```

```sh
ifreload -a
```

### Create the SDN Zone (Network Boundary & DHCP)

The Zone defines the network type. For a standalone host, we use a `Simple` zone (an isolated bridge) but empower it with the native IPAM (IP Address Management) and DHCP features.

1. Navigate to **Datacenter > SDN > Zones**.
2. Click **Add > Simple**.
3. **ID:** `localnat` (or your preferred naming convention).
4. **Nodes:** pve
5. **IPAM:** Select `pve` (This enables tracking VM IP assignments natively in the Proxmox GUI).
6. **Advanced Settings:** Check the **automatic DHCP** box. (Proxmox will manage an isolated `dnsmasq` service for this zone to lease IPs to your VMs automatically).
7. Click **Add**.

### Create the VNet (Virtual Switch)

The VNet acts as the logical switch that your VMs will connect to.

1. Navigate to **Datacenter > SDN > VNets**.
2. Click **Create**.
3. **Name:** `vnet0` (e.g., `k8s-net` if isolating for Talos/Kubernetes).
4. **Zone:** Select the `localnat` zone created in the previous step.
5. Click **Create**. _(Leave VLAN Aware unchecked for a flat subnet setup)._

### Define the Subnet and DHCP Range

This step defines your private IP space and enables SNAT (Source NAT) to allow outbound internet access using the host's single public IP.

1. Still under **VNets**, select your newly created `vnet0` on the left side.
2. On the right panel, under **Subnets**, click **Create**.
3. **Subnet:** `10.0.0.0/24` (Define your private CIDR block).
4. **Gateway:** `10.0.0.1`
5. **SNAT:** **Check this box.** (This tells Proxmox to auto-generate the masquerade/NAT rules for outbound internet access).
6. Switch to the **DHCP Ranges** tab.
7. Click **Add** and define the pool for VMs:

    - **Start Address:** `10.0.0.100`
    - **End Address:** `10.0.0.200`

8. Click **Create**.

### Apply the SDN Configuration

SDN configurations remain in a "pending" state until explicitly applied. If you skip this step, the network interfaces will not be generated.

1. Navigate back to the main **Datacenter > SDN** level.
2. Click the **Apply** button at the top of the interface.
3. Proxmox will generate the necessary files under `/etc/network/interfaces.d/sdn` and reload the networking services seamlessly.

_When creating a new VM, simply assign its network device to the `vnet0` bridge. It will automatically pull a `10.0.0.x` IP address, route through `10.0.0.1`, and securely access the internet._

### Testing the connectivity

Now we can create a CT to test the network connectivity.

Datacenter > prox > local > CT Templates > Templates > rockylinux-10 > Download
Dashboard > Create CT

- General
	- Node: prox
	- CT ID: 100
	- Password: <YOUR_VM_PASSWORD>
- Templates
	- Storage: local
	- Template: rocylinux-10
- Network
	- Bridge: vnet1
	- IPv4: DHCP

Datacenter > prox > 100 (CT 100) > Console

```sh
Rocky Linux 10.0 (Red Quartz)
Kernel 6.17.13-4-pve on x86_64

CT100 login: root
Password: 

[root@CT100 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether bc:24:11:a5:a6:82 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.0.0.100/24 brd 10.0.0.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever

[root@CT100 ~]# ping google.com
PING google.com (142.251.14.113) 56(84) bytes of data.
64 bytes from pm-in-f113.1e100.net (142.251.14.113): icmp_seq=1 ttl=114 time=5.25 ms
64 bytes from pm-in-f113.1e100.net (142.251.14.113): icmp_seq=2 ttl=114 time=5.19 ms
```

## API Token for Automation (OpenTofu/Terraform)

Create a dedicated user and API token for infrastructure-as-code instead of using the root account.

1. **Datacenter > Permissions > Users > Add** (e.g., `tofu-user@pve`).
2. **Datacenter > Permissions > API Tokens > Add**.
3. Assign `PVEVMAdmin` or `PVEDatastoreAdmin` roles to the token.

```sh
pveum user add admin@pve --comment "Full Admin API User"
pveum acl modify / --user admin@pve --role Administrator
pveum user token add admin@pve mytoken --privsep 0

# Test
curl -k -H 'Authorization: PVEAPIToken=admin@pve!mytoken=<YOUR_API_TOKEN>' \
     https://<TAILSCALE_IP>:8006/api2/json/nodes
```

## Tailscale

```sh
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --advertise-routes=10.0.0.0/24
```

```bash
Warning: UDP GRO forwarding is suboptimally configured on vmbr0, UDP forwarding throughput capability will increase with a configuration change.                                  
See https://tailscale.com/s/ethtool-config-udp-gro

ethtool -K vmbr0 rx-udp-gro-forwarding on rx-gro-list off

# Persistent
# /etc/network/interfaces
source /etc/network/interfaces.d/*
...
auto vmbr0
iface vmbr0 inet static
  # ... your current IP settings ...
  post-up /usr/sbin/ethtool -K vmbr0 rx-udp-gro-forwarding on rx-gro-list off

```

[Machines - Tailscale](https://login.tailscale.com/admin/machines) > proxmox > edit route settings > 10.0.0.0/24 > Save

>[!note]
>Health check:
>- Some peers are advertising routes but --accept-routes is false
>`sudo tailscale up --accept-routes`

# References

- [Install Proxmox on a Hetzner Dedicated Server with 1 IP using SDN and without KVM using QEMU - CyanLabs](https://cyanlabs.net/tutorials/install-proxmox-on-a-hetzner-dedicated-server-with-1-ip-using-sdn-and-without-kvm-using-qemu/#running-the-community-proxmox-ve-post-install-script)
