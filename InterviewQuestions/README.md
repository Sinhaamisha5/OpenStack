# OpenStack Interview - Potential Cross Questions & Answers

## 1. Architecture & Design Questions

### Q1: "Why did you choose a 3-node controller cluster? Why not 2 or 5?"
**Answer:**
"We chose 3 nodes because it's the minimum for proper quorum-based high availability. With 3 nodes, we can tolerate 1 node failure while maintaining quorum (2 out of 3). If we had 2 nodes, losing one would mean no quorum. With 5 nodes, we'd gain more fault tolerance but at higher cost and complexity—3 nodes was the sweet spot for our availability requirements and budget."

### Q2: "How does HAProxy know which controller is healthy? What's the health check mechanism?"
**Answer:**
"HAProxy performs health checks by periodically sending HTTP requests to specific endpoints on each controller—typically the OpenStack API endpoints. We configured health check intervals (every 2 seconds) and thresholds (3 consecutive failures) to mark a backend as down. Keepalived managed the VIP failover between HAProxy instances themselves using VRRP protocol."

### Q3: "What happens if your MariaDB Galera cluster loses quorum?"
**Answer:**
"If Galera loses quorum (less than 50% nodes available), the cluster enters a non-operational state to prevent split-brain scenarios. We'd need to manually bootstrap the cluster by identifying the node with the most recent data using `SHOW STATUS LIKE 'wsrep%'` and then running `galera_new_cluster` on that node. The other nodes would then rejoin automatically. This is why we monitored Galera cluster health closely."

### Q4: "How did you handle database backups for MariaDB?"
**Answer:**
"We used `mysqldump` with `--single-transaction` flag for consistent backups without locking tables. Backups were automated via cron jobs and stored in Swift object storage with a retention policy. For faster recovery, we also took periodic Galera SST snapshots. Critical data like Keystone credentials were additionally backed up to an external system."

### Q5: "What's the difference between bonding mode 1 (active-backup) and mode 4 (LACP)? Which did you use?"
**Answer:**
"Mode 1 (active-backup) uses only one interface at a time for simplicity and works without switch configuration. Mode 4 (LACP/802.3ad) aggregates bandwidth across multiple links but requires switch support. We primarily used mode 1 for management networks for reliability, and mode 4 on compute nodes for higher throughput on data networks where the switches supported it."

---

## 2. Neutron/Networking Deep Dive

### Q6: "Explain the packet flow when a VM with a private IP sends traffic to the internet via a floating IP."
**Answer:**
1. VM sends packet with source IP 192.168.10.5 (private)
2. Packet reaches br-int (integration bridge) via tap interface
3. OVS forwards to network node's router namespace (qrouter-xxx)
4. Router performs SNAT: changes source IP from 192.168.10.5 → floating IP 203.0.113.25
5. Packet forwarded to br-ex (external bridge)
6. Exits via physical NIC to external network
7. Return traffic follows reverse path with DNAT

### Q7: "What's the difference between VLAN and VXLAN? Why use both?"
**Answer:**
"VLANs operate at Layer 2 with a 4096 limit on VLAN IDs, suitable for provider/external networks. VXLAN encapsulates Layer 2 frames in Layer 3 UDP packets, supports 16 million IDs, and works across Layer 3 boundaries—ideal for multi-tenant isolation. We used VLANs for provider networks (external connectivity) and VXLAN for tenant networks (scalable isolation)."

### Q8: "How does Neutron handle DHCP for instances? Where does the DHCP server run?"
**Answer:**
"Neutron runs dnsmasq instances inside network namespaces (qdhcp-xxx) on network nodes or controller nodes depending on configuration. Each tenant network gets its own namespace with a dnsmasq process. The DHCP agent listens on the tenant network and assigns IPs from the subnet pool defined during network creation."

### Q9: "What are security groups vs. firewall rules in Neutron? When do you use each?"
**Answer:**
"Security groups are stateful, instance-level firewall rules applied at the compute node (via iptables/OVS flows). They're project-specific and follow instances. FWaaS (Firewall-as-a-Service) provides network-level, stateless firewall at the router, protecting entire subnets. We used security groups for instance-level protection and FWaaS for perimeter security at tenant routers."

### Q10: "If a VM cannot reach the internet, what's your troubleshooting process?"
**Answer:**
1. Check VM: `ip addr`, `ip route`, `ping gateway`
2. Verify security groups allow outbound traffic
3. Check floating IP association: `openstack server show`
4. On network node, check router namespace: `ip netns exec qrouter-xxx ping <destination>`
5. Verify NAT rules: `ip netns exec qrouter-xxx iptables -t nat -L`
6. Check L3 agent logs: `/var/log/neutron/neutron-l3-agent.log`
7. Verify external network connectivity from network node

---

## 3. Nova/Compute Deep Dive

### Q11: "How does Nova Scheduler decide which compute node to place an instance?"
**Answer:**
"The scheduler uses a two-phase process:
1. **Filtering**: Eliminates hosts that don't meet requirements (RAM, CPU, disk, availability zones). Filters include ComputeFilter, RamFilter, ImagePropertiesFilter, AvailabilityZoneFilter.
2. **Weighing**: Ranks remaining hosts using weights (RAMWeigher, DiskWeigher). By default, it picks the host with most free RAM.

We customized weights to balance load evenly rather than filling one host completely."

### Q12: "What's the difference between live migration and cold migration?"
**Answer:**
"**Live migration**: VM continues running during migration. Memory pages copied incrementally, then a brief pause (downtime ~1 second) for final sync. Requires shared storage or block migration.
**Cold migration**: VM is shut down, disk copied, then started on destination. Causes downtime but doesn't require shared storage.

We used live migration during maintenance windows and cold migration for permanent relocations."

### Q13: "What happens if a compute node fails? How do instances recover?"
**Answer:**
"If a compute node fails and VMs use local storage, those instances are lost until the node recovers. With shared storage (Ceph), we could evacuate instances to healthy compute nodes using `nova evacuate`. For critical workloads, we implemented instance HA using Masakari, which automatically detects failures and evacuates instances."

### Q14: "Explain CPU pinning and NUMA topology. When would you use it?"
**Answer:**
"CPU pinning dedicates specific physical CPU cores to VM vCPUs, preventing context switching and improving performance. NUMA (Non-Uniform Memory Access) ensures VMs access local memory on the same CPU socket. We used this for high-performance workloads like databases and real-time applications. It's configured via flavor extra specs: `hw:cpu_policy=dedicated` and `hw:numa_nodes=1`."

### Q15: "What's the difference between ephemeral disk and persistent volume?"
**Answer:**
"**Ephemeral disk**: Temporary storage on compute node's local disk. Data lost when instance deleted. Fast but not portable.
**Persistent volume (Cinder)**: Block storage that persists independently. Can be detached/reattached. Survives instance deletion.

We used ephemeral for stateless apps and Cinder volumes for databases and stateful services."

---

## 4. Storage (Cinder/Swift/Ceph) Deep Dive

### Q16: "Why did you choose Ceph over traditional SAN storage?"
**Answer:**
"Ceph provided software-defined, scale-out storage that was cost-effective and didn't lock us into vendor hardware. It offered unified block (RBD) and object storage, automatic replication, self-healing, and horizontal scalability. We could add storage capacity by simply adding nodes. It also eliminated single points of failure compared to traditional SAN controllers."

### Q17: "How does Ceph ensure data availability if a node fails?"
**Answer:**
"Ceph uses replication (default 3 copies) across different failure domains (different hosts/racks). Data is distributed using CRUSH algorithm across OSDs (Object Storage Daemons). If an OSD or node fails, Ceph detects it via heartbeat, marks it down, and automatically re-replicates data from remaining copies to maintain replication factor. We monitored this via `ceph status` and `ceph health detail`."

### Q18: "What's the difference between Ceph's replicated pools and erasure-coded pools?"
**Answer:**
"**Replicated**: Stores 3 full copies (default). 200% storage overhead but faster recovery and better performance. Used for Cinder volumes.
**Erasure-coded**: Stores data + parity chunks (like RAID 5/6). Lower overhead (e.g., k=4, m=2 gives ~50% overhead) but slower recovery. Used for Swift cold storage and backups.

We used replicated for hot data and erasure-coded for archives."

### Q19: "How does iSCSI work between Cinder and compute nodes?"
**Answer:**
"When a volume is attached:
1. Cinder creates LVM logical volume on storage node
2. tgtd (iSCSI target daemon) exposes it as an iSCSI target
3. Compute node uses `iscsiadm` to discover and login to target
4. Block device appears as `/dev/sdX` on compute node
5. Nova attaches it to VM (appears as `/dev/vdb` in VM)

We used multipath for redundancy across multiple iSCSI paths."

### Q20: "Explain Swift's ring mechanism. How does it work?"
**Answer:**
"Swift rings are hash-based mapping tables that determine where data should be stored:
- **Account ring**: Maps account data to servers
- **Container ring**: Maps container metadata to servers  
- **Object ring**: Maps objects to servers

Each ring uses consistent hashing with partition power. When data arrives, Swift hashes the object name, determines partition, then looks up which 3+ servers hold that partition. This enables horizontal scaling—you add capacity by adding to rings and rebalancing."

### Q21: "What happens when you rebalance a Swift ring?"
**Answer:**
"Rebalancing redistributes partitions across storage nodes to accommodate new/removed devices. Process:
1. `swift-ring-builder object.builder add` (add new device)
2. `swift-ring-builder object.builder rebalance` (calculate new distribution)
3. Copy ring files to all nodes
4. Replication processes migrate data to new locations

We did this during low-traffic windows and monitored replication lag. It's designed to minimize data movement—typically only 1/replica data moves."

---

## 5. Keystone/Identity Deep Dive

### Q22: "How does token validation work? Does every service call Keystone?"
**Answer:**
"To avoid overloading Keystone, services cache tokens locally using middleware. When a request arrives:
1. Service checks local cache first (TTL-based)
2. If miss or expired, validates with Keystone
3. Caches the result

We also used Fernet tokens (smaller, no persistence needed) instead of UUID tokens to reduce database load. Token expiry was set to 1 hour to balance security and caching efficiency."

### Q23: "What's the difference between a domain, project, and user in Keystone?"
**Answer:**
"- **Domain**: Top-level container for projects, users, and groups. Used for organizational separation (e.g., separate domain per department).
- **Project (Tenant)**: Resource container for VMs, networks, volumes. Provides billing/quota boundary.
- **User**: Individual account that can be assigned roles in one or more projects.

A user in domain A cannot see resources in domain B unless explicitly granted."

### Q24: "How did you integrate Keystone with LDAP/Active Directory?"
**Answer:**
"We configured Keystone to use LDAP as a read-only identity backend:
```
[identity]
driver = ldap

[ldap]
url = ldap://ldap.vodafone.com
user = cn=admin,dc=vodafone,dc=com
password = <secret>
```
This allowed employees to use corporate credentials. Keystone domains mapped to LDAP OUs. Local database still stored project and role assignments. This gave us centralized user management while maintaining OpenStack-specific authorization."

---

## 6. Deployment & Automation

### Q25: "Why did you use OpenStack-Ansible instead of DevStack or manual deployment?"
**Answer:**
"DevStack is for development/testing only—not production-ready. Manual deployment is error-prone and hard to maintain. OpenStack-Ansible provided:
- Idempotent, repeatable deployments
- Production-grade HA configurations  
- Container-based service isolation (LXC)
- Automated upgrades
- Tested reference architecture

It allowed us to deploy consistently across environments and scale operations."

### Q26: "How do OpenStack-Ansible playbooks deploy services into LXC containers?"
**Answer:**
"OpenStack-Ansible creates LXC containers on physical hosts for service isolation. Each container runs one or more OpenStack services (e.g., nova-api, neutron-server). Playbooks:
1. Create containers with proper network bridges
2. Install dependencies inside containers
3. Configure services with templated configs
4. Start services and register with load balancers

This provides process isolation, easier updates, and resource control via cgroups."

### Q27: "How did you handle OpenStack upgrades with zero downtime?"
**Answer:**
"We followed a rolling upgrade strategy:
1. **Read release notes**: Check for breaking changes
2. **Upgrade order**: Database → Keystone → other services
3. **One controller at a time**: Drain load balancer → upgrade → test → re-add
4. **Compute nodes**: Use `nova-manage` to mark hosts disabled, live-migrate VMs, upgrade, re-enable
5. **Database migrations**: Run schema migrations during low-traffic
6. **Rollback plan**: Kept previous version containers for quick rollback

API versioning ensured old and new services coexisted briefly."

---

## 7. Monitoring & Troubleshooting

### Q28: "What metrics did you monitor for OpenStack health?"
**Answer:**
"**Infrastructure level:**
- CPU, memory, disk I/O on physical hosts
- Network throughput and packet loss
- Storage latency and IOPS

**OpenStack services:**
- API response times and error rates
- RabbitMQ queue depths and consumer counts
- Database connection pool usage
- Nova scheduler delays
- Neutron agent heartbeats
- Cinder/Swift storage capacity

**Tenant level:**
- Instance creation success/failure rate
- VM CPU steal time
- Volume attachment times

We used Prometheus for metrics, Grafana for dashboards, and Nagios for alerting."

### Q29: "Describe a production outage you handled and how you resolved it."
**Best practice answer structure:**
"**Situation**: RabbitMQ on one controller crashed, causing API timeouts
**Detection**: Monitoring alerts showed RabbitMQ cluster degraded
**Investigation**: Checked `/var/log/rabbitmq/`, found OOM killer terminated process due to memory leak
**Immediate fix**: Restarted RabbitMQ service, cluster auto-rejoined
**Root cause**: Unacknowledged messages accumulated in queues
**Permanent fix**: Implemented message TTL policies and increased memory limits
**Prevention**: Added RabbitMQ memory usage monitoring with alerts"

### Q30: "How do you troubleshoot 'VM stuck in spawning state'?"
**Answer:**
"Troubleshooting steps:
1. Check nova-compute logs on target host: `/var/log/nova/nova-compute.log`
2. Verify compute service is up: `openstack compute service list`
3. Check RabbitMQ connectivity from compute node
4. Verify image accessibility from compute node
5. Check available disk space on compute node
6. Review instance fault: `openstack server show --diagnostics <instance>`
7. Check hypervisor logs: `journalctl -u libvirtd`

Common causes: Image download timeout, insufficient disk space, network issues, or hypervisor problems."

---

## 8. Security & Compliance

### Q31: "How did you secure the OpenStack API endpoints?"
**Answer:**
"Multiple layers:
- **TLS/SSL**: All APIs behind SSL certificates (Let's Encrypt or internal CA)
- **API rate limiting**: Via HAProxy or neutron-api built-in
- **Network segmentation**: Management APIs on separate VLAN, not directly internet-facing
- **Authentication**: Keystone with strong password policies, token expiry
- **RBAC**: Role-based access control, least-privilege principle
- **Audit logging**: Enabled Keystone audit middleware to log all API calls
- **Security groups**: Restrict API access to known IP ranges"

### Q32: "How did you implement project isolation and prevent tenant data leakage?"
**Answer:**
"- **Network isolation**: Each project has separate Neutron networks (VXLAN), no cross-project routing by default
- **Storage isolation**: Cinder volumes and Glance images scoped to projects
- **Compute isolation**: Security groups filter traffic at vNIC level
- **Hypervisor**: VMs isolated by libvirt/KVM, no shared memory
- **API-level**: Keystone policy files enforce project boundaries
- **Audit**: Regular reviews of shared resources (public images, networks)

We also ran periodic security scans and penetration tests."

---

## 9. Performance & Optimization

### Q33: "How did you optimize Nova API performance?"
**Answer:**
"- **Scaling**: Ran multiple nova-api workers (CPU count × 2)
- **Database**: Indexed frequently queried columns, used connection pooling
- **Caching**: Enabled cache for Keystone token validation (Memcached)
- **Load balancing**: Distributed requests across controllers
- **Database archiving**: Archived old instance records from nova database
- **Tuning**: Adjusted RabbitMQ prefetch counts and heartbeat timeouts

We also monitored slow queries and optimized them."

### Q34: "What caused high CPU steal time in VMs and how did you fix it?"
**Answer:**
"High steal time indicates host CPU oversubscription—hypervisor can't give VM its allocated CPU time.

**Causes:**
- Too many VMs on host (poor scheduling)
- Noisy neighbors (VMs with CPU spikes)
- Host CPU overcommit ratio too aggressive

**Fixes:**
- Adjusted `cpu_allocation_ratio` from default 16:1 to 4:1
- Implemented CPU pinning for critical workloads
- Created separate host aggregates for different workload types
- Monitored host CPU usage and rebalanced via live migration"

---

## 10. Real-World Scenarios

### Q35: "A developer says 'OpenStack is slow compared to AWS.' How do you respond?"
**Answer:**
"I'd first understand what they mean by 'slow':
- **VM boot time**: Could be image size, backend storage speed, network latency. We'd optimize by using smaller images, Ceph SSD pools, or pre-cached images.
- **API response**: Check controller load, database performance, network latency. Scale controllers or optimize queries.
- **Disk I/O**: Use SSD-backed volumes, tune Ceph for performance, enable caching.

Then I'd explain that OpenStack *can* match AWS performance with proper tuning, but requires more operational expertise since we manage the infrastructure ourselves. The tradeoff is control, compliance, and cost savings."

### Q36: "If you had to start from scratch, what would you do differently?"
**Answer:**
"Based on lessons learned:
1. **Use Ceph from day one** instead of starting with LVM—easier scaling
2. **Implement proper monitoring earlier**—saved us from several incidents
3. **Automate day-2 operations**—not just deployment but upgrades, backups, cleanups
4. **Better capacity planning**—we underestimated storage growth
5. **Document disaster recovery procedures**—and test them regularly
6. **Implement proper RBAC from start**—easier than retrofitting
7. **Use infrastructure-as-code** for everything—not just OpenStack but physical network configs too"

---

## 11. Behavioral/Soft Skills

### Q37: "Tell me about a time you disagreed with a team decision."
**Use STAR method:**
"**Situation**: Team wanted to use local storage for all VMs to save costs
**Task**: I needed to advocate for shared storage despite higher cost
**Action**: Prepared cost-benefit analysis showing downtime costs from non-live-migratable VMs, presented alternative of hybrid approach
**Result**: Adopted hybrid model—ephemeral for dev/test, Ceph for production. Reduced costs by 40% while maintaining production SLA"

### Q38: "How do you stay updated with OpenStack developments?"
**Answer:**
"- Read OpenStack release notes for each version
- Follow OpenStack blog and mailing lists
- Attend OpenStack Summit (virtual/in-person)
- Participate in local OpenStack user groups
- Follow key contributors on GitHub
- Test new features in lab environment before production
- Contribute to documentation/wiki when I solve unique problems"

---

## Quick Reference: Key Numbers to Remember

| Metric | Value | Context |
|--------|-------|---------|
| Controller cluster | 3 nodes | Minimum for HA with quorum |
| Token expiry | 1 hour | Default Keystone token TTL |
| VLAN limit | 4096 | Why VXLAN needed |
| VXLAN limit | 16 million | Tenant network scalability |
| Ceph replication | 3 copies | Default redundancy |
| Swift replication | 3 zones | Default durability |
| Nova CPU overcommit | 4:1 | Conservative production ratio |
| Nova RAM overcommit | 1.5:1 | Conservative production ratio |
| Health check interval | 2-5 sec | HAProxy typical setting |
| RabbitMQ heartbeat | 60 sec | Default timeout |

---

## Pro Tips for Interview

✅ **Do:**
- Admit if you don't know something, then explain how you'd find out
- Ask clarifying questions before answering
- Draw diagrams if offered whiteboard
- Relate technical details to business value

❌ **Don't:**
- Make up answers
- Blame others for failures
- Give one-word answers
- Get defensive about architectural choices