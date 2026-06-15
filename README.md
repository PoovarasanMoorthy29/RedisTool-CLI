# Redis Cluster Lifecycle Tool

A comprehensive DevOps tool for managing Redis cluster lifecycle operations including provisioning, data seeding, verification, and zero-downtime rolling upgrades. Built with Python, Ansible, and Docker/Podman for container orchestration.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Infrastructure](#infrastructure)
- [CLI Commands](#cli-commands)
- [Project Structure](#project-structure)
- [Docker Usage](#docker-usage)
- [Podman Usage](#podman-usage)
- [Rolling Upgrade Strategy](#rolling-upgrade-strategy)
- [Troubleshooting](#troubleshooting)

## Features

- ✅ **6-Node Redis Cluster** with 3 masters and configurable replicas
- ✅ **SSH-based deployment** with Ansible automation
- ✅ **Container runtime flexibility** - Docker or Podman
- ✅ **Deterministic data seeding** with SHA256 hashing
- ✅ **Comprehensive cluster verification** - health, data integrity, topology
- ✅ **Zero-downtime rolling upgrades** with automated failover
- ✅ **Structured logging** with JSON output
- ✅ **Static IP addressing** (10.10.0.0/24 subnet)
- ✅ **Idempotent operations** - safe to run multiple times

## Prerequisites

### System Requirements

- **Ubuntu/Debian**: 18.04 LTS or newer
- **RHEL/CentOS**: 7.x or newer
- **4GB+ RAM** (8GB recommended for development)
- **10GB+ disk space**

### Required Software

Before using this tool, ensure you have:

1. **Docker or Podman**
   ```bash
   # Ubuntu/Debian
   sudo apt-get update
   sudo apt-get install -y docker.io podman
   
   # RHEL/CentOS
   sudo yum install -y docker podman
   
   # macOS
   brew install docker podman
   ```

2. **Ansible >= 2.14**
   ```bash
   # Ubuntu/Debian
   sudo apt-get install -y ansible
   
   # RHEL/CentOS
   sudo yum install -y ansible
   
   # Via pip (all platforms)
   pip install ansible>=2.14
   ```

3. **Python 3.8+** with redis-py
   ```bash
   pip install redis
   ```

4. **SSH keys generated**
   ```bash
   ssh-keygen -f infra/id_rsa -N ""
   ```

### Verification

Check that all prerequisites are installed:

```bash
podman --version    # or docker --version
ansible --version
python3 --version
redis-cli --version
```

## Quick Start

### 1. Setup

```bash
# Clone or navigate to the project
cd /path/to/RedisTool-CLI

# Make the CLI executable
chmod +x redis-tool

# Generate SSH keys (if not present)
ssh-keygen -f infra/id_rsa -N ""
```

### 2. Build and Start Containers

```bash
# Using Docker Compose (with Docker)
docker-compose -f infra/compose.yml up -d

# Using Podman Compose (with Podman)
podman-compose -f infra/compose.yml up -d
```

### 3. Provision Redis Cluster

```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

### 4. Seed Test Data

```bash
./redis-tool data seed --keys 1000
```

### 5. Verify Cluster

```bash
./redis-tool status
./redis-tool verify --full
```

## Infrastructure

### Network Configuration

```
Network: 10.10.0.0/24
Driver: Bridge (compatible with Docker and Podman)

Nodes:
├── redis-node-1: 10.10.0.11:6379 (Master)
├── redis-node-2: 10.10.0.12:6379 (Master)
├── redis-node-3: 10.10.0.13:6379 (Master)
├── redis-node-4: 10.10.0.14:6379 (Replica of node-1)
├── redis-node-5: 10.10.0.15:6379 (Replica of node-2)
└── redis-node-6: 10.10.0.16:6379 (Replica of node-3)
```

### Node Configuration

Each node runs:
- **Ubuntu 22.04 LTS** base image
- **SSH Server** for Ansible access
- **Redis** (compiled from source to requested version)
- **Python3** for automation

### Cluster Setup

- **Cluster Mode**: Enabled
- **Replication**: 3 masters + 3 replicas (1:1 ratio)
- **Slot Distribution**: 5,462 slots per master
- **Node Timeout**: 5000ms
- **Cluster Persistence**: Enabled (AOF)

## CLI Commands

### Global Options

All commands check prerequisites before execution and provide installation instructions if dependencies are missing.

### provision

Provision a new Redis cluster with specified topology.

```bash
./redis-tool provision --version <VERSION> [--masters <N>] [--replicas-per-master <N>]

Options:
  --version              Redis version (e.g., 7.0.15, 7.2.6)
  --masters              Number of master nodes (default: 3)
  --replicas-per-master  Replicas per master (default: 1)

Example:
  ./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

**Output**: Provision log saved to `logs/provision.log`

### data seed

Seed the cluster with deterministic test data.

```bash
./redis-tool data seed [--keys <N>]

Options:
  --keys  Number of keys to seed (default: 1000)

Example:
  ./redis-tool data seed --keys 1000
```

**Data Format**: 
- Keys: `key:0001`, `key:0002`, ..., `key:1000`
- Values: SHA256 hash of the key name
- Deterministic: Same key always produces same value

### data verify

Verify data integrity of seeded data.

```bash
./redis-tool data verify
```

**Output**: 
```
✓ PASS: 1000/1000 keys verified successfully!
# or
✗ FAIL: X keys missing or corrupted.
```

### status

Display detailed cluster status.

```bash
./redis-tool status
```

**Output**:
```
Cluster State: ok
Redis Version: 7.0.15
Total Nodes: 6

MASTERS:
  10.10.0.11:6379
    Role: master
    Slots: 0-5461
    Keys: 334
    Memory: 2.45M

REPLICAS:
  10.10.0.14:6379
    Role: replica
    Keys: 334
    Memory: 2.45M
```

### verify

Verify cluster health.

```bash
./redis-tool verify [--full]

Options:
  --full  Perform comprehensive health check including all subsystems

Example:
  ./redis-tool verify --full
```

**Output**:
```
FULL CLUSTER VERIFICATION

1. Checking cluster state...
   ✓ Cluster state: OK

2. Checking node reachability...
   ✓ redis-node-1: OK
   ✓ redis-node-2: OK
   ...

3. Checking slot coverage...
   ✓ All 16384 slots covered

4. Checking version consistency...
   ✓ All nodes running same version: 7.0.15

5. Checking replication status...
   ✓ Masters: 3, Replicas: 3

6. Checking data integrity...
   ✓ All 1000/1000 keys verified successfully!

✓ FULL VERIFICATION PASSED
```

### upgrade

Perform zero-downtime rolling upgrade of the cluster.

```bash
./redis-tool upgrade --target-version <VERSION> [--strategy <STRATEGY>]

Options:
  --target-version  Target Redis version (e.g., 7.2.6)
  --strategy        Upgrade strategy: rolling (default)

Example:
  ./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```

**Output**: Upgrade progress with phase completion indicators

## Project Structure

```
RedisTool-CLI/
├── redis-tool                          # Main CLI tool (Python)
├── infra/
│   ├── compose.yml                     # Docker/Podman compose file
│   ├── Dockerfile                      # Ubuntu-based Redis container
│   ├── id_rsa                          # SSH private key (generated)
│   └── id_rsa.pub                      # SSH public key (generated)
├── ansible/
│   ├── ansible.cfg                     # Ansible configuration
│   ├── inventory/
│   │   └── hosts.ini                   # Node inventory with IPs
│   ├── playbooks/
│   │   ├── provision.yml               # Initial provisioning playbook
│   │   ├── upgrade.yml                 # Node upgrade playbook
│   │   ├── status.yml                  # Status check playbook
│   │   ├── verify.yml                  # Verification playbook
│   │   └── seed.yml                    # Data seeding reference
│   └── roles/
│       └── redis/
│           ├── tasks/
│           │   └── main.yml            # Role tasks (install, configure)
│           ├── handlers/
│           │   └── main.yml            # Service restart handlers
│           ├── defaults/
│           │   └── main.yml            # Default variables
│           └── templates/
│               └── redis.conf.j2       # Redis configuration template
├── output/
│   ├── provision_output.txt            # Provision command output
│   ├── data_seed_output.txt            # Data seeding output
│   ├── status_output.txt               # Status command output
│   ├── verify_output.txt               # Verification output
│   └── upgrade_output.txt              # Upgrade output
├── logs/                               # Structured JSON logs
│   ├── provision.log
│   ├── seed.log
│   ├── status.log
│   ├── verify.log
│   ├── upgrade.log
│   └── verify_full.log
└── README.md                           # This file
```

## Docker Usage

### Prerequisites

```bash
# Ensure Docker is running
sudo systemctl start docker

# Or use Docker Desktop on macOS/Windows
```

### Build and Run

```bash
# Build images and start containers
docker-compose -f infra/compose.yml up -d

# List running containers
docker-compose -f infra/compose.yml ps

# View logs
docker-compose -f infra/compose.yml logs -f redis-node-1

# Stop and remove containers
docker-compose -f infra/compose.yml down

# Remove volumes (WARNING: deletes all data)
docker-compose -f infra/compose.yml down -v
```

### Manual Container Commands

```bash
# Execute command in container
docker exec redis-node-1 redis-cli cluster info

# Access container shell
docker exec -it redis-node-1 bash

# View container resource usage
docker stats redis-node-1

# Inspect container
docker inspect redis-node-1 | grep -E '"IPAddress"'
```

## Podman Usage

### Prerequisites

```bash
# Start Podman service (on Linux)
sudo systemctl start podman

# Note: Podman on macOS/Windows requires a VM
```

### Build and Run

```bash
# Build images and start containers
podman-compose -f infra/compose.yml up -d

# List running containers
podman-compose -f infra/compose.yml ps

# View logs
podman-compose -f infra/compose.yml logs -f redis-node-1

# Stop and remove containers
podman-compose -f infra/compose.yml down
```

### Key Differences from Docker

- **Rootless**: Podman runs containers without root by default (more secure)
- **Pod Management**: Podman uses pods similar to Kubernetes
- **Compatibility**: Full Docker Compose compatibility

### Manual Container Commands

```bash
# Execute command in container
podman exec redis-node-1 redis-cli cluster info

# Access container shell
podman exec -it redis-node-1 bash

# List networks
podman network ls

# Inspect network
podman network inspect redis_net
```

## Rolling Upgrade Strategy

The tool implements a safe, zero-downtime rolling upgrade strategy:

### Phase 1: Pre-flight Checks
1. Verify cluster state is OK
2. Verify all nodes are reachable
3. Verify data integrity
4. Abort if any check fails

### Phase 2: Upgrade Replicas (sequential)
1. For each replica node (redis-node-4, 5, 6):
   - Stop Redis gracefully
   - Download and compile new version
   - Start Redis with new binary
   - Wait for node to rejoin cluster
   - Verify cluster state returns to OK

### Phase 3: Upgrade Masters (with failover)
1. For each master node (redis-node-1, 2, 3):
   - Trigger CLUSTER FAILOVER (promotes replica, demotes master)
   - Wait for promotion to complete
   - Stop Redis on old master (now replica)
   - Download and compile new version
   - Start Redis
   - Wait for node to rejoin cluster as replica
   - Verify cluster state returns to OK

### Phase 4: Post-upgrade Verification
1. Run full cluster verification
2. Verify all data integrity
3. Confirm all nodes running target version
4. Display upgrade completion summary

### Safety Features
- **Automatic Rollback**: Stops immediately if any node fails to upgrade
- **Data Protection**: Graceful shutdown with NOSAVE to preserve data
- **Replication Verification**: Waits for cluster state OK before proceeding
- **Comprehensive Logging**: All operations logged with timestamps

## Assumptions

1. **Network Connectivity**: SSH access from host to containers via 10.10.0.0/24 network
2. **Container Runtime**: Either Docker or Podman is available
3. **Ansible Installation**: Ansible >= 2.14 installed on host
4. **Python 3.8+**: Required for redis-py library
5. **Root/Sudo Access**: Required for container operations (docker/podman groups)
6. **SSH Keys**: Must be generated before provisioning (`ssh-keygen -f infra/id_rsa -N ""`)

## Limitations

1. **Single Physical Host**: All 6 nodes run on same machine
2. **Development Use**: Not recommended for production without modification
3. **Storage**: Data persisted only during container runtime (AOF mode)
4. **Networking**: Containers can only communicate via container network (10.10.0.0/24)
5. **Scaling**: Cluster size is fixed at 6 nodes (can be modified in compose.yml)
6. **Upgrade Strategy**: Currently only supports "rolling" strategy

## Troubleshooting

### Issue: "Missing prerequisites" error

**Solution**: Install missing software
```bash
# Install Docker
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Install Ansible
sudo apt-get install -y ansible

# Install Python Redis
pip install redis
```

### Issue: Container fails to start

**Solution**: Check Docker/Podman logs
```bash
docker-compose -f infra/compose.yml logs redis-node-1
# or
podman-compose -f infra/compose.yml logs redis-node-1
```

### Issue: SSH connection refused

**Solution**: Ensure SSH keys exist and containers are running
```bash
# Generate SSH keys
ssh-keygen -f infra/id_rsa -N ""

# Verify containers are running
docker-compose -f infra/compose.yml ps
# or
podman-compose -f infra/compose.yml ps

# Test SSH manually
ssh -i infra/id_rsa root@10.10.0.11
```

### Issue: Cluster not forming

**Solution**: 
1. Wait for containers to fully start (30-60 seconds)
2. Verify all containers are running: `docker ps`
3. Check Redis logs: `docker exec redis-node-1 tail /var/log/redis.log`
4. Manually form cluster:
   ```bash
   docker exec redis-node-1 redis-cli --cluster create \
     10.10.0.11:6379 10.10.0.12:6379 10.10.0.13:6379 \
     10.10.0.14:6379 10.10.0.15:6379 10.10.0.16:6379 \
     --cluster-replicas 1 --cluster-yes
   ```

### Issue: Data verification fails

**Solution**: Check if data was seeded
```bash
./redis-tool data seed --keys 1000
./redis-tool data verify
```

### Issue: Upgrade fails midway

**Solution**: 
1. Check logs: `cat logs/upgrade.log`
2. Manually verify cluster state:
   ```bash
   docker exec redis-node-1 redis-cli cluster info
   docker exec redis-node-1 redis-cli cluster nodes
   ```
3. If cluster is stuck, restart affected node:
   ```bash
   docker restart redis-node-X
   ```

## Logging

All operations generate structured JSON logs in the `logs/` directory:

```json
{
  "timestamp": "2025-06-15T10:30:45.123456",
  "operation": "provision",
  "level": "INFO",
  "message": "Provision completed successfully"
}
```

## Version Compatibility

- **Redis**: 5.0+ (tested with 7.0.15, 7.2.6)
- **Ansible**: 2.14+
- **Python**: 3.8+
- **Docker**: 20.10+
- **Podman**: 3.0+

## Performance Tuning

For production-like testing:

1. **Increase VM memory** to 16GB+
2. **Adjust redis.conf parameters**:
   ```
   maxmemory 2gb
   maxmemory-policy allkeys-lru
   ```
3. **Use separate storage** for each node
4. **Monitor with**: `redis-cli --stat`

## Support & Contribution

For issues, questions, or contributions, refer to the project repository.

## License

Internal DevOps Tool - All Rights Reserved
