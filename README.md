# Redis Cluster Lifecycle Tool (redis-tool)

A command-line interface orchestrator (`redis-tool`) that wraps Ansible to **provision, operate, and perform zero-downtime rolling upgrades/downgrades** on a 6-node containerized Redis Cluster (3 masters + 3 replicas).

This project conforms strictly to the requirements in [DevOps Engineer.pdf](file:///home/poovarasan/Desktop/RedisTool-CLI/DevOps%20Engineer.pdf).

---

## 1. System Architecture

```
┌──────────────────────────────────────────────────────────┐
│                     redis-tool CLI                       │
│              (Python orchestrator / entrypoint)          │
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
    │redis-node-1│redis-node-2│redis-node-3│  Masters (v7.0.15)
    │ 10.10.0.11 │ 10.10.0.12 │ 10.10.0.13 │
    ├────────────┼────────────┼────────────┤
    │redis-node-4│redis-node-5│redis-node-6│  Replicas (v7.0.15)
    │ 10.10.0.14 │ 10.10.0.15 │ 10.10.0.16 │
    └────────────┴────────────┴────────────┘
         Docker/Podman containers on 10.10.0.0/24
```

* **Control Node**: The host machine executing `redis-tool` and running Ansible.
* **Managed Nodes**: 6 containers running Ubuntu 22.04 with an active SSH daemon. The SSH keys are managed securely and local to the repository workspace to avoid password requirements.

---

## 2. Infrastructure Setup & Engine Compatibility

The infrastructure is defined in [infra/compose.yml](file:///home/poovarasan/Desktop/RedisTool-CLI/infra/compose.yml) and supports **both Docker and Podman** as container runtimes.

* **Dynamic Detection**: The tool automatically detects whether Docker or Podman is installed. If both are installed, it defaults to **Podman** as preferred. You can override the runtime using the `CONTAINER_ENGINE` environment variable.
* **Clean Transitions**: The orchestrator features a built-in cross-engine cleanup subsystem. When you deploy to one engine (e.g., Docker), the tool automatically stops and cleans up conflicting containers, volumes, and networks on the opposite engine (e.g., Podman) to free ports and IP subnets.

---

## 3. Prerequisites & Dependency Check

The tool performs an automated prerequisite check as the very first step of any command. It verifies:
1. **Container Runtime**: Podman or Docker is active.
2. **Ansible**: `ansible-playbook` is available and at version `2.14+`.

### System Dependency Map:
| Dependency | Version Requirement | Purpose |
|------------|---------------------|---------|
| **Podman** / **Docker** | Podman 4.0+ / Docker 20.10+ | Containerized server simulation |
| **Ansible** | 2.14+ | Compilation, packaging, configuration, and service orchestration |
| **Python** | 3.6+ | Runs the CLI wrapper (`redis-tool`) |

---

## 4. SSH Key Configuration

To ensure secure passwordless connections, the tool's pre-flight check automatically generates a dedicated SSH keypair in `infra/ssh_keys/` (`id_rsa` and `id_rsa.pub`) if they are missing.
* You can also place your own custom SSH keys in the `infra/ssh_keys/` directory before provisioning.
* These keys are automatically excluded from version control in `.gitignore`.

---

## 5. Command Reference & Execution Guide

Ensure the CLI tool is executable:
```bash
chmod +x redis-tool
```

### 1. Provision a Cluster
Deploy the 6-node container infrastructure, install Redis from source, configure clustering, start services, and join the cluster:
```bash
./redis-tool provision --version 7.0.15 --masters 3 --replicas-per-master 1
```

### 2. Seed Data
Generate and insert 1000 key-value pairs with deterministic, reproducible SHA256 hashes (e.g., `key:0001` → `SHA256("key:0001")`):
```bash
./redis-tool data seed --keys 1000
```

### 3. Verify Data
Read back the seeded keys, recompute expected values, and report verification results:
```bash
./redis-tool data verify
```

### 4. Check Cluster Status
Show a clear, human-readable summary of the cluster health, including node roles, master-replica mappings, slot allocations, key distribution, and memory usage:
```bash
./redis-tool status
```

### 5. Rolling Upgrade
Upgrade the cluster to a target version (e.g., `7.2.6`) with zero client-visible downtime:
```bash
./redis-tool upgrade --target-version 7.2.6 --strategy rolling
```

### 6. Full Health Verification
Run a comprehensive suite of post-upgrade health checks, verifying data integrity, version consistency, slots coverage, cluster state, and replication lag:
```bash
./redis-tool verify --full
```

### 7. Rollback
If an upgrade is unsuccessful or if a downgrade is requested, rollback the nodes to a previous target version:
```bash
./redis-tool rollback --target-version 7.0.15
```

### 8. Scale out / Scale In (Stretch Goals)
Add or remove nodes from the active cluster:
```bash
./redis-tool scale --add-nodes 2   
        # Add master + replica pair and rebalance
./redis-tool scale --remove-node <node-id>  # Reshard slots and decommission node
```

### 9. Infrastructure Teardown
Stop and delete all containerized nodes, networks, and volumes for both engines:
```bash
./redis-tool infra down
```

---

## 6. Zero-Downtime Rolling Upgrade Strategy

To guarantee **zero client-visible downtime** and maintain data availability throughout the upgrade process, we implement a **Replica-First, Failover-Promoted** migration strategy:

### Why this strategy?
If a Master node is upgraded directly, it goes offline, forcing the cluster to execute an unplanned failover which can drop client writes. Upgrading replicas first and then performing a coordinated failover ensures that all masters are always active during their software upgrade.

### Implementation Workflow:
```
               ┌────────────────────────────────────────┐
               │          Pre-flight checks             │
               │ (Check cluster state ok, seed verify)  │
               └───────────────────┬────────────────────┘
                                   │
                                   ▼
               ┌────────────────────────────────────────┐
               │         Upgrade Replicas First         │
               │  - Upgrade one replica at a time       │
               │  - Stop, build & install new Redis, start│
               │  - Wait for replication sync & ok state│
               └───────────────────┬────────────────────┘
                                   │
                                   ▼
               ┌────────────────────────────────────────┐
               │    Coordinated Master Upgrades         │
               │  For each Master:                      │
               │  - Run CLUSTER FAILOVER on its replica │
               │  - Replica is promoted to Master       │
               │  - Old Master is demoted to Replica    │
               │  - Upgrade old Master (now a replica)  │
               │  - Wait for replication sync & ok state│
               └───────────────────┬────────────────────┘
                                   │
                                   ▼
               ┌────────────────────────────────────────┐
               │       Post-upgrade Verification        │
               │  - Verify all nodes on target version  │
               │  - Run data verify check on all keys   │
               └────────────────────────────────────────┘
```

---

## 7. Assumptions & Design Trade-offs

1. **Source Compilation vs Packages**:
   * *Assumption*: The user wants to provision specific, arbitrary Redis versions (like `7.0.15` and `7.2.6`). Pre-built Debian/Ubuntu packages for these specific versions are not reliably available in the default package manager repositories.
   * *Trade-off*: We choose to compile Redis from source inside the target containers. This provides version flexibility but is highly CPU and memory intensive.
2. **Resource Constraints & Parallelism Mitigation**:
   * *Trade-off*: Running full parallel compilations across 6 containers concurrently on consumer-grade hardware (like a VM with 6GB RAM) triggers Out-Of-Memory (OOM) failures or SSH connection timeouts.
   * *Mitigation*: We set Ansible parallelism `forks = 2` in [ansible.cfg](file:///home/poovarasan/Desktop/RedisTool-CLI/ansible.cfg) and limit the compilation concurrency to `make -j2` in the Ansible playbooks. This reduces the peak compilation processes from 48 concurrent threads down to a safe maximum of 4, ensuring reliable deployment on constrained hosts.
3. **Dedicated Subnet Routing**:
   * *Assumption*: The container network bridge subnet `10.10.0.0/24` is free and not used by other virtual network interfaces on the host. 
4. **SSH Access in Containers**:
   * *Assumption*: Running SSH servers inside containers mimics real-world bare-metal or virtual server provisioning. We compile and configure SSH services directly inside the container images using a local keypair.

---

## 8. Known Limitations

* **Rootless Podman Routing**: On rootless Podman installations, container-to-container routing and host port mapping might require `slirp4netns` configuration. The orchestrator mitigates this by using the `bridge` driver with predefined static subnets and host-mapped port routing.
* **Host Port Bindings**: The tool maps container ports directly to the host ports (`2211-2216` for SSH and `7001-7006` for Redis). Running other services on these host ports will block container startup.
* **Control Node OS**: The Ansible orchestration playbooks require standard POSIX shell tools (`tar`, `ssh`, etc.) on the control host (Linux or macOS). Running the orchestrator on native Windows is not supported.

---

## 9. Directory Submission Structure

The repository structure conforms exactly to the requested submission format:

```
submission/
├── redis-tool                       ← CLI Entrypoint (Python wrapper)
├── ansible.cfg                      ← Root Ansible configuration
├── ansible/
│   ├── inventory/
│   │   └── hosts.ini                ← 6-node inventory configuration
│   ├── playbooks/
│   │   ├── provision.yml            ← Cluster provisioning playbook
│   │   ├── upgrade.yml              ← Zero-downtime upgrade playbook
│   │   ├── status.yml               ← Cluster status checks playbook
│   │   ├── seed.yml                 ← Data seed playbook
│   │   └── verify.yml               ← Data verification playbook
│   └── roles/redis/
│       ├── tasks/main.yml           ← Redis installer tasks
│       ├── templates/
│       │   ├── redis.conf.j2        ← Redis server config template
│       │   └── redis.service.j2     ← SysV init script template
│       └── defaults/main.yml        ← Default variables
├── infra/
│   ├── Containerfile                ← Base Ubuntu image with SSH daemon
│   └── compose.yml                  ← 6-node container compose file
├── logs/
│   └── operations.jsonl             ← Operation event audit logs
└── output/
    ├── provision_output.txt         ← Terminal outputs from provision
    ├── data_seed_output.txt         ← Terminal outputs from data seed
    ├── status_output.txt            ← Terminal outputs from status
    ├── upgrade_output.txt           ← Terminal outputs from upgrade
    └── verify_output.txt            ← Terminal outputs from verify
```
