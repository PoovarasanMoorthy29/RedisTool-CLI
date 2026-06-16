# Redis Cluster Lifecycle Tool

A command-line interface orchestrator (`redis-tool`) that wraps Ansible to **provision, operate, and perform zero-downtime rolling upgrades/downgrades** on a 6-node containerized Redis Cluster (3 masters + 3 replicas).

---

## 1. System Architecture

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
   └───────────────────┬───────────────────────────┘
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

* **Control Node**: Your local host machine running the Python orchestrator and Ansible.
* **Managed Nodes**: 6 Ubuntu 22.04 containers representing real production servers, accessed via key-based SSH.

---

## 2. Prerequisites

The tool automatically validates dependencies prior to executing any command:

| Tool | Version Requirement | Purpose |
|------|-------------|-------|
| **Container Runtime** | Podman (preferred) or Docker Engine | Hosts the cluster nodes |
| **Compose** | `podman-compose` or `docker compose` | Coordinates container network and volumes |
| **Ansible** | 2.14+ | Performs software compilation and cluster configuration |
| **Python 3** | 3.6+ | Runs the CLI wrapper |

---

## 3. SSH Key Configuration

To ensure secure passwordless connections, the tool's pre-flight check automatically generates a dedicated SSH keypair in `infra/ssh_keys/` (`id_rsa` and `id_rsa.pub`) if they are missing.

> [!IMPORTANT]  
> You must ensure you use your own secure SSH keys. If you wish to use a custom SSH keypair, place your keys in the `infra/ssh_keys/` directory (named `id_rsa` and `id_rsa.pub`) before provisioning the infrastructure.
>
> To avoid security leaks, these keys are ignored via Git settings.

---

## 4. Quick Start

Get your cluster up and running with a single sequence:

```bash
# Make the CLI tool executable
chmod +x redis-tool

# 1. Provision a 6-node cluster running Redis 7.0.15
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1

# 2. Seed 1000 test keys
./redis-tool data seed --keys 1000

# 3. Verify data integrity
./redis-tool data verify

# 4. Check cluster status
./redis-tool status

# 5. Perform a zero-downtime rolling upgrade to Redis 7.2.6
./redis-tool upgrade --target-version 7.2.6 --strategy rolling

# 6. Run full post-upgrade cluster verification
./redis-tool verify --full
```

---

## 5. Commands Reference

### `provision`
```bash
./redis-tool provision --version <ver> --masters <count> --replicas-per-master <count>
```
* Compiles and installs the specified Redis version from source on all nodes.
* Configures cluster configurations, starts services, and forms the cluster topology.

### `data seed`
```bash
./redis-tool data seed --keys <count>
```
* Seeds deterministic key-value pairs (`key:0001` → `SHA256("key:0001")`).

### `data verify`
```bash
./redis-tool data verify --keys <count>
```
* Re-queries all keys, validates values against expected SHA256 hashes, and reports mismatch counts.

### `status`
```bash
./redis-tool status
```
* Displays cluster state (`ok`/`fail`), role assignments, memory consumption, and slots coverage.

### `upgrade`
```bash
./redis-tool upgrade --target-version <ver> --strategy rolling
```
* Performs a zero-downtime rolling upgrade of all nodes.

### `rollback`
```bash
./redis-tool rollback --target-version <ver>
```
* Downgrades nodes, re-seeds data, and verifies integrity.

### `scale`
```bash
./redis-tool scale --add-nodes 2           # Add master + replica pair and rebalance
./redis-tool scale --remove-node <node-id>  # Reshard slots and remove node
```

### `verify`
```bash
./redis-tool verify [--full]
```
* Runs data integrity checks, version consistency, slots coverage, cluster state, and replica lag checks.

---

## 6. Zero-Downtime Rolling Upgrade Strategy

The upgrade executes a replica-first, failover-promoted sequence to ensure continuous availability:

1. **Pre-flight Check**: Verifies cluster health (`ok`) and takes a baseline database backup verify.
2. **Upgrade Replicas First**: One replica node is taken down, built from source with the target version, restarted, and synced back to its master.
3. **Upgrade Masters (with Failover)**:
   * Promotes the newly upgraded replica to Master using `CLUSTER FAILOVER`.
   * Upgrades the old master (which is now a replica).
   * Verifies replication status and cluster stabilization before moving to the next node.
4. **Post-Upgrade verification**: Ensures all keys remain intact and all nodes report the target version.

---

## 7. Project Structure

```
RedisTool-CLI/
├── redis-tool                       ← Python CLI Entrypoint
├── ansible/
│   ├── ansible.cfg                  ← Ansible configurations
│   ├── inventory/
│   │   └── hosts.ini                ← Node IP inventory mappings
│   ├── playbooks/
│   │   ├── provision.yml            ← Cluster setup playbook
│   │   ├── upgrade.yml              ← Node upgrade playbook
│   │   ├── status.yml               ← Cluster health report
│   │   ├── seed.yml                 ← Data injection playbook
│   │   └── verify.yml               ← Integrity check playbook
│   └── roles/redis/
│       ├── tasks/main.yml           ← Redis installation tasks
│       ├── templates/
│       │   ├── redis.conf.j2        ← Redis configuration template
│       │   └── redis.service.j2     ← SysV init service template
│       └── defaults/main.yml        ← Defaults variables
├── infra/
│   ├── Containerfile                ← Dockerfile for SSH node image
│   └── compose.yml                  ← Cluster container services
└── logs/
    └── operations.jsonl             ← Event audit logs
```

---

## 8. License

MIT
