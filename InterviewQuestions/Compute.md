# List running VMs
virsh list

# Start/stop VMs
virsh start vm-name
virsh shutdown vm-name

# View VM configuration
virsh dumpxml vm-name

# Attach/detach devices
virsh attach-disk vm-name /path/to/disk.img vdb
```

### **Virtualization Concepts**

**CPU Virtualization**
- **vCPU**: Virtual CPUs assigned to VMs
- **CPU Pinning**: Bind vCPUs to specific physical cores (performance)
- **Overcommit**: Allocate more vCPUs than physical cores
  - Ratio: 4:1 or 8:1 typical (4-8 vCPUs per physical core)
  - Works because VMs don't use 100% CPU constantly
- **CPU Steal Time**: Time vCPU waits for physical CPU (indicates overcommit issues)

**Memory Virtualization**
- **Memory Overcommit**: Allocate more RAM than physically available
- **Balloon Driver**: Guest reclaims unused memory
- **KSM (Kernel Same-page Merging)**: Deduplicate identical memory pages
- **Huge Pages**: Large memory pages (2MB/1GB vs 4KB) for better performance
- **NUMA (Non-Uniform Memory Access)**: Memory locality for performance

**Storage Virtualization**
- **Virtual Disks**: Files representing physical disks (qcow2, raw, vmdk)
- **Thin Provisioning**: Allocate storage on-demand (grow as needed)
- **Thick Provisioning**: Pre-allocate full disk space
- **Snapshots**: Point-in-time copy of VM state
- **Live Storage Migration**: Move VM disks while running

### **Container vs VM Virtualization**

| Feature | VMs | Containers |
|---------|-----|------------|
| Isolation | Full OS isolation | Process-level isolation |
| Startup Time | Minutes | Seconds |
| Resource Overhead | High (GB RAM, full OS) | Low (MB RAM, shared kernel) |
| Density | 10s per host | 100s per host |
| Security | Stronger isolation | Weaker (shared kernel) |
| Use Case | Different OSes, strong isolation | Microservices, high density |

**Apple's Approach**: One-VM-per-container model
- Each container runs in lightweight VM
- Combines container efficiency with VM security
- Uses macOS Hypervisor.framework

### **Interview Questions on Compute**

**Q: "How would you troubleshoot poor VM performance?"**
```
1. Check CPU steal time (host overcommit)
2. Verify memory allocation (no swapping)
3. Check disk I/O (IOPS limits, storage backend)
4. Review network throughput
5. Examine hypervisor logs
6. Check for resource contention with other VMs
7. Verify CPU/memory topology (NUMA alignment)
```

**Q: "Explain live migration of VMs"**
*"Live migration moves a running VM between hosts without downtime:
1. **Memory pre-copy**: Copy memory pages to destination while VM runs
2. **Iterative copy**: Copy changed pages (dirty pages)
3. **Stop-and-copy**: Pause VM briefly, copy final changes
4. **Activate**: Resume VM on destination host
5. **Cleanup**: Remove VM from source

Requires shared storage or storage migration, and compatible CPU features between hosts."*

**Q: "What's the difference between paravirtualization and full virtualization?"**
- **Full Virtualization**: Guest OS unmodified, hypervisor emulates hardware (slower)
- **Paravirtualization**: Guest OS modified to be VM-aware (faster, requires OS changes)
- **Hardware-assisted**: CPU features (VT-x/AMD-V) help hypervisor (best of both)

---

## 2. STORAGE VIRTUALIZATION

### **Core Concepts**

**What is Storage Virtualization?**
- Abstraction layer between physical storage and applications
- Pools multiple storage devices into logical units
- Provides flexibility, mobility, and management simplification

**Storage Types in Cloud**

**Block Storage**
- Raw storage volumes (like physical disks)
- Formatted with filesystem by user
- Examples: AWS EBS, OpenStack Cinder, iSCSI, Fibre Channel
- Use cases: Databases, boot volumes, applications needing consistent performance
- Protocols: iSCSI, Fibre Channel, NVMe-oF

**Object Storage**
- Store data as objects with metadata
- Accessed via HTTP/REST API
- Examples: AWS S3, OpenStack Swift, MinIO
- Use cases: Backups, media files, data lakes
- Features: Unlimited scale, high durability, eventual consistency

**File Storage**
- Shared filesystem accessible by multiple clients
- Examples: NFS, CIFS/SMB, AWS EFS
- Use cases: Shared application data, home directories
- Protocols: NFS, SMB, CIFS

### **Key Technologies**

**Ceph** (Software-Defined Storage)
- Unified storage: block (RBD), object (RADOS Gateway), file (CephFS)
- Distributed, self-healing, no single point of failure
- **CRUSH algorithm**: Determines data placement
- **Replication**: Typically 3x copies for durability
- Used widely with OpenStack

**iSCSI (Internet Small Computer Systems Interface)**
- Block storage over IP networks
- Target (storage server) and Initiator (client)
- Cheaper than Fibre Channel, uses standard Ethernet
- Performance sensitive to network latency

**LVM (Logical Volume Manager)**
- Flexible disk management on Linux
- Create logical volumes from physical volumes
- Resize, snapshot, stripe volumes
- Used by Cinder for volume management

**Cinder (OpenStack Block Storage)**
- Provides persistent block storage to VMs
- Supports multiple backends: Ceph, LVM, NetApp, EMC
- Features: volumes, snapshots, backups, clones
- Volume types define QoS and backend

### **Storage Concepts**

**IOPS (Input/Output Operations Per Second)**
- Measure of storage performance
- SSD: 10,000-100,000+ IOPS
- HDD: 100-200 IOPS
- Critical for databases and transactional workloads

**Throughput vs IOPS**
- **IOPS**: Number of operations (random access)
- **Throughput**: MB/s transferred (sequential access)
- Different workloads need different optimization

**Storage Tiering**
- **Hot**: Frequently accessed (SSD/NVMe)
- **Warm**: Occasionally accessed (SAS/SATA SSD)
- **Cold**: Rarely accessed (HDD, tape)
- Automatic data movement based on access patterns

**Data Redundancy**

**RAID Levels**:
- **RAID 0**: Striping (performance, no redundancy)
- **RAID 1**: Mirroring (redundancy, 50% capacity loss)
- **RAID 5**: Striping with parity (1 disk failure tolerance)
- **RAID 6**: Striping with dual parity (2 disk failure tolerance)
- **RAID 10**: Mirrored stripes (best performance + redundancy)

**Replication**:
- **Synchronous**: Data written to both sites simultaneously (zero RPO, high latency)
- **Asynchronous**: Data written locally first, replicated later (some data loss possible, lower latency)

**Snapshots vs Backups**
- **Snapshots**: Point-in-time copy (fast, space-efficient, same storage)
- **Backups**: Full copy to different location (slower, separate storage, better protection)

### **Performance Optimization**

**Thin Provisioning**
- Allocate storage on-demand
- Over-subscribe capacity (allocate more than available)
- Monitor actual usage, expand as needed
- Risk: Run out of space if not monitored

**Caching**
- **Write-through**: Write to cache and storage simultaneously (safe, slower)
- **Write-back**: Write to cache first, storage later (fast, risk of data loss)
- **Read cache**: Cache frequently read data (SSD for hot data)

**QoS (Quality of Service)**
- Limit IOPS/bandwidth per volume
- Prevent noisy neighbor problems
- Guarantee minimum performance for critical workloads

### **Interview Questions on Storage**

**Q: "A database VM is experiencing slow I/O. How do you troubleshoot?"**
```
1. Check IOPS limits on volume (QoS throttling?)
2. Verify storage backend health (Ceph cluster status)
3. Review network latency (iSCSI over slow network?)
4. Check for noisy neighbors (other VMs saturating storage)
5. Examine disk queue depth in guest OS
6. Review storage tier (is DB on HDD instead of SSD?)
7. Check for snapshot overhead
8. Verify filesystem alignment and settings
```

**Q: "How do you handle a storage backend failure?"**
*"Depends on architecture:
- **With replication** (Ceph): Cluster auto-recovers, VMs continue running (may see brief performance degradation)
- **Without replication**: Immediate impact, VMs on that storage fail
- **Response**: 
  1. Identify affected volumes/VMs
  2. Evacuate or restart VMs on healthy storage
  3. Notify users of potential data loss
  4. Restore from backups if needed
  5. Investigate root cause
  6. Implement preventive measures"*

**Q: "Explain the difference between volume snapshots and backups in Cinder"**
- **Snapshots**: Quick point-in-time copy, stored on same backend, fast recovery, vulnerable to backend failure
- **Backups**: Full copy to different backend (Swift/S3), slower, survives backend failure, used for disaster recovery

---

## 3. NETWORK VIRTUALIZATION

### **Core Concepts**

**What is Network Virtualization?**
- Abstraction of physical network into logical networks
- Software-defined networking (SDN)
- Isolate tenant networks on shared infrastructure
- Programmatic network management

### **Key Technologies**

**Open vSwitch (OVS)**
- Software-based multilayer switch
- Supports VLANs, tunneling (VXLAN, GRE), QoS
- OpenFlow protocol for SDN
- Used by OpenStack Neutron, KVM

**Linux Bridge**
- Simple software bridge in Linux kernel
- Connects network interfaces
- Less feature-rich than OVS but lower overhead

**Neutron (OpenStack Networking)**
- Network-as-a-Service for OpenStack
- Provides virtual networks, subnets, routers, firewalls
- Plugin architecture (ML2)
- Supports multiple networking technologies

### **Network Isolation Technologies**

**VLANs (Virtual LANs)**
- Layer 2 network segmentation
- 802.1Q tagging (VLAN ID: 1-4094)
- Limited scalability (~4000 VLANs)
- Good for smaller deployments
- Hardware switch support required

**VXLAN (Virtual Extensible LAN)**
- Layer 2 overlay over Layer 3 network
- 24-bit VNI (16 million networks)
- Encapsulates Ethernet frames in UDP
- Solves VLAN scalability limits
- Used in large multi-tenant clouds

**GRE (Generic Routing Encapsulation)**
- Tunneling protocol
- Encapsulates various protocols
- Simpler than VXLAN but less scalable
- No built-in multicast support

**Comparison**:
| Technology | Scale | Complexity | Hardware Support | Use Case |
|------------|-------|------------|------------------|----------|
| VLAN | 4K networks | Low | Required | Small/Medium |
| VXLAN | 16M networks | Medium | Optional | Large multi-tenant |
| GRE | Unlimited | Low | None | Overlay networks |

### **Network Components**

**Virtual Networks**
- Isolated Layer 2 broadcast domains
- Each tenant has separate networks
- Connected via virtual routers

**Virtual Routers**
- Route traffic between networks
- Provide NAT, routing, firewall
- Can be distributed (DVR) or centralized

**Security Groups**
- Stateful firewall rules
- Applied at VM network interface
- Default deny, explicitly allow traffic
- Rules: protocol, port, source/destination

**Floating IPs**
- Public IP addresses for VMs
- NAT to private IP
- Allows external access
- Can be moved between VMs

**Load Balancers**
- Distribute traffic across VMs
- Health checks, SSL termination
- LBaaS (Load Balancer as a Service) in OpenStack

### **Network Topologies**

**Provider Networks**
- Direct connection to physical network
- Shared across tenants
- Flat or VLAN-based
- Better performance (no overlay overhead)

**Tenant Networks**
- Isolated per tenant
- Overlay networks (VXLAN/GRE)
- Flexible, scalable
- Slight performance overhead

**Network Flow**

**East-West Traffic**: VM to VM within cloud
- Optimized with DVR (Distributed Virtual Router)
- Traffic doesn't leave hypervisor if VMs on same host

**North-South Traffic**: External to/from VMs
- Goes through network node or gateway
- Floating IP translation
- Can bottleneck on centralized router

### **SDN (Software-Defined Networking)**

**Control Plane**: Decides where traffic goes
- Routing decisions, topology
- Centralized controller

**Data Plane**: Actually forwards packets
- Switches, routers forwarding traffic
- Fast path

**Benefits**:
- Programmatic control
- Centralized management
- Dynamic reconfiguration
- Automation-friendly

### **Network Performance Optimization**

**SR-IOV (Single Root I/O Virtualization)**
- Direct hardware access for VMs
- Bypasses hypervisor networking
- Near-native performance
- Limited portability (tied to hardware)

**DPDK (Data Plane Development Kit)**
- User-space packet processing
- Bypasses kernel network stack
- High packet rates (millions pps)
- Used with OVS for performance

**Jumbo Frames**
- MTU > 1500 bytes (typically 9000)
- Reduces CPU overhead
- Better throughput for large transfers
- Requires end-to-end support

### **Interview Questions on Networking**

**Q: "A VM cannot reach the internet. Walk through your troubleshooting steps."**
```
1. Verify VM has IP address (DHCP working?)
2. Check routing table (default gateway set?)
3. Test connectivity to gateway (ping default route)
4. Verify security group rules (allow outbound traffic?)
5. Check floating IP association (if needed for external)
6. Verify NAT on virtual router
7. Check network node connectivity
8. Review Neutron agent status
9. Examine OVS flows
10. Check physical network connectivity
```

**Q: "Explain the difference between security groups and firewall rules in Neutron"**
- **Security Groups**: Stateful, applied at VM port level, default deny, per-instance protection
- **Firewall Rules**: Stateless (in FWaaS v1) or stateful (v2), applied at router level, protects entire network
- **Use together**: Security groups for per-VM protection, firewalls for network-level policies

**Q: "How does DVR (Distributed Virtual Router) improve performance?"**
*"Traditional routing sends all traffic through centralized network node (bottleneck). DVR distributes routing to compute nodes:
- East-West traffic stays on compute node (VM-to-VM on different hosts routed locally)
- North-South traffic (floating IP) goes directly from compute node
- SNAT (outbound without floating IP) still centralized
- Reduces network node bottleneck, improves scalability"*

**Q: "Why use VXLAN over VLANs in a large cloud?"**
- **Scalability**: VLANs limited to 4094, VXLAN supports 16M network IDs
- **Flexibility**: VXLAN works over Layer 3, doesn't require VLAN trunking across switches
- **Isolation**: Better multi-tenant isolation
- **Cloud-native**: Designed for virtualized environments

---

## 4. LARGE-SCALE MULTI-TENANT INFRASTRUCTURE MANAGEMENT

### **What is Multi-Tenancy?**

**Definition**: Multiple isolated customers (tenants) sharing same physical infrastructure

**Isolation Requirements**:
- **Compute**: CPU/memory allocation, process isolation
- **Storage**: Separate volumes, encryption
- **Network**: Isolated networks, security groups
- **Data**: No cross-tenant data access
- **Performance**: Prevent noisy neighbors

### **Key Challenges at Scale**

**Noisy Neighbor Problem**
- One tenant's workload impacts others
- **CPU**: High usage starving other VMs
- **Storage**: I/O saturation slowing everyone
- **Network**: Bandwidth hogging

**Solutions**:
- Resource quotas and limits
- QoS enforcement (IOPS limits, bandwidth caps)
- Overcommit ratios (carefully tuned)
- Monitoring and alerting
- Fair share scheduling

**Capacity Planning**
- Predicting growth accurately
- Balancing utilization vs. headroom
- Handling sudden spikes (product launches, sales)
- Hardware refresh cycles

**Approach**:
- Historical trend analysis
- Headroom targets (80% utilization max)
- Regular capacity reviews
- Automated alerting on thresholds
- Staged hardware procurement

**Resource Scheduling**
- Placing VMs efficiently across hosts
- Balancing load
- Affinity/anti-affinity rules
- Maintenance planning

**Strategies**:
- Spread VMs across failure domains
- Pack similar workloads
- Reserve capacity for HA failover
- Automated rebalancing

### **Multi-Tenant Architecture Patterns**

**Project/Tenant Isolation**
- Logical grouping of resources
- Separate quotas per project
- Role-based access control (RBAC)
- Identity management (Keystone in OpenStack)

**Network Isolation**
- Separate networks per tenant (VXLAN)
- Security groups preventing cross-tenant access
- Private IP ranges (overlapping allowed)
- Dedicated routers per tenant

**Storage Isolation**
- Separate volumes per tenant
- Encryption at rest (per-tenant keys)
- Quota enforcement (storage capacity, IOPS)
- Isolated backups

**Resource Quotas**
```
Per-tenant limits:
- Number of instances: 100
- Number of vCPUs: 200
- RAM: 512 GB
- Storage: 10 TB
- Floating IPs: 50
- Security groups: 100
```

### **Managing at Scale**

**Automation is Critical**
- Manual operations don't scale to 10,000+ VMs
- Infrastructure as Code (Terraform, Heat)
- Automated provisioning and deprovisioning
- Self-service portals for users
- API-driven operations

**Monitoring & Observability**
- Track resource utilization across thousands of hosts
- Identify trends and anomalies
- Capacity dashboards
- Per-tenant usage tracking

**Key Metrics**:
- CPU/memory/disk utilization per host
- Network bandwidth usage
- Storage IOPS and latency
- VM density per host
- Error rates and failures
- Time to provision resources

**Operational Best Practices**

**Cell Architecture**
- Divide infrastructure into cells (isolated failure domains)
- Limits blast radius of failures
- Nova cells in OpenStack (groups of compute nodes)
- Independent control planes per cell

**Availability Zones**
- Logical grouping (typically different data centers)
- Users spread VMs across AZs for HA
- Independent power, networking, cooling

**Failure Domain Isolation**
- Separate racks, network switches, storage
- Anti-affinity for critical services
- No single point of failure

**Upgrade Strategies**
- Rolling upgrades (one cell at a time)
- Blue-green deployments
- Canary testing (small subset first)
- Automated rollback on failures
- Minimize tenant impact

### **Security in Multi-Tenant Environments**

**Isolation Enforcement**
- Hypervisor-level isolation (VMs can't escape)
- Network isolation (separate VLANs/VXLANs)
- API access control (Keystone authentication)
- Audit logging (track all actions)

**Compliance**
- Data residency requirements (EU data stays in EU)
- Encryption (data at rest, in transit)
- Regular security audits
- Compliance reporting (SOC 2, ISO 27001)

**Vulnerability Management**
- Regular patching (hypervisor, control plane)
- Security scanning
- Penetration testing
- Incident response procedures

### **Cost Management**

**Chargeback/Showback**
- Track resource usage per tenant
- Bill based on consumption
- Show costs even if not charging (showback)
- Encourage efficient resource usage

**Resource Optimization**
- Identify idle resources
- Right-sizing recommendations
- Scheduled shutdowns (dev/test environments)
- Reserved capacity for predictable workloads

### **Interview Questions on Multi-Tenant Management**

**Q: "How do you prevent one tenant from impacting another's performance?"**
*"Multi-layered approach:
1. **Resource Quotas**: Hard limits on CPU, memory, storage per tenant
2. **QoS Enforcement**: IOPS limits on storage, bandwidth caps on network
3. **Overcommit Controls**: Conservative CPU overcommit ratios (4:1 max)
4. **Monitoring**: Alert on resource saturation, proactive rebalancing
5. **Isolation**: Dedicated resources for large/critical tenants
6. **Scheduling**: Anti-affinity to spread workloads
7. **Fair Share**: CPU scheduler fair allocation across VMs"*

**Q: "You have 10,000 VMs across 500 hosts. A critical security patch needs to be applied. What's your approach?"**
```
1. **Plan**:
   - Test patch on dev/staging first
   - Identify maintenance windows
   - Coordinate with tenants on critical workloads
   
2. **Automate**:
   - Script the entire process
   - Automated VM migration (live migration where possible)
   - Patch application (Ansible)
   - Verification tests
   
3. **Phased Rollout**:
   - Start with non-production hosts (20%)
   - Monitor for issues (24-48 hours)
   - Continue to production in waves
   - One availability zone at a time
   
4. **Execution**:
   - Evacuate VMs from host (live migration)
   - Apply patch and reboot
   - Verify host health
   - Return VMs or keep on new hosts
   - Move to next host
   
5. **Rollback Plan**:
   - Keep old kernel available
   - Quick rollback procedure tested
   - Decision criteria for abort
   
Timeline: 2-3 weeks for 500 hosts (20-30/day)
```

**Q: "How do you handle capacity planning for unpredictable growth?"**
*"Combination of proactive and reactive strategies:
- **Trend Analysis**: Historical growth rates (weekly/monthly)
- **Headroom Targets**: Maintain 20% free capacity
- **Lead Time Awareness**: Know hardware procurement time (8-12 weeks)
- **Triggers**: Alert at 70% utilization (order more), 80% (expedite)
- **Just-in-Time**: Stage hardware ready but not racked until needed
- **Elasticity**: Use public cloud for burst capacity
- **Regular Reviews**: Monthly capacity meetings with stakeholders
- **Scenario Planning**: Model high-growth scenarios (product launch)
- **Over-build Slightly**: Better to have extra than scramble"*

**Q: "A tenant reports their VMs are running slowly. Others report no issues. How do you diagnose?"**
```
1. **Isolate the Scope**:
   - All their VMs or specific ones?
   - Recent change or ongoing issue?
   - What does "slow" mean? (CPU, network, disk?)

2. **Check Tenant Resources**:
   - Are they hitting quota limits?
   - Review their resource utilization trends
   - Any recent spike in usage?

3. **Examine Specific VMs**:
   - CPU steal time (overcommit issue?)
   - Memory swapping (overallocated?)
   - Disk I/O wait times (storage bottleneck?)
   - Network latency/packet loss

4. **Check Host Health**:
   - Which hosts are their VMs on?
   - Overall host utilization
   - Other VMs on same host (noisy neighbor?)

5. **Network/Storage Path**:
   - Any shared bottlenecks?
   - Storage backend health
   - Network congestion

6. **Recent Changes**:
   - Did we migrate their VMs?
   - Any maintenance activities?
   - Did they change their workload?

Common Causes:
- Noisy neighbor on shared host
- Hit IOPS limit on storage
- CPU overcommit too aggressive
- Network bandwidth saturation
- Application issue (not infrastructure)