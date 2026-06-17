# Troubleshooting Guide

This guide documents common errors, pre-flight check failures, and runtime issues you might encounter while using the Redis Cluster Lifecycle Tool, along with their causes and step-by-step instructions to resolve them.

---

## Table of Contents
1. [Subnet Already Reserved (IP Conflict)](#1-subnet-already-reserved-ip-conflict)
2. [SSH Forward Ports in Use](#2-ssh-forward-ports-in-use)
3. [Container Runtime Not Found / Compose Missing](#3-container-runtime-not-found--compose-missing)
4. [Ansible Not Found / Version Incompatible](#4-ansible-not-found--version-incompatible)
5. [SSH Not Reachable after 30 Attempts](#5-ssh-not-reachable-after-30-attempts)
6. [Pre-flight Cluster Health Check Failed](#6-pre-flight-cluster-health-check-failed)
7. [Verification Failures (Replication Lag / Slot Coverage)](#7-verification-failures-replication-lag--slot-coverage)
8. [Scaling Constraints / Configuration Errors](#8-scaling-constraints--configuration-errors)
9. [Redis Download Failed (HTTP Error 404: Not Found)](#9-redis-download-failed-http-error-404-not-found)

---

## 1. Subnet Already Reserved (IP Conflict)

### Error Message
```text
[✘] Required subnet 10.10.0.0/24 is already reserved on this host.
    Route: 10.10.0.0/24 dev br-xxxxx proto kernel scope link src 10.10.0.1 linkdown
    This blocks Podman/Docker from creating infra_redis-cluster-net.
```

### Cause
A custom bridge network interface (usually named like `br-<network_id>`) from a previous run or another container network was not cleaned up properly by the system kernel, even if Docker or Podman claims the network no longer exists. The kernel still keeps the IP route, blocking the tool from reserving `10.10.0.0/24`.

### Solution
1. Identify the name of the bridge interface from the error output (e.g., `br-192f1b51c603`).
2. Delete the stale interface from the host kernel using `sudo`:
   ```bash
   sudo ip link delete br-<bridge-name>
   ```
3. Re-run your infrastructure or provision command:
   ```bash
   ./redis-tool infra up
   ```

---

## 2. SSH Forward Ports in Use

### Error Message
```text
[✘] Required SSH forward ports are already in use.
    Busy listeners:
      LISTEN 0 128 0.0.0.0:2211 ...
    If these are orphaned Podman rootlessport helpers from a failed run, clear them:
      kill <pids>
```

### Cause
Another process on the host is already listening on one of the required SSH forwarding ports (ports `2211` through `2218`). This is often caused by:
- Stale `rootlessport` helper processes left behind by a previous failed Podman execution.
- Another service (like an SSH daemon or local service) bound to those ports.

### Solution
1. Terminate the blocking process(es) indicated by the `kill <pids>` helper command output:
   ```bash
   kill <pid1> <pid2> ...
   ```
2. If the processes belong to rootless Podman, you can also perform a system-wide reset:
   ```bash
   podman system prune --volumes -f
   ```
3. Re-run your command:
   ```bash
   ./redis-tool infra up
   ```

---

## 3. Container Runtime Not Found / Compose Missing

### Error Message
```text
[✘] Container runtime not found (Docker or Podman)
```
or
```text
[✘] Docker found but no compose plugin available
```

### Cause
The orchestration tool relies on either **Podman** (preferred) or **Docker** along with their compose utilities (`podman-compose`, `podman compose`, or `docker compose`) to provision the 6-node network environment.

### Solution
* **For Podman (Recommended)**:
  Install Podman and its compose plugin:
  ```bash
  # Debian/Ubuntu
  sudo apt-get update && sudo apt-get install -y podman podman-compose
  ```
* **For Docker**:
  Ensure both Docker Engine and the modern Compose plugin are installed:
  ```bash
  # Install Docker Compose Plugin
  sudo apt-get install docker-compose-plugin
  ```

---

## 4. Ansible Not Found / Version Incompatible

### Error Message
```text
[✘] Ansible not found
```
or
```text
[✘] Ansible 2.14+ required, found <version>
```

### Cause
Ansible is required to run the automated provisioning, upgrading, scaling, and rolling upgrade tasks. The playbooks use modern Ansible modules and syntax that require version **2.14 or greater**.

### Solution
Install or upgrade Ansible using `pip3`:
```bash
pip3 install --upgrade "ansible>=2.14"
```

---

## 5. SSH Not Reachable after 30 Attempts

### Error Message
```text
[✘] SSH not reachable on 10.10.0.x after 30 attempts
```

### Cause
The containers were successfully started by the container runtime, but the orchestrator cannot establish an SSH connection to them. This typically occurs because:
- The SSH daemon inside the container failed to start.
- The SSH public keys configured in the container build do not match the keys in `infra/ssh_keys/`.
- Local firewall/routing rules are blocking container-to-host bridge routing.

### Solution
1. Tear down the corrupted container setup:
   ```bash
   ./redis-tool infra down
   ```
2. Delete the generated SSH keys to force the tool to regenerate them:
   ```bash
   rm -rf infra/ssh_keys/
   ```
3. Re-run the provision flow:
   ```bash
   ./redis-tool provision --version <version>
   ```

---

## 6. Pre-flight Cluster Health Check Failed

### Error Message
```text
[✘] upgrade pre-flight check failed: Cluster is not healthy
```
or
```text
[✘] scale pre-flight check failed: Could not query cluster state: <error>
```

### Cause
Before performing topology-altering operations (like upgrading Redis versions or scaling nodes), the tool verifies that the cluster is currently online, all hash slots (16384 slots) are successfully covered, and there are no failed/unreachable master nodes.

### Solution
1. Check the status of the cluster:
   ```bash
   ./redis-tool status
   ```
2. Look for offline nodes or unassigned slots. If nodes are down, check the container logs:
   ```bash
   # If using Podman
   podman logs redis-node-1
   
   # If using Docker
   docker logs redis-node-1
   ```
3. Ensure all nodes can communicate and restart any stopped containers:
   ```bash
   ./redis-tool infra up
   ```

---

## 7. Verification Failures (Replication Lag / Slot Coverage)

### Error Message
```text
[✘] Slot coverage: 15300/16384
```
or
```text
[✘] Replication lag: FAIL — some replicas have master_link_status:down
```

### Cause
After a rolling upgrade or scaling operation, the verification step tests for full cluster integrity.
- **Slot coverage failures** mean some slots are in an `importing` or `migrating` state, or a master went offline without successfully promoting its replica.
- **Replication link failures** indicate that replica nodes are unable to connect to their masters (often due to networking blocks or password authentication issues).

### Solution
1. Execute a check of the nodes:
   ```bash
   ./redis-tool status
   ```
2. Manually fix slot assignments if they got stuck during a migrate:
   ```bash
   # Connect to a working master node and run cluster check
   podman exec -it redis-node-1 redis-cli --cluster check 10.10.0.11:6379
   
   # Automatically fix slot coverage issues
   podman exec -it redis-node-1 redis-cli --cluster fix 10.10.0.11:6379
   ```

---

## 8. Scaling Constraints / Configuration Errors

### Error Message
```text
[✘] Scaling nodes (node 7 and 8) are not defined/configured in composition configurations.
```
or
```text
[✘] Currently only supports adding exactly 2 nodes (1M + 1R)
```

### Cause
The scale-out design adds exactly one master and one replica pairing at a time (total of 2 nodes) to maintain cluster parity. If you specify any other count or if `redis-node-7` and `redis-node-8` parameters are missing from `infra/compose.yml` or `ansible/inventory/hosts.ini`, the precheck fails.

### Solution
- Ensure you scale by exactly 2 nodes:
  ```bash
  ./redis-tool scale --add-nodes 2
  ```
- Verify your configuration has provisions for `redis-node-7` (IP `10.10.0.17`, port `2217`) and `redis-node-8` (IP `10.10.0.18`, port `2218`).

---

## 9. Redis Download Failed (HTTP Error 404: Not Found)

### Error Message
```text
fatal: [redis-node-1 -> localhost]: FAILED! => changed=false 
  dest: /tmp/redis-x.y.z.tar.gz
  msg: Request failed
  response: 'HTTP Error 404: Not Found'
  status_code: 404
  url: https://download.redis.io/releases/redis-x.y.z.tar.gz
```

### Cause
The Redis version specified in the command arguments (`--version` or `--target-version`) is invalid or does not exist on the official Redis release download registry.

### Solution
1. Verify the list of valid official Redis releases by visiting [download.redis.io/releases](https://download.redis.io/releases/).
2. Run the command again with a valid, existing version number (for example, `7.0.15` instead of `7.0.45`):
   ```bash
   ./redis-tool provision --version 7.0.15
   ```
