# Phase 1 Lesson: Infrastructure Foundation Concepts

## Introduction: Why Infrastructure Matters in Kubernetes

When building a Kubernetes cluster, the foundation layer—your operating system, networking, and security configuration—directly impacts everything that runs above it. Unlike cloud-managed Kubernetes services where the infrastructure is abstracted away, building your own cluster gives you complete control and deep understanding of how all the pieces fit together.

Think of infrastructure as the foundation of a house. You could build on unstable ground, but every floor above would inherit those problems. Similarly, networking issues, security gaps, or misconfigured systems will compound as you add applications, monitoring, and automation layers.

## Understanding Linux Distribution Selection

### Why Ubuntu Server 22.04 LTS?

**Long-Term Support Philosophy**
LTS releases receive 5 years of security updates and bug fixes. In enterprise environments, stability trumps having the latest features. Your Kubernetes cluster needs to run reliably for years, not break due to an unexpected package update.

**Container Ecosystem Maturity**
Ubuntu has deep integration with container technologies. Docker, containerd, and Kubernetes are extensively tested on Ubuntu. The package repositories are well-maintained, and you'll find most Kubernetes documentation assumes Ubuntu or similar Debian-based systems.

**Community and Enterprise Support**
Ubuntu strikes a balance between cutting-edge features and stability. It's used by both hobbyists and Fortune 500 companies, ensuring a robust ecosystem of tools, documentation, and community support.

### Alternative Considerations

- **CentOS/RHEL**: More conservative, enterprise-focused, but longer release cycles
- **Debian**: Ubuntu's upstream, more minimal but requires more manual configuration
- **Alpine Linux**: Extremely lightweight, good for containers but less tooling for bare metal
- **Flatcar Container Linux**: Purpose-built for containers but steeper learning curve

For learning Kubernetes administration, Ubuntu Server provides the best balance of ease-of-use, documentation, and real-world applicability.

## Networking Fundamentals for Kubernetes

### Static IP Strategy

**Why Static IPs?**
Kubernetes nodes form a cluster by communicating over the network. Each node needs a stable identity that doesn't change when DHCP leases renew. If a worker node suddenly gets a new IP address, the control plane loses contact, and your workloads start failing.

**Network Topology Design**
```
Internet → Router → Switch → Kubernetes Nodes
                           ├── k8s-control (192.168.1.10)
                           ├── k8s-worker-1 (192.168.1.11)
                           └── k8s-worker-2 (192.168.1.12)
```

This simple flat network design ensures:
- Low latency between nodes (single switch hop)
- Easy troubleshooting (no complex routing)
- Sufficient IP space for growth (254 possible addresses in /24)
- Clear IP allocation strategy (control plane gets .10, workers get .11+)

**Subnet Planning**
The `/24` subnet (255.255.255.0 netmask) provides 254 usable IP addresses:
- Router typically claims `.1`
- Reserve `.2-.9` for infrastructure (managed switches, access points)
- Kubernetes nodes get `.10-.50`
- DHCP pool can use `.51-.200`
- Reserve `.201-.254` for future static services

### DNS and Service Discovery

External DNS servers (8.8.8.8, 1.1.1.1) provide internet name resolution, but Kubernetes will add its own internal DNS for service discovery. The `/etc/hosts` entries create a backup resolution method and improve bootstrap reliability.

## Security Principles and Implementation

### SSH Key Authentication

**Why Disable Password Authentication?**
Passwords can be brute-forced, intercepted, or shared. SSH keys provide cryptographic proof of identity that's computationally infeasible to forge. Ed25519 keys offer excellent security with small key sizes and fast operations.

**Key Distribution Strategy**
By copying your public key to all nodes, you establish a trust relationship that allows secure automation. This enables:
- Remote administration without passwords
- Secure file transfers (scp, rsync)
- Automation scripts that don't expose credentials
- Easy integration with configuration management tools

### Firewall Configuration (UFW)

**Defense in Depth**
Even on a private network, host-based firewalls provide additional security layers. They protect against:
- Lateral movement if one machine is compromised
- Accidental exposure of development services
- Network-based attacks from other devices

**Kubernetes-Specific Ports**
- `6443/tcp`: Kubernetes API server (the "brain" of your cluster)
- `10250/tcp`: Kubelet API (allows the control plane to manage worker nodes)

Additional ports will be opened in later phases as we add more components (etcd, overlay networking, ingress controllers).

### Automatic Security Updates

**Unattended Upgrades**
Security vulnerabilities are discovered constantly. Automatic security updates ensure your systems stay protected without manual intervention. The configuration only applies security updates, not feature updates that might break functionality.

**Risk vs. Reward**
Some argue against automatic updates in production, preferring scheduled maintenance windows. For learning environments, the security benefit outweighs the small risk of update-related issues.

## Kubernetes Prerequisites Deep Dive

### Swap Disable Requirement

**Why Kubernetes Hates Swap**
Kubernetes makes scheduling decisions based on memory requests and limits. If the kernel starts swapping container memory to disk:
- Performance becomes unpredictable
- Memory-based scheduling breaks down
- Quality of Service guarantees fail
- Pods may be killed unexpectedly

By disabling swap, Kubernetes maintains predictable performance characteristics and can accurately enforce resource constraints.

### Kernel Modules and Network Configuration

**Container Runtime Requirements**
- `overlay`: Enables overlay filesystems for efficient container layer management
- `br_netfilter`: Allows bridge traffic to be processed by iptables (needed for Kubernetes networking)

**Network Bridge Configuration**
```bash
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

These settings ensure that:
- Traffic crossing bridges gets properly filtered by iptables
- IP forwarding works (essential for pod-to-pod communication)
- IPv6 traffic is handled correctly (future-proofing)

### System Resource Limits

**File Descriptor Limits**
Kubernetes components and containers can open many network connections and files simultaneously. The default limits (1024 file descriptors) are too low for container workloads. Increasing to 65536 provides headroom for busy applications.

**Process Limits**
Similarly, containerized applications may spawn many processes or threads. Raising process limits prevents resource exhaustion that could crash applications or system components.

## Hardware Considerations for Mini PC Clusters

### Why Mini PCs for Learning?

**Power Efficiency**
Mini PCs consume 10-15 watts vs. 200-400 watts for traditional servers. For a home lab running 24/7, this makes a significant difference in electricity costs and heat generation.

**Realistic Resource Constraints**
Learning on resource-constrained hardware teaches optimization and real-world thinking. In production, you'll need to understand resource management, and mini PCs force you to be thoughtful about resource allocation.

**Cost and Space**
Three mini PCs cost less than a single enterprise server and fit on a desk. This accessibility makes Kubernetes learning practical for more people.

### Resource Planning

**Memory Allocation Strategy**
- Control plane: Reserve 2-4GB for Kubernetes components
- Worker nodes: Reserve 1-2GB for kubelet, container runtime
- Remaining memory available for application workloads

**Storage Considerations**
- OS and system components: ~20GB
- Container images and logs: 50-100GB
- Application persistent volumes: Remaining space

**Network Bandwidth**
Gigabit ethernet provides sufficient bandwidth for most learning scenarios. Real bottlenecks usually occur at storage I/O or CPU rather than network.

## Connecting to Broader Kubernetes Concepts

### Infrastructure as Code Evolution

This manual setup phase teaches the fundamentals, but everything you're doing could be automated with:
- Cloud-init for initial OS configuration
- Ansible for configuration management
- Terraform for infrastructure provisioning
- GitOps for ongoing management

Understanding the manual steps makes automation more meaningful and helps with troubleshooting when automation fails.

### Production Considerations

**High Availability**
Production clusters typically have:
- Multiple control plane nodes (3 or 5)
- External load balancers
- Separate etcd clusters
- Network redundancy

**Security Hardening**
Additional production security includes:
- Certificate-based authentication for all components
- Network segmentation
- Pod Security Standards
- Regular security scanning
- Audit logging

**Monitoring and Observability**
Production clusters need comprehensive monitoring of:
- Node resource utilization
- Network performance
- Application metrics
- Security events

## Common Pitfalls and Troubleshooting

### Network Issues
- **IP conflicts**: Always verify your chosen static IPs aren't in use
- **DNS resolution**: Test both forward and reverse DNS lookup
- **Firewall overreach**: Start permissive, then tighten rules incrementally

### SSH Problems
- **Key permissions**: SSH keys must have correct file permissions (600 for private, 644 for public)
- **Agent forwarding**: Understand when you need ssh-agent for key management
- **Host key verification**: Plan for managing known_hosts across multiple machines

### Storage Challenges
- **Disk space monitoring**: Kubernetes logs can consume significant space
- **LVM flexibility**: Logical Volume Management allows resizing filesystems later
- **Backup strategy**: Plan for configuration and data backup from day one

## Preparation for Phase 2

By completing Phase 1, you've established:
- A stable, secure operating system foundation
- Reliable networking between all cluster nodes
- Proper security posture for cluster components
- System configuration optimized for container workloads

Phase 2 will build on this foundation by installing:
- Container runtime (containerd)
- Kubernetes components (kubelet, kubeadm, kubectl)
- Cluster networking (CNI plugin)
- Basic cluster verification

Understanding why each Phase 1 step was necessary makes Phase 2's requirements more obvious and troubleshooting more intuitive.