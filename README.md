# Redis Cluster Lifecycle Tool

A CLI tool (`redis-tool`) that wraps Ansible to **provision, operate, and perform zero-downtime rolling upgrades** on a 6-node Redis Cluster (3 masters + 3 replicas) running inside containers with SSH access.

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     redis-tool CLI                       │
│              (Python orchestrator / entrypoint)           │
└────────┬──────────┬──────────┬───────────┬───────────────┘
         │          │          │           │
   ┌─────▼──┐ ┌────▼────┐ ┌──▼───┐ ┌────▼─────┐
   │Provision│ │ Upgrade │ │Status│ │Data Seed │
   │  Flow   │ │  Flow   │ │Check │ │& Verify  │
   └────┬────┘ └────┬────┘ └──┬───┘ └────┬─────┘
        │           │         │           │
        ▼           ▼         ▼           ▼
  ┌──────────────────────────────────────────────┐
  │          Ansible Playbooks & Roles            │
  │  provision.yml │ upgrade.yml │ status.yml     │
  │  seed.yml      │ verify.yml                   │
  └───────────────────┬──────────────────────────-┘
                      │ SSH (key-based)
         ┌────────────┼────────────────┐
         ▼            ▼                ▼
  ┌────────────┬────────────┬────────────┐
  │redis-node-1│redis-node-2│redis-node-3│  Masters
  │ 10.10.0.11 │ 10.10.0.12 │ 10.10.0.13 │
  ├────────────┼────────────┼────────────┤
  │redis-node-4│redis-node-5│redis-node-6│  Replicas
  │ 10.10.0.14 │ 10.10.0.15 │ 10.10.0.16 │
  └────────────┴────────────┴────────────┘
        Docker/Podman containers on 10.10.0.0/24
```

Your **host machine** is the Ansible control node. The 6 containers are managed nodes, accessed via SSH — mirroring real production infrastructure.

---

## Prerequisites

The tool auto-detects and validates prerequisites on every command:

| Tool | Requirement | Notes |
|------|-------------|-------|
| **Container Runtime** | Docker Engine or Podman | Podman preferred (Apache 2.0, no licensing concerns) |
| **Compose** | docker compose / podman-compose | For container orchestration |
| **Ansible** | 2.14+ | `ansible-playbook` must be available |
| **SSH client** | Any | For key-based container access |
| **Python 3** | 3.6+ | CLI is written in Python |

### Installing Prerequisites

```bash
# Podman (recommended — fully open source)
sudo apt install podman podman-compose

# Docker (alternative)
curl -fsSL https://get.docker.com | sh

# Ansible
pip3 install ansible>=2.14
```

### SSH Key Configuration

To facilitate secure, key-based container access without password prompts, the tool's pre-flight check automatically generates a dedicated SSH keypair in the `infra/ssh_keys/` directory (specifically `id_rsa` and `id_rsa.pub`) if they are missing.

> [!IMPORTANT]
> You must ensure you use your own secure SSH keys. If you wish to use a custom SSH keypair, place your keys in the `infra/ssh_keys/` directory (named `id_rsa` and `id_rsa.pub`) before provisioning the infrastructure.

These key files are used by Ansible to connect securely to the cluster nodes. They are explicitly ignored in `.gitignore` to ensure they are never committed or pushed to Git and GitHub.

---

## Quick Start

```bash
# Make the CLI executable
chmod +x redis-tool

# 1. Provision a 6-node cluster with Redis 7.0.15
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1

# 2. Seed 1000 deterministic test keys
./redis-tool data seed --keys 1000

# 3. Verify data integrity
./redis-tool data verify

# 4. Check cluster status
./redis-tool status

# 5. Perform zero-downtime rolling upgrade to 7.2.6
./redis-tool upgrade --target-version 7.2.6 --strategy rolling

# 6. Full post-upgrade verification
./redis-tool verify --full
```

---

## How to Bring Up the Container Infrastructure

The `provision` command handles everything automatically, but you can also manage infrastructure directly:

```bash
# Bring up 6 containers (auto-detects Docker or Podman)
./redis-tool infra up

# Tear down all containers and volumes
./redis-tool infra down
```

**With Docker:**
```bash
cd infra/
docker compose up -d --build
```

**With Podman:**
```bash
cd infra/
podman-compose up -d --build
```

The compose file creates 6 Ubuntu 22.04 containers with:
- SSH server running (key-based auth, no passwords)
- Static IPs on a dedicated bridge network (10.10.0.0/24)
- Build tools pre-installed for compiling Redis from source

---

## Commands Reference

### `provision` — Phase 1
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

Installs the exact Redis version from source on all 6 nodes, configures cluster mode (`cluster-enabled yes`), starts Redis, and forms the cluster with proper master/replica assignment and hash slot distribution.

| Flag | Default | Description |
|------|---------|-------------|
| `--version` | `7.0.15` | Exact Redis version to install |
| `--masters` | `3` | Number of master nodes |
| `--replicas-per-master` | `1` | Replicas per master |

### `data seed` — Phase 2
```bash
./redis-tool data seed --keys 1000
```

Generates 1000 deterministic key-value pairs (`key:0001` → `SHA256("key:0001")`) and inserts them into the cluster. Prints total inserted, distribution across masters, and any failures.

### `data verify` — Phase 2
```bash
./redis-tool data verify
```

Reads all 1000 keys, recomputes expected SHA256 values, compares, and reports:
- `PASS — 1000/1000 keys verified` or
- `FAIL — X keys missing, Y values mismatched`

### `status` — Phase 3
```bash
./redis-tool status
```

Prints a clear, human-readable cluster summary:
- Each node's IP, port, role (master/replica), and Redis version
- For masters: hash slot range and number of keys
- For replicas: which master they're replicating
- Cluster state: `ok` or `fail`
- Memory usage per node

### `upgrade` — Phase 4 (The Core Challenge)
```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```

Performs a zero-downtime rolling upgrade. See [Rolling Upgrade Strategy](#rolling-upgrade-strategy) below.

### `verify --full` — Phase 5
```bash
./redis-tool verify --full
```

Comprehensive post-upgrade health check:
1. **Data integrity** — all 1000 keys present and correct
2. **Version consistency** — all nodes report the same Redis version
3. **Topology health** — all 16384 hash slots covered, every master has a replica
4. **Cluster state** — `cluster_state:ok`
5. **Replication lag** — all replicas have `master_link_status:up`

### `infra up|down`
```bash
./redis-tool infra up    # Bring up containers
./redis-tool infra down  # Tear down containers + volumes
```

### `scale` — Stretch Goal S1/S2
```bash
./redis-tool scale --add-nodes 2           # Add master + replica pair, rebalance
./redis-tool scale --remove-node <node-id>  # Migrate slots, remove node
```

### `rollback` — Stretch Goal S3
```bash
./redis-tool rollback --target-version 7.0.15  # Roll back to previous version
```

---

## Rolling Upgrade Strategy

The upgrade achieves **zero client-visible downtime** using a replica-first, master-failover approach:

### Step 1: Pre-flight Checks
- Verify `cluster_state:ok`
- Verify all nodes are reachable (PING → PONG)
- Verify current version differs from target version
- Run `data verify` to establish a pre-upgrade integrity baseline

### Step 2: Upgrade Replicas First (one at a time)
For each replica node:
1. Stop Redis on the node
2. Build and install the new Redis version from source
3. Start Redis with the same configuration
4. Wait for the replica to rejoin the cluster and complete sync
5. Verify `cluster_state:ok` before moving to the next node
6. Print progress: `[N/6] Upgraded replica 10.10.0.XX — cluster: ok`

### Step 3: Upgrade Masters (one at a time, with failover)
For each master node:
1. `CLUSTER FAILOVER` on its replica — the replica becomes the new master
2. Wait for failover to complete
3. The old master is now a replica — stop Redis on it
4. Build and install the new Redis version
5. Start Redis, wait for it to rejoin as a replica
6. Verify `cluster_state:ok` before moving to the next master

### Step 4: Post-upgrade Verification
- Run `data verify` — all 1000 keys must still be present and correct
- Run `status` — all nodes must show the new version
- Print `UPGRADE COMPLETE — all nodes on v7.2.6, data integrity verified`

### Why Zero Downtime?
- **Replicas first**: upgrading a replica doesn't affect write availability
- **Failover before master upgrade**: the promoted (already-upgraded) replica serves traffic
- **One node at a time**: cluster maintains quorum throughout
- **Health gates**: no progression until cluster reports `ok`
- **Fail-fast**: any failure stops immediately, prints which node failed and at which step

---

## Project Structure

```
RedisTool-CLI/
├── redis-tool                       ← CLI entrypoint (Python)
├── ansible/
│   ├── ansible.cfg                  ← Ansible configuration
│   ├── inventory/
│   │   └── hosts.ini                ← 6+2 nodes with fixed IPs
│   ├── playbooks/
│   │   ├── provision.yml            ← Install Redis + form cluster
│   │   ├── upgrade.yml              ← Upgrade single node
│   │   ├── status.yml               ← Cluster status report
│   │   ├── seed.yml                 ← Insert deterministic data
│   │   └── verify.yml               ← Verify data integrity
│   └── roles/redis/
│       ├── tasks/main.yml           ← Build Redis from source
│       ├── handlers/main.yml        ← Restart/stop handlers
│       ├── templates/
│       │   ├── redis.conf.j2        ← Redis cluster config
│       │   └── redis.service.j2     ← Init script (SysV)
│       └── defaults/main.yml        ← Default variables
├── infra/
│   ├── Containerfile                ← Ubuntu 22.04 + SSH base image
│   ├── compose.yml                  ← 6-node cluster (+ 2 scale nodes)
│   └── ssh_keys/                    ← Auto-generated SSH keypair
├── README.md                        ← This file
├── output/                          ← Terminal output captures
│   ├── provision_output.txt
│   ├── data_seed_output.txt
│   ├── status_output.txt
│   ├── upgrade_output.txt
│   └── verify_output.txt
└── logs/                            ← Structured operation logs
    └── operations.jsonl             ← JSON Lines event log
```

---

## Assumptions & Trade-offs

| Assumption | Rationale |
|---|---|
| Redis built from source | Ensures exact version control; no distro package lag |
| Ubuntu 22.04 containers | Stable LTS with full build toolchain |
| SSH key-based auth only | Security best practice; no password leaks |
| `cluster-announce-ip` per node | Required for container networking to work with Redis Cluster |
| Python3 on containers | Used for seed/verify scripts; pre-installed on Ubuntu 22.04 |
| `appendonly yes` | Durability during upgrades; data survives restart |
| `maxmemory 256mb` | Sufficient for 1000-key testing; adjustable via role defaults |
| SysV init scripts (not systemd) | Containers don't run systemd by default; init.d scripts are reliable |
| Deterministic SHA256 values | Enables independent value recomputation for integrity verification |
| Single-threaded seed/verify | Simplicity over speed; uses `-c` flag for proper cluster routing |

---

## Known Limitations

1. **Build time**: Compiling Redis from source takes 1–3 minutes per node on first provision
2. **No TLS**: Cluster communication is unencrypted (suitable for isolated container network)
3. **No persistence format migration**: Assumes RDB/AOF format compatibility between versions
4. **Sequential upgrades**: Nodes upgraded one at a time (safe but slower)
5. **Fixed initial topology**: Nodes 1–3 start as masters, 4–6 as replicas (may change after failovers)
6. **No automatic rollback on failure**: If upgrade fails mid-way, use `rollback` command manually
7. **Bridge networking**: Uses Docker/Podman bridge network, not host networking

---

## Stretch Goals

| Goal | Status | Description |
|------|--------|-------------|
| S1: Scale Out | ✅ | `./redis-tool scale --add-nodes 2` — adds master + replica pair, rebalances slots |
| S2: Scale In | ✅ | `./redis-tool scale --remove-node <id>` — migrates slots, removes node |
| S3: Rollback | ✅ | `./redis-tool rollback --target-version 7.0.15` — rolls back to previous version |
| S4: Idempotency | ✅ | `provision` detects existing cluster; `upgrade` skips if already at target version |
| S5: Structured Logging | ✅ | JSON Lines event log at `logs/operations.jsonl` with timestamps and outcomes |
| Auto-install | ✅ | `--auto-install` flag offers to install missing Podman/Ansible with confirmation |

---

## Rules Compliance

- ✅ Custom Ansible playbooks and roles (no Galaxy roles)
- ✅ Uses only Ansible built-in modules (`command`, `shell`, `template`, `copy`, `apt`, `wait_for`, etc.)
- ✅ No managed Redis services
- ✅ CLI orchestrates Ansible (does not replace it)
- ✅ Supports both Docker Engine and Podman (auto-detection, Podman preferred)
- ✅ Prerequisite check on every command
- ✅ Data operations use `redis-cli` on nodes via Ansible

---

## License

MIT
