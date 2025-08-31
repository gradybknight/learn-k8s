# Phase 1: Infrastructure Foundation

## Learning Objectives

- Install Ubuntu Server on all three machines
- Configure static networking for Kubernetes
- Set up secure SSH access
- Implement basic security hardening
- Prepare machines for Kubernetes installation

## Prerequisites

- 3 mini PCs (1 Lenovo, 2 Beelink)
- Network switch and router
- USB drives for installation media
- Basic understanding of Linux command line

## Step 1: Choose and Download Ubuntu Server

**Why Ubuntu Server 24.04 LTS?**

- Long-term support (5 years)
- Excellent Kubernetes documentation and community support
- Stable package repositories
- Well-tested container runtime compatibility
- Latest kernel and system improvements

1. Download Ubuntu Server 24.04.3 LTS ISO:

   ```bash
   # On Linux/WSL
   wget https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso

   # On macOS (if wget not available)
   curl -O https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso
   ```

2. Verify the download (optional but recommended):

   ```bash
   # On Linux/WSL
   wget https://releases.ubuntu.com/24.04/SHA256SUMS
   sha256sum -c SHA256SUMS 2>&1 | grep ubuntu-24.04.3-live-server-amd64.iso

   # On macOS
   curl -O https://releases.ubuntu.com/24.04/SHA256SUMS
   shasum -a 256 -c SHA256SUMS 2>&1 | grep ubuntu-24.04.3-live-server-amd64.iso
   ```

## Step 2: Create Installation Media

1. **On macOS:**

   ```bash
   # Find your USB device
   diskutil list

   # Unmount the USB (replace diskX with your USB)
   diskutil unmountDisk /dev/diskX

   # Write the ISO to USB
   sudo dd if=ubuntu-24.04.3-live-server-amd64.iso of=/dev/rdiskX bs=1m
   ```

2. **On Linux:**

   ```bash
   # Find your USB device
   lsblk

   # Write the ISO to USB (replace sdX with your USB)
   sudo dd if=ubuntu-24.04.3-live-server-amd64.iso of=/dev/sdX bs=4M status=progress
   ```

## Step 3: Plan Your Network Configuration

Before installation, plan your network topology:

```
Internet → Router → Switch → [k8s-control, k8s-worker-1, k8s-worker-2]
```

**Recommended IP Scheme:**

- Router: 192.168.1.1 (typically default)
- k8s-control (Lenovo): 192.168.1.10
- k8s-worker-1 (Beelink #1): 192.168.1.11
- k8s-worker-2 (Beelink #2): 192.168.1.12
- DNS: 8.8.8.8, 1.1.1.1

**Why static IPs for Kubernetes?**

- Kubernetes nodes need stable identities
- Simplifies cluster configuration
- Prevents connection issues after DHCP lease renewals

## Step 4: Configure Auto Power-On (Optional but Recommended)

Before installing Ubuntu Server, configure each machine to automatically power on after power loss. This prevents needing to physically press power buttons after power outages.

### Lenovo ThinkCentre M720Q

1. **Enter BIOS**: Boot the machine and press **F1** (or **F12**) repeatedly during startup
2. **Navigate to Power settings**: Use arrow keys to find the "Power" section/tab
3. **Find AC Recovery setting**: Look for "After Power Loss" or "AC Power Recovery"
4. **Set to Auto-On**: Change the setting to "Power On" (not "Power Off" or "Last State")
5. **Save and Exit**: Press **F10** to save changes and exit

### Beelink Mini S12 Pro

**Note**: The S12 Pro may not support this feature in BIOS.

1. **Enter BIOS**: Boot the machine and press **Delete** repeatedly during startup
2. **Check Boot section**: Navigate to "Boot" menu and look for "Auto Power On" or "Power On Type"
3. **Check Advanced/Power sections**: If not in Boot, try "Advanced" or "Power Management" sections
4. **Enable if found**: Set to "Auto Power On" or "Enabled"
5. **Save**: Press **F4** or **F10** to save changes

**Alternative**: If BIOS doesn't support auto power-on, consider using smart power strips with remote control.

## Step 5: Install Ubuntu Server (Repeat for Each Machine)

### Boot from USB and Basic Setup

1. Insert USB and boot from it (may need to access BIOS/UEFI settings)
2. Select "Try or Install Ubuntu Server"
3. Follow the installation wizard:

**Language & Keyboard:** English (US)

**Network Configuration:**

- Select your ethernet interface
- Choose "Manual" configuration
- Configure static IP:
  ```
  Subnet: 192.168.1.0/24
  Address: 192.168.1.10 (adjust for each machine)
  Gateway: 192.168.1.1
  Name servers: 8.8.8.8,1.1.1.1
  Search domains: (leave blank)
  ```

**Storage Configuration:**

- Use entire disk
- Set up LVM (recommended for flexibility)
- No encryption (for learning purposes)

**Profile Setup:**

```
Your name: Kubernetes Admin
Your server's name: k8s-control (or k8s-worker-1, k8s-worker-2)
Username: k8sadmin
Password: [choose a strong password]
```

**SSH Setup:**

- ✅ Install OpenSSH server
- ✅ Import SSH identity: No (we'll set this up manually)

**Snaps:** Skip for now

### Post-Installation Configuration

After reboot, log in and update the system:

```bash
# Update package lists and upgrade
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget vim htop tree net-tools

# Verify network configuration
ip addr show
ping -c 3 8.8.8.8
```

### Configure Static IP (If Missed During Installation)

If you used DHCP during installation, you can configure static IP addresses afterward:

1. **Identify your network interface:**

   ```bash
   ip addr show
   # Look for your ethernet interface (usually enp1s0, ens3, or similar)
   ```

2. **Edit the Netplan configuration:**

   ```bash
   # Find the netplan file
   sudo ls /etc/netplan/

   # Edit the configuration file (filename may vary)
   sudo vim /etc/netplan/00-installer-config.yaml
   ```

3. **Configure static IP (replace `enp1s0` with your interface name):**

   ```yaml
   network:
     version: 2
     ethernets:
       enp1s0: # Replace with your interface name
         dhcp4: false
         addresses:
           - 192.168.1.10/24 # Adjust IP for each machine
         routes:
           - to: default
             via: 192.168.1.1
         nameservers:
           addresses:
             - 8.8.8.8
             - 1.1.1.1
   ```

4. **Apply the configuration:**

   ```bash
   # Test the configuration first
   sudo netplan try

   # If successful, apply permanently
   sudo netplan apply

   # Verify the new configuration
   ip addr show
   ping -c 3 8.8.8.8
   ```

**Important:** Use these IP addresses:

- k8s-control: 192.168.1.10
- k8s-worker-1: 192.168.1.11
- k8s-worker-2: 192.168.1.12

## Step 6: Configure SSH Key-Based Authentication

**On your main computer (one-time setup):**

```bash
# Generate SSH key pair if you don't have one
ssh-keygen -t ed25519 -C "k8s-cluster-admin"

# Copy public key to each server
ssh-copy-id k8sadmin@192.168.1.10  # k8s-control
ssh-copy-id k8sadmin@192.168.1.11  # k8s-worker-1
ssh-copy-id k8sadmin@192.168.1.12  # k8s-worker-2
```

**On each server, disable password authentication:**

```bash
# Edit SSH configuration
sudo vim /etc/ssh/sshd_config

# Find and modify these lines:
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no

# Restart SSH service
sudo systemctl restart ssh
```

**Test SSH access:**

```bash
# From your main computer
ssh k8sadmin@192.168.1.10
```

## Step 7: Basic Security Hardening

**Configure firewall (UFW):**

```bash
# Enable and configure UFW
sudo ufw enable

# Allow SSH
sudo ufw allow ssh

# Allow Kubernetes ports (we'll add more later)
sudo ufw allow 6443/tcp  # Kubernetes API server
sudo ufw allow 10250/tcp # Kubelet API

# Check status
sudo ufw status verbose
```

**Set up automatic security updates:**

```bash
# Install unattended-upgrades
sudo apt install -y unattended-upgrades

# Configure automatic updates
sudo dpkg-reconfigure -plow unattended-upgrades
# Select "Yes" when prompted
```

**Configure system limits:**

```bash
# Edit limits for Kubernetes
sudo vim /etc/security/limits.conf

# Add these lines:
* soft nofile 65536
* hard nofile 65536
* soft nproc 32768
* hard nproc 32768
```

## Step 8: Prepare for Kubernetes

**Disable swap (required for Kubernetes):**

```bash
# Turn off swap
sudo swapoff -a

# Remove swap from fstab to make it permanent
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled
free -h
```

**Configure kernel modules:**

```bash
# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl parameters
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl parameters
sudo sysctl --system
```

**Set hostnames and hosts file:**

```bash
# Set hostname (run on each machine with appropriate name)
sudo hostnamectl set-hostname k8s-control  # or k8s-worker-1, k8s-worker-2

# Edit hosts file on all machines
sudo vim /etc/hosts

# Add these lines:
192.168.1.10    k8s-control
192.168.1.11    k8s-worker-1
192.168.1.12    k8s-worker-2
```

## Step 9: Verification

**Note**: Complete through step 8 on each PC before moving to this step

**Test connectivity between nodes:**

```bash
# From k8s-control
ping -c 3 k8s-worker-1
ping -c 3 k8s-worker-2

# Test SSH between nodes
ssh k8sadmin@k8s-worker-1
```

**Verify system readiness:**

```bash
# Check system resources
free -h
df -h
lscpu

# Verify kernel modules
lsmod | grep br_netfilter
lsmod | grep overlay

# Check sysctl settings
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

## Troubleshooting

**Network Issues:**

- Verify physical connections to switch
- Check router DHCP range doesn't conflict with static IPs
- Test with `ping` and `traceroute`

**SSH Issues:**

- Ensure SSH service is running: `sudo systemctl status sshd`
- Check firewall rules: `sudo ufw status`
- Verify SSH keys: `ssh -v k8sadmin@192.168.1.10`

**Storage Issues:**

- Check disk space: `df -h`
- Verify LVM setup: `sudo lvdisplay`

## Next Steps

With infrastructure foundation complete, you're ready for **Phase 2: Kubernetes Cluster Setup**. You should now have:

- ✅ Three Ubuntu Server machines with static IPs
- ✅ SSH key-based authentication configured
- ✅ Basic security hardening applied
- ✅ System prepared for Kubernetes installation
- ✅ Network connectivity verified between all nodes

**Key Files Created:**

- `/etc/hosts` - Node name resolution
- `/etc/modules-load.d/k8s.conf` - Kernel modules for Kubernetes
- `/etc/sysctl.d/k8s.conf` - Network configuration for Kubernetes
