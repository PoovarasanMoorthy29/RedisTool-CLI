# Redis Cluster Lifecycle Tool

A CLI tool that provisions, operates, and performs **zero-downtime rolling upgrades** on a 6-node Redis Cluster (3 masters + 3 replicas) running in containers with Ansible automation.

---

## 📋 Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Commands Reference](#commands-reference)
- [Rolling Upgrade Strategy](#rolling-upgrade-strategy)
- [Project Structure](#project-structure)
- [Assumptions & Trade-offs](#assumptions--trade-offs)
- [Known Limitations](#known-limitations)

---

## 🏗 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    redis-tool CLI                       │
│         (Bash orchestrator, entry point)                │
└──────────┬──────────┬──────────┬───────────┬────────────┘
           │          │          │           │
     ┌─────▼──┐ ┌────▼────┐ ┌──▼───┐ ┌────▼─────┐
     │Provision│ │ Upgrade │ │Status│ │Data Seed │
     │  Flow   │ │  Flow   │ │Check │ │& Verify  │
     └────┬────┘ └────┬────┘ └──┬───┘ └────┬─────┘
          │           │         │           │
          ▼           ▼         ▼           ▼
    ┌─────────────────────────────────────────────┐
    │          Ansible Playbooks & Roles           │
    │  provision.yml | upgrade.yml | status.yml    │
    │  seed.yml      | verify.yml                  │
    └──────────────────┬──────────────────────────-┘
                       │ SSH (key-based)
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   ┌────────────┬────────────┬────────────┐
   │redis-node-1│redis-node-2│redis-node-3│  Masters
   │ 10.10.0.11 │ 10.10.0.12│ 10.10.0.13 │
   ├────────────┼────────────┼────────────┤
   │redis-node-4│redis-node-5│redis-node-6│  Replicas
   │ 10.10.0.14 │ 10.10.0.15│ 10.10.0.16 │
   └────────────┴────────────┴────────────┘
         Docker/Podman containers on 10.10.0.0/24
```

---

## ✅ Prerequisites

The tool auto-detects and validates prerequisites on every run:

| Tool | Minimum Version | Notes |
|------|----------------|-------|
| Docker or Podman | Any recent | Podman preferred if both exist |
| docker-compose / podman-compose | v2+ | For container orchestration |
| Ansible | 2.14+ | For configuration management |
| SSH client | Any | For key-based container access |
| Bash | 4.0+ | CLI entry point |

### Installing Prerequisites

```bash
# Docker (Ubuntu/Debian)
curl -fsSL https://get.docker.com | sh

# Ansible
pip3 install ansible>=2.14

# Podman (alternative to Docker)
sudo apt install podman podman-compose
```

---

## 🚀 Quick Start

```bash
# 1. Make the CLI executable
chmod +x redis-tool

# 2. Provision a 6-node cluster with Redis 7.0.15
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1

# 3. Seed 1000 deterministic test keys
./redis-tool data seed --keys 1000

# 4. Verify data integrity
./redis-tool data verify

# 5. Check cluster status
./redis-tool status

# 6. Perform zero-downtime rolling upgrade to 7.2.6
./redis-tool upgrade --target-version 7.2.6 --strategy rolling

# 7. Full post-upgrade verification
./redis-tool verify --full
```

---

## 📖 Commands Reference

### `provision`
Brings up 6 Ubuntu containers, installs Redis from source, and forms the cluster.

```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

| Flag | Default | Description |
|------|---------|-------------|
| `--version` | 7.0.15 | Redis version to install |
| `--masters` | 3 | Number of master nodes |
| `--replicas-per-master` | 1 | Replicas per master |

### `data seed`
Inserts deterministic key-value pairs where value = SHA256(key).

```bash
./redis-tool data seed --keys 1000
```

### `data verify`
Reads all keys, recomputes expected SHA256 values, compares, reports PASS/FAIL.

```bash
./redis-tool data verify --keys 1000
```

### `status`
Prints cluster info: nodes, roles, slots, keys, memory, cluster state.

```bash
./redis-tool status
```

### `upgrade`
Zero-downtime rolling upgrade with automatic failover orchestration.

```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```

### `verify --full`
Comprehensive verification: data integrity, version consistency, topology, slots, replication lag.

```bash
./redis-tool verify --full
```

### `infra`
Manage container infrastructure directly.

```bash
./redis-tool infra up    # Bring up containers
./redis-tool infra down  # Tear down containers + volumes
```

---

## 🔄 Rolling Upgrade Strategy

The upgrade follows a **replica-first, master-failover** strategy to achieve zero downtime:

### Phase 1: Pre-flight Checks
- Verify cluster state is `ok`
- Ping all 6 nodes for reachability
- Run baseline data verification (all 1000 keys intact)

### Phase 2: Upgrade Replicas (one-by-one)
For each replica node:
1. **Stop** Redis gracefully (`BGSAVE` → `SHUTDOWN SAVE`)
2. **Download & build** new Redis version from source
3. **Install** new binaries, preserving config and data
4. **Start** Redis with existing cluster config
5. **Wait** for node to rejoin cluster
6. **Verify** cluster returns to `cluster_state:ok`
7. **Proceed** to next replica only after health check passes

### Phase 3: Upgrade Masters (with failover)
For each master node:
1. **Identify** the master's replica
2. **CLUSTER FAILOVER** — send to the replica, promoting it to master
3. **Wait** for failover completion (replica reports `role:master`)
4. **Verify** cluster is healthy after role swap
5. **Upgrade** the old master (now a replica) — same stop/build/install/start cycle
6. **Wait** for cluster convergence
7. **Verify** cluster health before proceeding to next master

### Phase 4: Post-upgrade Verification
- Data integrity check (all 1000 keys)
- Version consistency (all nodes at target version)
- Full cluster status report

### Why Zero Downtime?
- **Replicas first**: upgrading a replica doesn't affect write availability
- **Failover before master upgrade**: the promoted replica serves traffic while the old master upgrades
- **One node at a time**: cluster maintains quorum throughout
- **Health gates**: no progression until cluster reports `ok`
- **Fail-fast**: any failure stops the entire process immediately

---

## 📁 Project Structure

```
RedisTool-CLI/
├── redis-tool                    ← CLI executable (Bash)
├── ansible/
│   ├── ansible.cfg               ← Ansible configuration
│   ├── inventory/
│   │   └── hosts.ini             ← 6 fixed IPs (10.10.0.11–16)
│   ├── playbooks/
│   │   ├── provision.yml         ← Install Redis + form cluster
│   │   ├── upgrade.yml           ← Upgrade single node
│   │   ├── status.yml            ← Cluster status report
│   │   ├── seed.yml              ← Insert deterministic data
│   │   └── verify.yml            ← Verify data integrity
│   └── roles/redis/
│       ├── tasks/main.yml        ← Build Redis from source
│       ├── handlers/main.yml     ← Restart/stop handlers
│       ├── templates/
│       │   ├── redis.conf.j2     ← Redis cluster config
│       │   └── redis.service.j2  ← Init script
│       └── defaults/main.yml     ← Default variables
├── infra/
│   ├── Dockerfile                ← Ubuntu + SSH container
│   └── compose.yml               ← 6 containers, static IPs
├── README.md                     ← This file
├── output/                       ← Command output logs
│   ├── provision_output.txt
│   ├── upgrade_output.txt
│   └── verify_output.txt
└── logs/                         ← Runtime logs
```

---

## ⚠️ Assumptions & Trade-offs

| Assumption | Rationale |
|-----------|-----------|
| Redis built from source | Ensures exact version control; no distro package lag |
| Ubuntu 22.04 containers | Stable LTS with good build toolchain |
| SSH key-based auth only | Security best practice; no password leaks |
| `cluster-announce-ip` per node | Required for Docker networking to work with Redis Cluster |
| Python3 on containers | Used for seed/verify scripts; pre-installed on Ubuntu |
| `appendonly yes` | Durability during upgrades; survives restart |
| `maxmemory 256mb` | Sufficient for testing; adjustable via role defaults |
| Single-threaded seed/verify | Simplicity over speed; uses `-c` flag for cluster routing |

---

## ⚡ Known Limitations

1. **Build time**: Compiling Redis from source takes 1-3 minutes per node on first provision
2. **No TLS**: Cluster communication is unencrypted (suitable for isolated Docker network)
3. **No persistence migration**: Upgrading assumes RDB/AOF format compatibility between versions
4. **Sequential upgrades**: Nodes upgraded one at a time (safe but slower)
5. **Fixed topology**: Initial cluster always uses nodes 1-3 as masters, 4-6 as replicas
6. **No automatic rollback**: If upgrade fails mid-way, manual intervention needed (see stretch goals)
7. **Host networking**: Uses Docker bridge network, not host networking

---

## 🎯 Stretch Goals Status

| Goal | Status | Notes |
|------|--------|-------|
| S1: Scale add nodes | ❌ Not implemented | Would use `redis-cli --cluster add-node` + `rebalance` |
| S2: Scale remove node | ❌ Not implemented | Would use `redis-cli --cluster del-node` + slot migration |
| S3: Rollback | ❌ Not implemented | Would reverse the upgrade process |
| S4: Idempotency | ✅ Partial | Provision checks if cluster already formed; skips if so |
| S5: Structured logging | ✅ Partial | Output captured to `output/` directory with timestamps |

---

## 📝 License

MIT
