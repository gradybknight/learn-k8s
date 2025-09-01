# SSH Key Rotation Guide

This guide provides step-by-step instructions for rotating the exposed `k8s-cluster-admin` SSH key and updating it across all machines in your Kubernetes cluster.

## Overview

After accidentally committing the SSH private key to Git, it's essential to:
1. Generate a new SSH key pair
2. Update the public key on all cluster nodes
3. Update your local SSH configuration
4. Test connectivity to ensure everything works

## Hardware Architecture Reminder

- **Control Plane Node**: Lenovo ThinkCentre M720Q
- **Worker Nodes**: 2x Beelink Mini S12 Pro  
- **Management Machine**: Your laptop (macOS)

## Step 1: Generate New SSH Key Pair

On your laptop, generate a new SSH key pair:

```bash
# Generate new SSH key pair
ssh-keygen -t ed25519 -f ~/.ssh/k8s-cluster-admin-new -C "k8s-cluster-admin-rotated"

# Set appropriate permissions
chmod 600 ~/.ssh/k8s-cluster-admin-new
chmod 644 ~/.ssh/k8s-cluster-admin-new.pub
```

## Step 2: Backup and Replace Old Key

```bash
# Backup old key (if it still exists locally)
mv ~/.ssh/k8s-cluster-admin ~/.ssh/k8s-cluster-admin.old.backup 2>/dev/null || true
mv ~/.ssh/k8s-cluster-admin.pub ~/.ssh/k8s-cluster-admin.pub.old.backup 2>/dev/null || true

# Move new key to proper location
mv ~/.ssh/k8s-cluster-admin-new ~/.ssh/k8s-cluster-admin
mv ~/.ssh/k8s-cluster-admin-new.pub ~/.ssh/k8s-cluster-admin.pub
```

## Step 3: Update Public Key on All Cluster Nodes

You'll need to update the `authorized_keys` file on each node. First, get your new public key:

```bash
# Display your new public key
cat ~/.ssh/k8s-cluster-admin.pub
```

### Method A: Direct SSH Access (if you still have access)

If you still have SSH access using your old key or password authentication:

```bash
# For each node, replace <NODE_IP> with actual IP addresses
# Control plane node (Lenovo ThinkCentre)
ssh user@<CONTROL_PLANE_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/k8s-cluster-admin.pub | ssh user@<CONTROL_PLANE_IP> "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Worker node 1 (Beelink Mini S12 Pro #1)  
ssh user@<WORKER1_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
cat ~/.ssh/k8s-cluster-admin.pub | ssh user@<WORKER1_IP> "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Worker node 2 (Beelink Mini S12 Pro #2)
ssh user@<WORKER2_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh"  
cat ~/.ssh/k8s-cluster-admin.pub | ssh user@<WORKER2_IP> "cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### Method B: Physical Console Access (if SSH is not available)

If you don't have SSH access, use physical keyboard/monitor on each machine:

1. **Log in directly on each machine**
2. **Create/edit the authorized_keys file:**
   ```bash
   mkdir -p ~/.ssh
   chmod 700 ~/.ssh
   nano ~/.ssh/authorized_keys
   ```
3. **Add your new public key** (copy from your laptop)
4. **Set permissions:**
   ```bash
   chmod 600 ~/.ssh/authorized_keys
   ```

## Step 4: Clean Up Old Keys

After confirming the new key works, remove the old public key entries:

```bash
# On each node, edit authorized_keys to remove old key entries
ssh -i ~/.ssh/k8s-cluster-admin user@<NODE_IP> "nano ~/.ssh/authorized_keys"
```

Remove any lines containing the old public key (the one that was exposed).

## Step 5: Update SSH Configuration

Update your local SSH configuration if you have one:

```bash
# Edit SSH config file
nano ~/.ssh/config
```

Ensure any entries for your cluster nodes reference the correct key:

```
# Example SSH config entries
Host k8s-control
    HostName <CONTROL_PLANE_IP>
    User <USERNAME>
    IdentityFile ~/.ssh/k8s-cluster-admin
    
Host k8s-worker1  
    HostName <WORKER1_IP>
    User <USERNAME>
    IdentityFile ~/.ssh/k8s-cluster-admin
    
Host k8s-worker2
    HostName <WORKER2_IP>
    User <USERNAME>
    IdentityFile ~/.ssh/k8s-cluster-admin
```

## Step 6: Test SSH Connectivity

Test SSH access to each node:

```bash
# Test control plane
ssh -i ~/.ssh/k8s-cluster-admin user@<CONTROL_PLANE_IP>

# Test worker nodes
ssh -i ~/.ssh/k8s-cluster-admin user@<WORKER1_IP>
ssh -i ~/.ssh/k8s-cluster-admin user@<WORKER2_IP>

# Or using SSH config hostnames (if configured)
ssh k8s-control
ssh k8s-worker1
ssh k8s-worker2
```

## Step 7: Update Any Scripts or Automation

Search for and update any scripts, Ansible playbooks, or automation that references the SSH key:

```bash
# Search for references to the old key in your project
grep -r "k8s-cluster-admin" . --exclude-dir=.git
```

## Step 8: Secure the New Key

Ensure your new SSH key is properly secured:

```bash
# Verify permissions
ls -la ~/.ssh/k8s-cluster-admin*

# Should show:
# -rw------- k8s-cluster-admin (private key)  
# -rw-r--r-- k8s-cluster-admin.pub (public key)
```

## Step 9: Update Documentation

Update any documentation that references:
- SSH key names
- Connection instructions  
- Node access procedures

## Troubleshooting

### SSH Connection Refused
- Verify SSH service is running: `sudo systemctl status ssh`
- Check SSH configuration: `sudo nano /etc/ssh/sshd_config`
- Restart SSH service: `sudo systemctl restart ssh`

### Permission Denied
- Verify key permissions (600 for private, 644 for public)
- Check authorized_keys permissions (600)
- Verify ~/.ssh directory permissions (700)

### Key Not Working
- Test with verbose output: `ssh -vvv -i ~/.ssh/k8s-cluster-admin user@<NODE_IP>`
- Verify public key was added correctly to authorized_keys
- Check for multiple entries or formatting issues

## Security Best Practices

1. **Never commit SSH private keys to Git**
2. **Use unique SSH keys per environment/purpose**
3. **Regularly rotate SSH keys (every 90-365 days)**
4. **Use strong passphrases for SSH keys**
5. **Monitor SSH access logs**
6. **Disable password authentication once key-based auth is working**

## Cleanup

After successful rotation and testing:

```bash
# Remove backup files (only after confirming everything works)
rm ~/.ssh/k8s-cluster-admin.old.backup
rm ~/.ssh/k8s-cluster-admin.pub.old.backup
```

## Notes

- Replace `<NODE_IP>` with actual IP addresses of your nodes
- Replace `user` with the actual username on your nodes
- Replace `<USERNAME>` with your actual username
- Consider using SSH config files for easier management
- Document your actual IP addresses and usernames for future reference