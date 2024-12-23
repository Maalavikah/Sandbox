### Step-by-Step Implementation of a Sandboxed Network Using VirtualBox

This document details the creation of a sandboxed network environment using VirtualBox. The setup includes three virtual machines (VMs): a Desktop VM, a Gateway VM, and an Application VM. These VMs are configured in an isolated network to enable secure communication. Static IP addresses were assigned to subnets and gateways as follows:

- **Desktop VM**: `192.168.17.2`
- **Gateway VM (Interface 1)**: `192.168.17.1` (default gateway for the Desktop VM)
- **Gateway VM (Interface 2)**: `192.168.117.1` (default gateway for the Application VM)
- **Application VM**: `192.168.117.2`

The Gateway VM also has a third network interface configured for NAT, enabling internet access.

---

### Network Components

#### Virtual Machines
1. **Desktop VM**: Ubuntu Desktop
2. **Gateway VM**: Ubuntu Server
3. **Application VM**: Bitnami Debian WordPress

---

### Prerequisites

#### Hardware Requirements:
- Host machine with at least 16GB RAM and a multi-core processor
- Minimum 50GB free disk space

#### Software Requirements:
- VirtualBox installed
- ISO images for Ubuntu Desktop and Server
- Bitnami WordPress VM for the Application Server

#### Networking Knowledge:
- Basic understanding of IP addressing, routing, and firewall rules

---

### VirtualBox Configuration

#### Create Virtual Machines
1. Open VirtualBox and click **New** to create each VM.
2. Assign the following specifications:
   - **Desktop VM**: 2 cores, 4GB RAM, 20GB storage
   - **Gateway VM**: 2 cores, 2GB RAM, 10GB storage
   - **Application VM**: 2 cores, 4GB RAM, 20GB storage

#### Configure Network Adapters
1. **All VMs**: Use **Internal Network** for isolation by changing the adapter settings in VirtualBox.
2. **Gateway VM**: Configure three adapters:
   - Adapter 1: Internal Network (for Desktop VM communication)
   - Adapter 2: NAT (for internet access)
   - Adapter 3: Internal Network (for Application VM communication)

Rename adapter names as needed to match network roles.

---

### IP Address Configuration

#### **Desktop VM (Ubuntu Desktop)**:
1. Open network settings.
2. Set the IPv4 method to **Manual** and disable DHCP.
3. Assign the IP address (`192.168.17.2`), subnet mask, and default gateway (`192.168.17.1`).

#### **Gateway VM (Ubuntu Server)**:
1. Edit the network configuration file:
   ```bash
   sudo nano /etc/netplan/00-installer-config.yaml
   ```
2. Configure static IPs:
   ```yaml
   network:
     ethernets:
       enp0s3:  # NAT
         dhcp4: true
       enp0s8:  # Internal Network 1
         addresses:
           - 192.168.17.1/24
         dhcp4: false
       enp0s9:  # Internal Network 2
         addresses:
           - 192.168.117.1/24
         dhcp4: false
     version: 2
   ```
3. Apply changes:
   ```bash
   sudo netplan apply
   ```
4. Enable IP forwarding:
   - Uncomment the `net.ipv4.ip_forward=1` line in `/etc/sysctl.conf`.
   - Apply with:
     ```bash
     sudo sysctl -p
     ```

5. Configure `iptables` for forwarding and NAT:
   ```bash
   # Forwarding between enp0s8 and enp0s9
   sudo iptables -A FORWARD -i enp0s8 -o enp0s9 -j ACCEPT
   sudo iptables -A FORWARD -i enp0s9 -o enp0s8 -j ACCEPT

   # Forwarding between enp0s3 and internal networks
   sudo iptables -A FORWARD -i enp0s3 -o enp0s8 -j ACCEPT
   sudo iptables -A FORWARD -i enp0s8 -o enp0s3 -j ACCEPT
   sudo iptables -A FORWARD -i enp0s3 -o enp0s9 -j ACCEPT
   sudo iptables -A FORWARD -i enp0s9 -o enp0s3 -j ACCEPT

   # Enable NAT
   sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
   ```

6. Make `iptables` rules persistent:
   ```bash
   sudo apt install iptables-persistent
   sudo netfilter-persistent save
   sudo netfilter-persistent reload
   ```

#### **Application VM (Bitnami Debian WordPress)**:
1. Edit the network configuration file:
   ```bash
   sudo nano /etc/network/interfaces
   ```
2. Configure static IP:
   ```text
   auto enp0s3
   iface enp0s3 inet static
     address 192.168.117.2
     netmask 255.255.255.0
     gateway 192.168.117.1
   ```
3. Apply changes with:
   ```bash
   sudo reboot
   ```

---

### Functional Testing
1. **Desktop VM (Ubuntu Desktop)**: Verify connectivity to the Gateway VM and internet access.
2. **Gateway VM (Ubuntu Server)**: Ensure proper routing between the Desktop and Application VMs, and internet access via NAT.
3. **Application VM (Bitnami Debian WordPress)**: Confirm communication with the Gateway VM and successful access to hosted WordPress services.

