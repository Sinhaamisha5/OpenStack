# OpenStack
# OpenStack Interview Preparation - Vodafone Experience

## Table of Contents
1. [Overall Architecture Overview](#1-overall-architecture-overview)
2. [Environment Setup & Deployment](#2-environment-setup--deployment)
3. [Keystone - Identity Service](#3-keystone---identity-service)
4. [Neutron - Networking Service](#4-neutron---networking-service)
5. [Nova - Compute Service](#5-nova---compute-service)
6. [Glance - Image Service](#6-glance---image-service)
7. [Cinder - Block Storage Service](#7-cinder---block-storage-service)
8. [Swift - Object Storage Service](#8-swift---object-storage-service)
9. [Key Interview Talking Points](#9-key-interview-talking-points)

---

## 1. Overall Architecture Overview

### High-Level Summary
**Interview Answer:**
"At Vodafone, we used OpenStack as our private cloud platform to provision and manage compute, storage, and networking resources internally—similar to how AWS or Azure operate, but within our own data centers for compliance and cost efficiency."

### Architecture Components

| Component | Role | Description |
|-----------|------|-------------|
| **Controller Nodes** | Brain | Run all APIs, databases (MariaDB Galera), and messaging (RabbitMQ) |
| **Compute Nodes** | Workers | Run virtual machines using KVM hypervisor |
| **Storage Nodes** | Data | Store VM volumes and objects (Cinder, Swift, Ceph) |
| **Load Balancers** | Traffic Managers | HAProxy + Keepalived for API redundancy and VIP |
| **Networking** | Connectivity | Neutron manages virtual networks, VLANs, and VXLANs |

### Design Principles
- **High Availability**: 3-node controller cluster behind HAProxy
- **Redundancy**: Bonded NICs on all nodes, Ceph storage replication
- **Scalability**: Horizontal scaling of compute and storage nodes
- **Automation**: OpenStack-Ansible for deployment and configuration management
- **Monitoring**: Prometheus, Grafana, and Nagios for health tracking

### Network Architecture
**Three-tier model:**
1. **Management network** - Internal communication between OpenStack services
2. **Provider network (external)** - VLAN-backed for floating IPs and public access
3. **Tenant network (internal)** - VXLAN-based virtual networks for project isolation

---

## 2. Environment Setup & Deployment

### Lab Environment Setup

**What We Built:**
A Vodafone private cloud lab environment with:
- Controller node(s) - Manages OpenStack services
- Compute node(s) - Hosts virtual machines
- Client machine(s) - CLI/API interaction
- Multiple network interfaces per node (management, VLAN, external, VXLAN)

**Tools Used:**
- **VirtualBox** - Virtual machine creation
- **Vagrant** - VM orchestration and automation
- **Git** - Clone OpenStack deployment repositories
- **OpenStack-Ansible** - Service deployment and configuration

**Deployment Process:**
```bash
# Clone the lab repository
git clone https://github.com/OpenStackCookbook/vagrant-openstack
cd vagrant-openstack

# Launch the environment
vagrant up
```

### Deployment Workflow

**Interview Answer:**
"In our Vodafone setup, the network and infrastructure teams first configured the physical servers—including NIC bonding and VLAN setup. Once ready, our cloud engineering team validated the host network configuration, then used OpenStack-Ansible to deploy the OpenStack services. We ensured controller, compute, and storage nodes were reachable on their respective management, storage, and VXLAN networks before running the playbooks. Post-deployment, our SRE team handled ongoing monitoring and scaling."

### Validation Steps
- Verified inter-node connectivity using `ping`, `ip addr`, `ovs-vsctl show`
- Checked service status across all nodes
- Validated network paths across management, VXLAN, and storage networks
- Sourced `openrc` file for CLI authentication

---

## 3. Keystone - Identity Service

### Overview
Keystone is the identity and authentication service—the entry point for all OpenStack operations.

### Key Concepts
- **Users** - Individual accounts (developers, admins)
- **Projects (Tenants)** - Logical grouping of resources with isolation
- **Roles** - Permissions assigned to users within projects
- **Service Catalog** - Directory of all OpenStack service endpoints
- **Tokens** - Temporary authentication credentials

### How It Works
1. User authenticates with credentials → Keystone issues a token
2. Token used for all subsequent API calls
3. Service catalog returned—tells client where Nova, Neutron, etc. are located
4. Token expires after configured timeout (default: 1 hour)

### Real-World Usage

**Interview Answer:**
"Developers authenticated through Keystone to access their projects. Once authenticated, they received a token and a service catalog, which enabled them to interact with any OpenStack service without needing to know individual endpoint addresses. This made the system API-driven and centralized—Keystone was the single source of truth for identity and authorization."

---

## 4. Neutron - Networking Service

### Architecture Overview

**Interview Answer:**
"In our Vodafone OpenStack environment, networking was handled using OpenStack Neutron, which provided both provider and tenant networking capabilities. We had dedicated network nodes managing routing, DHCP, and NAT, while controller nodes ran the neutron-server and API components. Compute nodes hosted VMs and used Open vSwitch (OVS) to connect instance interfaces to virtual networks."

### Network Components
- **neutron-server** - API endpoint on controllers
- **L3 agent** - Handles routing and floating IP NAT
- **DHCP agent** - Provides IP addressing to instances
- **Metadata agent** - Delivers instance metadata
- **Open vSwitch** - Virtual switching (br-int, br-ex, br-vlan bridges)

### Network Flow Example

**Scenario: Developer Creates a VM with External Access**

1. VM created in tenant network (e.g., 192.168.10.0/24) → assigned private IP
2. Floating IP allocated from provider network (e.g., 203.0.113.25)
3. Neutron L3 agent maps floating IP to private IP via NAT on router
4. Developer can SSH into instance or host applications externally
5. Tenant networks remain isolated internally

### Validation & Management

**Post-Deployment Tasks:**
```bash
# Check Neutron agents
neutron agent-list

# Inspect OVS bridges
ovs-vsctl show

# Troubleshoot namespace-based routing
ip netns
```

### High Availability
- HA routers using `keepalived` - no single point of failure
- DHCP and L3 agents configured in HA mode across multiple network nodes

### Issues & Resolutions

**Problem:** Floating IP connectivity failures
**Resolution:** Validated security group rules, checked OVS bridge mappings, verified L3 agent logs

---

## 5. Nova - Compute Service

### Overview
Nova manages the lifecycle of virtual machines (instances) in OpenStack.

### Key Responsibilities

**Interview Summary:**
"At Vodafone, I worked on managing Nova—the compute service of OpenStack. I created and managed flavors defining vCPUs, RAM, and disk resources. To maintain performance, we imposed CPU and IOPS limits using flavor metadata. For instance lifecycle management, we used the CLI to boot, snapshot, shelve, and live-migrate instances across compute hosts. We integrated Nova with Neutron (for networking) and Glance (for images)—ensuring seamless provisioning. During maintenance, I handled live migrations between KVM hosts to ensure zero downtime."

### Core Operations
- **Flavors** - Resource templates (vCPU, RAM, disk)
- **Instance lifecycle** - Boot, stop, restart, shelve, delete
- **Snapshots** - Create point-in-time backups
- **Live migration** - Move running VMs between hosts without downtime
- **Scheduling** - Nova Scheduler places VMs on appropriate compute nodes

### Integration Points
- **Glance** - Retrieves VM images
- **Neutron** - Connects instances to networks
- **Cinder** - Attaches persistent volumes
- **Keystone** - Authentication and authorization

---

## 6. Glance - Image Service

### Overview
Glance manages VM images—registration, discovery, retrieval, and storage.

### Key Features
- Supports multiple formats: qcow2, raw, vmdk, iso
- Storage backends: local filesystem, Ceph RBD, Swift, HTTP
- Integration with Nova (boot instances) and Cinder (create volumes)

### Architecture
- **glance-api** - Handles image requests
- **glance-registry** - Manages image metadata in database

### Practical Usage

**Interview Answer:**
"We configured Glance to store images in a Ceph RBD backend for scalability and redundancy. Our image repository was shared across all compute nodes, and we used qcow2 format for compatibility with KVM hypervisors. We enforced metadata tagging for OS type and architecture to enable Nova's ImagePropertiesFilter during instance scheduling. For golden images, we created snapshots of production-ready instances and shared them securely across projects using the `--shared` visibility model."

### Common Commands
```bash
# Upload image
openstack image create COOKBOOK_CIRROS_IMAGE \
  --disk-format qcow2 \
  --file /tmp/cirros.img

# List images
openstack image list

# Share image with another project
openstack image set --shared <image-id>
```

---

## 7. Cinder - Block Storage Service

### Environment Context

**Interview Answer:**
"In our Vodafone OpenStack environment, we used Cinder as the block storage service. It was deployed using OpenStack-Ansible, and our backend was LVM over iSCSI, which allowed persistent volumes to be attached to instances."

### Architecture & Workflow
1. User requests volume via Horizon or CLI
2. Cinder Scheduler decides which backend/node to use
3. Cinder Volume service creates LVM logical volume from `cinder-volumes` VG
4. Volume exposed to compute node over iSCSI
5. Nova attaches it to VM as block device (e.g., `/dev/vdb`)
6. Volume persists even if instance is deleted

### My Responsibilities
- Validated storage nodes had proper volume groups configured
- Initialized storage using `pvcreate` and `vgcreate`
- Verified Cinder services: `cinder service-list`
- Created and attached test volumes
- Monitored storage capacity and performance

### Common Commands
```bash
# Create volume
openstack volume create --size 10 test_volume

# Attach to instance
openstack server add volume test_instance test_volume

# Inside instance - verify and format
lsblk
mkfs.ext4 /dev/vdb
mount /dev/vdb /mnt
```

### Issues & Resolutions

| Problem | Cause | Resolution |
|---------|-------|------------|
| Volume stuck in "Attaching" | API call interrupted | `cinder reset-state <volume-id> --state available` |
| iSCSI mount failure | Network misconfiguration | Verified connectivity with `iscsiadm -m session`, fixed VLAN routing |
| VG space exhausted | High utilization | Extended VG: `pvcreate /dev/sdc && vgextend cinder-volumes /dev/sdc` |
| Snapshot failures | Volume not detached | Implemented pre-check to ensure detachment before snapshot |

### Improvements Implemented
- Added Prometheus monitoring for Cinder services and storage utilization
- Created automation scripts to clean up unused volumes/snapshots
- Defined volume types with QoS policies (HighIOPS, Standard)

---

## 8. Swift - Object Storage Service

### Environment Context

**Interview Answer:**
"In our Vodafone environment, I worked on managing OpenStack Swift, which was our internal object-storage layer—similar to Amazon S3. It had proxy servers on the controller nodes that handled requests and authentication through Keystone, and multiple storage nodes that actually stored the data."

### Architecture Components

| Component | Function |
|-----------|----------|
| **Proxy Servers** | Entry points for requests (upload/download/delete), run on controllers |
| **Storage Nodes** | Physical/virtual servers storing data—horizontally scalable |
| **Account Service** | Maintains user account metadata |
| **Container Service** | Manages containers (like S3 buckets) |
| **Object Service** | Stores and retrieves actual data |
| **Ring Files** | Maps logical data to physical storage locations |
| **Load Balancer** | HAProxy for high availability |

### How It Works
1. User authenticates via Keystone → receives token
2. Request sent to Swift Proxy → token validated
3. Proxy determines object placement using ring files
4. Object replicated across multiple storage nodes (3x replication)
5. Metadata stored separately for faster lookups

### Use Cases at Vodafone
- Application log storage
- VM image snapshots and backups
- Archived reports and media content
- CI/CD artifact storage

### My Responsibilities
- Created and managed containers (similar to S3 buckets)
- Uploaded/downloaded objects via OpenStack CLI and Swift CLI
- Managed large file uploads (>5GB) using Dynamic Large Objects (DLO)
- Configured Swift ACLs for access control
- Monitored replication status and ring rebalance tasks
- Investigated issues like replication lag or disk-full events

### Common Commands
```bash
# Create container
openstack container create backup-container

# Upload object
openstack object create backup-container backup.log

# Upload large file with segmentation
swift upload -S 1073741824 backup-container large_file.tar.gz

# List objects
openstack object list backup-container

# Set ACL (share with specific project)
swift post -r "admin:*" backup-container
```

### Issues & Resolutions

| Problem | Resolution |
|---------|------------|
| **Replication lag** | Checked `swift-replicator` logs, manually rebalanced rings |
| **Disk full on node** | Identified via monitoring, temporarily removed node from ring |
| **ACL misconfiguration** | Audited ACLs, used restricted project-based sharing vs public |

### Key Takeaway
"Swift gave Vodafone a scalable, redundant, S3-like storage system for logs, backups, and VM images. I helped keep it healthy and consistent through monitoring, troubleshooting, and capacity management."

---

## 9. Key Interview Talking Points

### Elevator Pitch (30 seconds)
"I worked on Vodafone's private OpenStack cloud, where I managed compute, storage, and networking services. The environment used a 3-node controller cluster with HAProxy for HA, multiple compute nodes running KVM, and Ceph-backed storage. I deployed services using OpenStack-Ansible, handled instance lifecycle operations, troubleshot networking and storage issues, and implemented monitoring with Prometheus and Grafana."

### Technical Depth (2 minutes)
"Our OpenStack setup at Vodafone was built for multi-tenancy, scalability, and high availability. We had:

- **Controllers**: 3-node cluster running all management APIs, MariaDB Galera for database replication, and RabbitMQ for messaging—fronted by HAProxy and Keepalived
- **Compute**: Multiple KVM-based nodes with bonded NICs for redundancy
- **Storage**: Ceph for both Cinder block storage and Glance images, plus Swift for object storage
- **Networking**: Neutron with OVS, using VLANs for provider networks and VXLAN for tenant isolation

I was involved in the deployment using OpenStack-Ansible, where I validated network configurations, created flavors and images, managed storage volumes, and troubleshot issues like stuck volumes, replication lag, and floating IP connectivity. I also handled live migrations during maintenance windows and implemented monitoring dashboards for capacity planning."

### Demonstrating Problem-Solving
Always include:
1. **The problem** - Specific issue encountered
2. **The diagnosis** - How you identified root cause
3. **The solution** - What you did to fix it
4. **The prevention** - How you avoided it in future

### Questions to Ask Interviewer
1. "What's your current OpenStack version, and are you planning any upgrades?"
2. "What storage backend are you using for Cinder and Glance?"
3. "How do you handle high availability for your control plane?"
4. "What's your approach to monitoring and alerting in the OpenStack environment?"
5. "Are you using any SDN overlay technologies beyond standard Neutron?"

---

## Final Tips

✅ **Do:**
- Speak in terms of business value (cost efficiency, compliance, agility)
- Mention specific technologies (KVM, Ceph, OVS, HAProxy)
- Share real problems you solved
- Show understanding of integration between services

❌ **Don't:**
- Claim expertise in areas you haven't touched
- Memorize commands without understanding concepts
- Ignore the "why"—always explain business reasoning
- Overcomplicate explanations—start simple, then go deeper if asked

**Remember:** Interviewers value practical experience, problem-solving ability, and clear communication over rote memorization of commands.