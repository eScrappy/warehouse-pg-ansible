# WarehousePG Ansible Roles

Ansible roles for deploying and managing WarehousePG clusters.

## Roles

### edb-repository

Installs the EDB package repository on Red Hat based systems. This role must run before `warehouse-pg`, in order to make the WarehousePG packages available.

#### EDB Token

This role requires an EDB token and a repository name.

Set the variable `$edb_token` with the token, and the variable `$edb_repository` with the repository name.

The repository name defaults to `dev`, can also be `gsupp` or another available repository.
Ask your EDB contact for details.

#### Installation process

This role must run as root, and therefore requires Ansible `become` privileges (usually `sudo`). It will download a script which sets up the repository.

If you do not want to run arbitrary scripts (`setup.sh`) as root, exclude this role and setup the repository manually.

### warehouse-pg

Deploys and configures a WarehousePG cluster. Handles pre-flight validation, OS-level user/group setup, SSH key exchange, package installation, data directory calculation, and cluster hostfile generation.

**Task flow:**

1. `pre-flight-checks.yml`: Validates all required and optional variables
2. `unix-user.yml`: Creates the user and group specified in `$WHPG_UNIX_USER` and `$WHPG_UNIX_GROUP` (usually `gpadmin:gpadmin`)
3. `hostfiles.yml`: Generates per-cluster hostfiles in the Unix user's home directory
4. `os.yml`: Updates the `/etc/hosts` files on all cluster systems
5. `ssh-keyexchange.yml`: Exchanges SSH keys across the cluster
6. `install.yml`: Installs WarehousePG and optional backup packages, calculates data directories
7. `initialize-database.yml`: Initializes the WarehousePG database

### warehouse-pg-cluster-data

Snapshots the state of every cluster declared as a child of `whpg_clusters`. Runs on each cluster's coordinator (`whpg_role: coordinator`); other hosts short-circuit out. Captures the database version, global cluster localization, connection / FTS / interconnect / network tuning / memory GUCs, OS-level memory tuning, segment topology, resource manager configuration (with the active groups or queues exported), non-default GUCs, tablespaces with per-segment locations, all databases, available and installed extensions, the coordinator's `pg_hba.conf` (comment lines stripped), and an inventory of the cluster hostfiles on disk.

Writes two files into the Unix user's home directory on each coordinator:

```
~/cluster_state_<cluster_name>.txt   # human-readable
~/cluster_state_<cluster_name>.json  # machine-readable
```

Invoked as the last play of `playbook.yml`, or on its own via `ansible-playbook cluster-data.yml`.

> **Note:** This role is **not idempotent**. It rewrites both `cluster_state_<cluster_name>.{txt,json}` files on every run; the `scanned_at` timestamp changes each time, and any drift in the cluster state will surface in the file contents. Expect every Ansible run to report this role as `changed`.

### warehouse-pg-wem-dashboard

Installs the WarehousePG Enterprise Manager (WEM) on the host defined by `wem_host` — a monitoring and administration dashboard for WarehousePG. The role installs the `whpg-enterprise-manager` RPM, edits the RPM-shipped `/etc/wem/wem.conf` in place (preserving the shipped commented examples and any RPM-generated secrets), and enables the `wem` systemd service. Default user/password for WEM is admin/admin and 
should be accesseible at http://<wem_host_ip>:8080/ prvoided filewall and security group settings are modifed
correctly.

See [roles/warehouse-pg-wem-dashboard/README.md](roles/warehouse-pg-wem-dashboard/README.md) for variables, prerequisites, and details.


### warehouse-pg-wem-state-database

Installs PostgreSQL 15 on the host defined by `wem_host` via the DNF AppStream module. After installation, the role creates the WEM state database and user (`wem_admin`), the Temporal user and databases (`temporal`, `temporal_visibility`) required by `warehouse-pg-wem-workflow-manager`, and updates `pg_hba.conf` to allow passwordless trust auth for local connections. This role must run before `warehouse-pg-wem-workflow-manager`.

See [roles/warehouse-pg-wem-state-database/README.md](roles/warehouse-pg-wem-state-database/README.md) for variables and details.

### warehouse-pg-wem-workflow-manager

Installs the Temporal.io workflow service on the host defined by `wem_host`. WEM uses Temporal for background job scheduling and durable workflow execution.

See [roles/warehouse-pg-wem-workflow-manager/README.md](roles/warehouse-pg-wem-workflow-manager/README.md) for variables and details.

### warehouse-pg-observability-prometheus

Downloads and installs a Prometheus binary from GitHub releases. Configures scrape jobs for Prometheus self-monitoring and WEM metrics, with remote-write-receiver enabled.

See [roles/warehouse-pg-observability-prometheus/README.md](roles/warehouse-pg-observability-prometheus/README.md) for variables and details.

### warehouse-pg-observability-loki

Downloads and installs a Loki binary from GitHub releases. Runs in single-binary filesystem mode for log aggregation.

See [roles/warehouse-pg-observability-loki/README.md](roles/warehouse-pg-observability-loki/README.md) for variables and details.

### warehouse-pg-observability-alertmanager

Downloads and installs an Alertmanager binary from GitHub releases with a minimal default-receiver configuration.

See [roles/warehouse-pg-observability-alertmanager/README.md](roles/warehouse-pg-observability-alertmanager/README.md) for variables and details.

### warehouse-pg-observability-collector

Installs the `edb-whpg-observability-collector` package on all cluster nodes and enables Grafana Alloy. On the coordinator only, configures the collector and runs the `deploy-observability` script.

See [roles/warehouse-pg-observability-collector/README.md](roles/warehouse-pg-observability-collector/README.md) for variables and details.

### acp-host-agent

Installs the `acp-host-agent` RPM on coordinator nodes only. Connects to `warehouse-pg-wem-workflow-manager` (Temporal) and registers `whpg.cluster.*` activity handlers for WEM remote cluster management.

See [roles/acp-host-agent/README.md](roles/acp-host-agent/README.md) for variables and details.

---


## Deployment Configuration (group_vars)

Ansible automatically loads variables from `group_vars/` based on group membership. This is the primary place to configure shared settings across roles without repeating them in every inventory file. Role `defaults/` serve as fallback values — `group_vars` overrides them for actual deployments.

```
group_vars/
  whpg_clusters/
    vars.yml           # variables for cluster node specific deployment (observability collector, ACP host agent)
  wem_stack/
    vars.yml           # variables for WEM stack hosts (PostgreSQL, Temporal, WEM dashboard)
  monitoring_stack/    # [optional] create to override Prometheus/Loki/Alertmanager settings
    vars.yml
```

> **Note on dynamic groups:** `wem_stack` and `monitoring_stack` are built at runtime by the `add_host` pre-play. Ansible still loads `group_vars/wem_stack/` and `group_vars/monitoring_stack/` correctly for these dynamic groups because `group_vars` are resolved relative to the playbook directory at play execution time — by which point the host is already in the group.

### All hosts (`group_vars/all/vars.yml`)

Variables here apply to every host regardless of group — `whpg_clusters`, `wem_stack`, and `monitoring_stack`. Use this for variables that are shared between the cluster plays (which create the DB role) and the WEM plays (which connect to it).

### WEM stack hosts (`group_vars/wem_stack/vars.yml`)

Variables here apply only to hosts in the `wem_stack` group. The `wem_stack` group is built dynamically by the pre-play from the `wem_host` variable — set per-cluster in inventory (see [Inventory Examples](#inventory-examples)) or overridden at runtime with `-e wem_host=<host>`. This covers all three WEM components: warehouse-pg-wem-state-database, warehouse-pg-wem-workflow-manager, and WEM dashboard.

To use external observability services instead of the local defaults, override `loki_host`, `prometheus_host`, or `alertmanager_host` here or via `-e`.

| Variable | Default | Description |
|---|---|---|
| `pg_port` | `5433` | warehouse-pg-wem-state-database port for WEM state database |
| `whpg_user` | `wem_user` | WarehousePG database user created for WEM |
| `whpg_password` | _(set it using group_vars/wem_stack/vars.yml)_ | Password for `whpg_user` |
| `whpg_port` | `5432` | WarehousePG coordinator port. |
| `wem_state_db` | `wem` | WEM state database name |
| `wem_state_db_host` | `127.0.0.1` | WEM state database host |
| `wem_state_db_port` | `{{ pg_port }}` | WEM state database port |
| `wem_state_db_user` | `wem_admin` | WEM state database user |
| `wem_state_db_password` | _(empty)_ | WEM state database password. Leave empty to use trust auth (pg_hba.conf allows local connections without password). |
| `wem_port` | `8080` | Port the WEM web interface listens on. |
| `wem_admin_password` | `admin` | Initial admin password. User need to reset on first login. |
| `temporal_db_host` | `127.0.0.1` | warehouse-pg-wem-workflow-manager PostgreSQL host. Use explicit IPv4 to avoid `localhost` resolving to `::1`. |
| `temporal_db_name` | `temporal` | warehouse-pg-wem-workflow-manager main database name |
| `temporal_visibility_db_name` | `temporal_visibility` | Temporal visibility database name |
| `temporal_db_user` | `temporal` | warehouse-pg-wem-workflow-manager database user |
| `temporal_db_password` | _(empty)_ | warehouse-pg-wem-workflow-manager database password. Leave empty to use trust auth (pg_hba.conf allows local connections without password). |
| `temporal_db_port` | `{{ pg_port }}` | warehouse-pg-wem-workflow-manager database port |
| `temporal_address` | `localhost:7233` | warehouse-pg-wem-workflow-manager frontend address WEM connects to |
| `wem_prometheus_url` | `http://localhost:9090` | Prometheus URL for WEM dashboard |
| `wem_loki_url` | `http://localhost:3100` | Loki URL for WEM dashboard |
| `wem_alertmanager_url` | `http://localhost:9093` | Alertmanager URL for WEM dashboard |

To override observability endpoints (e.g. external Prometheus/Loki), set `loki_host`, `prometheus_host`, `alertmanager_host` in `group_vars/wem_stack/vars.yml` or pass via `-e`.

<details>
<summary>Full example: <code>group_vars/wem_stack/vars.yml</code></summary>

```yaml
---

# =============================================================================
# Shared configuration for the wem_stack group
# Applies to all hosts in wem_stack (wem-state-database, wem-workflow-manager, wem-dashboard).
# Role defaults serve as fallback — these values override them for deployments.
# =============================================================================

# -----------------------------------------------------------------------------
# PostgreSQL state database settings (warehouse-pg-wem-state-database)
# -----------------------------------------------------------------------------
pg_port: 5433
pg_db: "postgres"

# -----------------------------------------------------------------------------
# Temporal databases and user (warehouse-pg-wem-workflow-manager)
# -----------------------------------------------------------------------------
temporal_db_host: "127.0.0.1"
temporal_db_port: "{{ pg_port }}"
temporal_db_user: "temporal"
temporal_db_name: "temporal"
temporal_visibility_db_name: "temporal_visibility"
temporal_default_namespace: "wem"
# Leave empty to use trust auth (pg_hba.conf must allow it and postgres is installed locally)
# temporal_db_password: ""

# -----------------------------------------------------------------------------
# WEM dashboard (warehouse-pg-wem-dashboard)
# -----------------------------------------------------------------------------
# Port the WEM web interface listens on. Leave it commented to use the default 8080.
# wem_port: 8080
# Initial password for the WEM web interface for user 'admin'. Leave it commented to
# have the role auto-generate a password on first run.
wem_admin_password: "admin"
#
# -------------------------------------------------------------
# WHPG Database connection settings for WEM dashboard.
# -------------------------------------------------------------
# If coordinator is in inventory, its IP is used automatically as whpg_host. 
# If not, whpg_host from here will be usedor -e whpg_host= is used.
whpg_host: "localhost"

# WHPG user for WEM dashboard to connect to the WarehousePG cluster.
# This user must exist in the WarehousePG cluster and have super user privileges.
whpg_user: "wem_user"

# WHPG password for WEM dashboard to connect to the WarehousePG cluster.
# Must be set by the operator — no built-in default.
whpg_password: ""

# WHPG port for WEM dashboard to connect to the WarehousePG cluster.
whpg_port: 5432

# WHPG database for WEM dashboard to connect to the WarehousePG cluster.
# The database should be accesible for whpg_user metioned above.
# If not provided/commented, the WEM dashboard will attempt to connect to the 
# the default postgres database on the WarehousePG cluster and create wem database.
# whpg_database: "wem"

# Optional existing separate monitoring database for WEM to monitor
# the WarehousePG cluster. If not provided/commented, the WEM dashboard 
# will reuse whpg_database for monitoring.
# whpg_monitoring_db: "wem_monitoring"

# -------------------------------------------------------------
# WEM state database connection settings for WEM dashboard
# -------------------------------------------------------------
# Postgresql database connection settings for WEM dashboard to connect to the WEM state database.
wem_state_db_host: "127.0.0.1"
wem_state_db_port: "{{ pg_port }}"
wem_state_db_user: "wem_admin"
# Leave empty to use trust auth (pg_hba.conf must allow it and postgres is installed locally)
# wem_state_db_password: ""
# Optional existing WEM state database for WEM dashboard to connect to.
# If not provided/commented, the WEM dashboard will attempt to connect to the
# the default postgres database on the WarehousePG cluster and create wem database.
# wem_state_db: "wem"

# -------------------------------------------------------------
# Workflow manager settings for WEM dashboard
# -------------------------------------------------------------
temporal_address: "localhost:7233"

# -------------------------------------------------------------
# Storage and monitoring settings for WEM dashboard
# -------------------------------------------------------------
alertmanager_host: "localhost"
alertmanager_port: 9093
loki_host: "localhost"
loki_port: 3100
prometheus_host: "localhost"
prometheus_port: 9090

wem_alertmanager_url: "http://{{ alertmanager_host }}:{{ alertmanager_port }}"
wem_loki_url: "http://{{ loki_host }}:{{ loki_port }}"
wem_prometheus_url: "http://{{ prometheus_host }}:{{ prometheus_port }}"
```

</details>

### Cluster nodes (`group_vars/whpg_clusters/vars.yml`)

Variables here apply to all hosts in the `whpg_clusters` group — coordinator and segment nodes. This covers the observability collector and ACP host agent, which are deployed on every cluster node.

The playbook derives `loki_host`, `prometheus_host`, and `temporal_host` from `monitoring_host` and `wem_host` set in the inventory `cluster_test/vars` section. Override for a single run with `-e monitoring_host=<host>` or `-e wem_host=<host>`. If neither is set, the defaults below apply (`localhost`).

| Variable | Default | Description |
|---|---|---|
| `loki_host` | `localhost` | Host where Loki is running |
| `loki_port` | `3100` | Loki port |
| `loki_endpoint` | `http://{{ loki_host }}:{{ loki_port }}/loki/api/v1/push` | Full Loki push endpoint for the collector |
| `prometheus_host` | `localhost` | Host where Prometheus is running |
| `prometheus_port` | `9090` | Prometheus port |
| `prometheus_endpoint` | `http://{{ prometheus_host }}:{{ prometheus_port }}/api/v1/write` | Full Prometheus remote-write endpoint for the collector |
| `temporal_host` | `localhost` | Host where Temporal is running |
| `temporal_frontend_grpc_port` | `7233` | Temporal frontend gRPC port |
| `temporal_address` | `{{ temporal_host }}:{{ temporal_frontend_grpc_port }}` | Temporal address for the ACP host agent |

To point the collector at an external monitoring stack, override `loki_host` and `prometheus_host` in `group_vars/whpg_clusters/vars.yml` or pass via `-e`.

<details>
<summary>Full example: <code>group_vars/whpg_clusters/vars.yml</code></summary>

```yaml
---

# =============================================================================
# Shared configuration for node-level agents (collector + acp-host-agent)
# Applies to all hosts in whpg_clusters.
# =============================================================================

greenplum_path: "/usr/local/greenplum-db/greenplum_path.sh"

# -----------------------------------------------------------------------------
# Observability collector (warehouse-pg-observability-collector)
# -----------------------------------------------------------------------------
# Loki and Prometheus endpoint to push logs and metrics.
loki_host: "localhost"
loki_port: 3100
prometheus_host: "localhost"
prometheus_port: 9090

loki_endpoint: "http://{{ loki_host }}:{{ loki_port }}/loki/api/v1/push"
prometheus_endpoint: "http://{{ prometheus_host }}:{{ prometheus_port }}/api/v1/write"

# -----------------------------------------------------------------------------
# ACP host agent (acp-host-agent)
# -----------------------------------------------------------------------------
# IP of the host running Temporal. Derived from wem_host if passed via -e;
# falls back to 127.0.0.1. Override with -e acp_temporal_host=<ip> to point
# at a Temporal instance running elsewhere.
temporal_host : "localhost"
temporal_frontend_grpc_port: 7233
temporal_address: "{{ temporal_host }}:{{ temporal_frontend_grpc_port }}"

# Data directory for the WarehousePG cluster.
WHPG_DATA_DIR: " {{ whpg_data_dir | default('/whpgdata') }}"
```

</details>

---

## Inventory Examples

Five inventory files are generated automatically by Terraform (`Testing/main.tf`). Choose the one that matches your deployment topology.

| File | Nodes | `wem_host` / `monitoring_host` | What installs automatically (no `--tags`) |
|---|---|---|---|
| `inventory.yml` | coordinator + segments | not set | WarehousePG cluster only |
| `inventory-with-standby.yml` | coordinator + standby + segments | not set | WarehousePG cluster only |
| `inventory-with-monitoring.yml` | coordinator + segments + monitoring_instance | `monitoring_instance` | WarehousePG cluster + Observability stack + WEM stack + Observability collector + ACP host agent |
| `inventory-with-standby-and-monitoring.yml` | coordinator + standby + segments + monitoring_instance | `monitoring_instance` | WarehousePG cluster (with standby) + Observability stack + WEM stack + Observability collector + ACP host agent |
| `inventory-with-standby-as-monitoring.yml` | coordinator + standby + segments | `standby` | WarehousePG cluster (with standby) + Observability stack + WEM stack on standby + Observability collector + ACP host agent |

`wem_host` and `monitoring_host` are set per-cluster in `cluster_test/vars` (not at `all/vars` level). They can be overridden per-run with `-e wem_host=<host>` or `-e monitoring_host=<host>`.

All clusters are defined as children of `whpg_clusters`. Each host must have:
- `whpg_role`: one of `coordinator`, `standby`, or `segment`
- `ip_address`: private IP used to derive observability/WEM endpoint URLs at play-level

`wem_stack` and `monitoring_stack` groups are built dynamically by the playbook pre-play from `wem_host` and `monitoring_host`. No need to define them in inventory. Inventories with a **dedicated external monitoring host** (one that is not already in `whpg_clusters`) define a static `whpg_clusters_monitoring` group so that `hostvars` is populated for that host. When the standby doubles as the monitoring host it is already in `whpg_clusters` with full connection details — no extra group is needed.

### inventory.yml — cluster only

No `wem_host` or `monitoring_host` set. Pass them via `-e` to deploy WEM or observability.

<details>
<summary>Show example</summary>

```yaml
all:
  vars:
    ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -J <jumphost>"
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            ansible_python_interpreter: /usr/bin/python3
            whpg_host: <coordinator-private-ip>
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            seg1:
              ansible_host: <seg1-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
```

</details>

### inventory-with-standby.yml — cluster with standby

No `wem_host` or `monitoring_host` set. Pass them via `-e` to deploy WEM or observability.

<details>
<summary>Show example</summary>

```yaml
all:
  vars:
    ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -J <jumphost>"
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            ansible_python_interpreter: /usr/bin/python3
            whpg_host: <coordinator-private-ip>
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
```

</details>

### inventory-with-monitoring.yml — dedicated monitoring node

`wem_host` and `monitoring_host` are set to `monitoring_instance` in `cluster_test/vars`. A static `monitoring_hosts` group defines the host so `hostvars` is populated.

<details>
<summary>Show example</summary>

```yaml
all:
  vars:
    ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -J <jumphost>"
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            ansible_python_interpreter: /usr/bin/python3
            whpg_host: <coordinator-private-ip>
            wem_host: monitoring_instance
            monitoring_host: monitoring_instance
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            seg1:
              ansible_host: <seg1-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
    monitoring_hosts:
      hosts:
        monitoring_instance:
          ansible_host: <monitoring-public-ip>
          ansible_user: rocky
          ansible_ssh_private_key_file: /path/to/key.pem
          ip_address: <monitoring-private-ip>
          hostname: monitoring
```

</details>

### inventory-with-standby-and-monitoring.yml — standby + dedicated monitoring node

<details>
<summary>Show example</summary>

```yaml
all:
  vars:
    ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -J <jumphost>"
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            ansible_python_interpreter: /usr/bin/python3
            whpg_host: <coordinator-private-ip>
            wem_host: monitoring_instance
            monitoring_host: monitoring_instance
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
    monitoring_hosts:
      hosts:
        monitoring_instance:
          ansible_host: <monitoring-public-ip>
          ansible_user: rocky
          ansible_ssh_private_key_file: /path/to/key.pem
          ip_address: <monitoring-private-ip>
          hostname: monitoring
```

</details>

### inventory-with-standby-as-monitoring.yml — standby doubles as WEM/monitoring host

The standby is already in `whpg_clusters` with full connection details. `wem_host` and `monitoring_host` point to `standby` in the cluster group vars — no separate monitoring group is needed.

<details>
<summary>Show example</summary>

```yaml
all:
  vars:
    ansible_ssh_common_args: "-o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/dev/null -o ServerAliveInterval=30 -J <jumphost>"
  children:
    whpg_clusters:
      children:
        cluster_test:
          vars:
            cluster_name: test
            whpg_cluster_name: test
            whpg_data_directory: /data/whpg
            ansible_python_interpreter: /usr/bin/python3
            whpg_host: <coordinator-private-ip>
            wem_host: standby
            monitoring_host: standby
          hosts:
            coordinator:
              ansible_host: <coordinator-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: coordinator
              ip_address: <coordinator-private-ip>
              hostname: coordinator
            standby:
              ansible_host: <standby-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: standby
              ip_address: <standby-private-ip>
              hostname: standby
            seg1:
              ansible_host: <seg1-public-ip>
              ansible_user: rocky
              ansible_ssh_private_key_file: /path/to/key.pem
              whpg_role: segment
              ip_address: <seg1-private-ip>
              hostname: seg1
```

</details>

---

## Usage

Create an Ansible inventory which defines the roles for each server. See the `Inventory Example` section earlier in this document.

### Playbook tags

The playbook uses tags to allow selective deployment of different parts of the stack:

| Tag | Hosts | Requires | What runs |
|---|---|---|---|
| `deploy` | `whpg_clusters` | Any inventory | Full cluster deployment: OS prep, packages, cluster init, pg_hba, standby, resource management |
| `cluster-data` | `whpg_clusters` | Any inventory | Snapshot cluster state to `cluster_state_<name>.{txt,json}` |
| `update-pg-hba` | `wem_stack` (delegates to coordinator) | Inventory with `wem_host` | Update `pg_hba.conf` on coordinator without a full deploy. Discovers paths via `psql`. |
| `warehouse-pg-observability` | `monitoring_stack` (built from `monitoring_host`) | Inventory with `monitoring_host` | Deploy Prometheus, Loki, and Alertmanager |
| `wem` | `wem_stack` (built from `wem_host`) | Inventory with `wem_host` | Deploy all WEM components: PostgreSQL, Temporal, and WEM dashboard |
| `warehouse-pg-wem-state-database` | `wem_stack` (built from `wem_host`) | Inventory with `wem_host` | Install PostgreSQL state database only |
| `warehouse-pg-wem-workflow-manager` | `wem_stack` (built from `wem_host`) | Inventory with `wem_host` | Install Temporal.io workflow service only |
| `warehouse-pg-wem-dashboard` | `wem_stack` (built from `wem_host`) | Inventory with `wem_host` | Install WEM dashboard only |
| `warehouse-pg-observability-collector` | `whpg_clusters` | Inventory with `monitoring_host` | Install observability collector and Grafana Alloy on all cluster nodes |
| `acp-host-agent` | `whpg_clusters` (coordinator only) | Inventory with `wem_host` | Install ACP host agent on coordinator |

### Examples

#### Full stack deployment (no `--tags`)

Run without tags — what deploys depends entirely on the inventory file. WEM and observability are deployed automatically when `wem_host`/`monitoring_host` are set in the inventory.

> **Note:** This also runs the `cluster-data` play on every execution, which is not idempotent (rewrites snapshot files each time). For repeated runs, prefer the explicit-tags approach in [Section 4](#section-4-all-at-once).

| Inventory | Command | What deploys |
|---|---|---|
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml` | WarehousePG cluster (coordinator + segments) |
| `inventory-with-standby.yml` | `ansible-playbook -i inventory-with-standby.yml playbook.yml` | WarehousePG cluster (coordinator + standby + segments) |
| `inventory-with-monitoring.yml` | `ansible-playbook -i inventory-with-monitoring.yml playbook.yml` | WarehousePG cluster + Observability stack, WEM stack (on monitoring_instance) + Collector + ACP host agent on all cluster nodes |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml` | WarehousePG cluster (with standby) + Observability stack, WEM stack (on monitoring_instance) + Collector + ACP host agent on all cluster nodes |
| `inventory-with-standby-as-monitoring.yml` | `ansible-playbook -i inventory-with-standby-as-monitoring.yml playbook.yml` | WarehousePG cluster (with standby) + Observability stack, WEM (on standby) + Collector + ACP host agent on all cluster nodes |

---

#### Selective deployment (`--tags`)

##### Section 1: Cluster deployment

| Inventory | Command | What deploys |
|---|---|---|
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml --tags deploy` | WarehousePG cluster |
| `inventory.yml` | `ansible-playbook -i inventory.yml playbook.yml --tags cluster-data` | Snapshot cluster state only |
| `inventory-with-standby.yml` | `ansible-playbook -i inventory-with-standby.yml playbook.yml --tags deploy` | WarehousePG cluster |

All tags above work with any inventory that has `whpg_clusters` hosts defined — replace `-i` accordingly.

---

##### Section 2: WEM + Observability stack

Use an inventory that has `wem_host` and/or `monitoring_host` set in `cluster_test/vars`.

| Inventory | Command | What deploys |
|---|---|---|
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags "wem,warehouse-pg-observability"` | WEM stack + observability stack |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags wem` | PostgreSQL + Temporal + WEM dashboard |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.ymlplaybook.yml --tags warehouse-pg-observability` | Prometheus + Loki + Alertmanager |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.ymlplaybook.yml --tags warehouse-pg-wem-state-database` | PostgreSQL state DB only |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.ymlplaybook.yml --tags warehouse-pg-wem-workflow-manager` | Temporal only |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.ymlplaybook.yml --tags warehouse-pg-wem-dashboard` | WEM dashboard only |

The tags in the table above work with any inventory that has `wem_host`/`monitoring_host` set — replace `-i` accordingly.

---

##### Section 3: Observability collector and ACP host agent

These run on `whpg_clusters` hosts. The collector is only deployed when `monitoring_stack` group is non-empty; the ACP agent only when `wem_stack` is non-empty. Both conditions are satisfied automatically when `monitoring_host`/`wem_host` are set in inventory.

| Inventory | Command | What deploys |
|---|---|---|
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags "warehouse-pg-observability-collector,acp-host-agent"` | Collector + ACP host agent on all cluster nodes |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-observability-collector` | Observability collector only |
| `inventory-with-standby-and-monitoring.yml` | `ansible-playbook -i inventory-with-standby-and-monitoring.yml  playbook.yml --tags acp-host-agent` | ACP host agent only on coordinator |

---

#### Upgrade

**RPM-based components** (WEM, collector, ACP host agent, WarehousePG): set the upgrade variable to `always-upgrade-to-latest-minor` and re-run the relevant tag.

| Component | Command |
|---|---|
| WarehousePG cluster | `ansible-playbook -i <inventory> playbook.yml -e whpg_upgrade=always-upgrade-to-latest-minor --tags deploy` |
| WEM dashboard | `ansible-playbook -i <inventory> playbook.yml -e wem_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-wem-dashboard` |
| Observability collector | `ansible-playbook -i <inventory> playbook.yml -e collector_upgrade=always-upgrade-to-latest-minor --tags warehouse-pg-observability-collector` |
| ACP host agent | `ansible-playbook -i <inventory> playbook.yml -e acp_agent_upgrade=always-upgrade-to-latest-minor --tags acp-host-agent` |

**Binary-based components** (Prometheus, Loki, Alertmanager, Temporal): set the version variable and re-run the relevant tag — the role reinstalls automatically when the version differs from what is installed.

| Component | Variable | Command |
|---|---|---|
| Prometheus | `prometheus_version` | `ansible-playbook -i <inventory> playbook.yml -e prometheus_version=3.6.0 --tags warehouse-pg-observability` |
| Loki | `loki_version` | `ansible-playbook -i <inventory> playbook.yml -e loki_version=3.6.0 --tags warehouse-pg-observability` |
| Alertmanager | `alertmanager_version` | `ansible-playbook -i <inventory> playbook.yml -e alertmanager_version=0.34.0 --tags warehouse-pg-observability` |
| Temporal | `temporal_version` | `ansible-playbook -i <inventory> playbook.yml -e temporal_version=1.32.0 --tags warehouse-pg-wem-workflow-manager` |

To pin a specific RPM version for RPM-based components:

| Component | Variable | Example |
|---|---|---|
| WEM dashboard | `wem_version` | `-e wem_version=1.2.3` |
| Observability collector | `obs_collector_version` | `-e obs_collector_version=1.2.3` |
| ACP host agent | `acp_agent_version` | `-e acp_agent_version=1.2.3` |

---

### Infrastructure provisioning (Terraform)

The `Testing/` directory contains OpenTofu/Terraform configuration that provisions the underlying infrastructure on AWS:

- **7 EC2 instances** (`m7a.large`, Rocky Linux 9) by default

Terraform generates all five inventory files automatically on `apply`. To override the instance count, type, or jumphost username without editing `main.tf`, create `Testing/terraform.tfvars` and/or `Testing/override.tf`:

```hcl
# Testing/terraform.tfvars — variable overrides
edb_jumphost_username = "your.name"
```

```hcl
# Testing/override.tf — resource attribute overrides
resource "aws_instance" "ec2_nodes" {
  count         = 3
  instance_type = "t3a.small"
}
```

```bash
cd Testing
aws sso login --profile whpg
tofu apply -auto-approve

cd ../WarehousePG
ansible-playbook -i inventory-with-monitoring.yml playbook.yml --tags "deploy,wem,warehouse-pg-observability,warehouse-pg-observability-collector,acp-host-agent"
```

---

## Generated files

The role will generate a number of files In the home directory of `WHPG_UNIX_USER`:

```
~/cluster_hostfile_all_<cluster_name>            # contains all hostnames
~/cluster_hostfile_coordinator_<cluster_name>.   # contains the hostname of the coordinator
~/cluster_hostfile_standby_<cluster_name>        # only if standby host is defined
~/cluster_hostfile_segments_<cluster_name>       # only if segment hosts are defined
~/gpinitsystem_<cluster_name>                    # the gpinitsystem file used to initialize the cluster
```

---

## Database users

The `warehouse-pg` role can create or update WarehousePG database roles on the coordinator via the `whpg_db_roles` variable. Each entry creates the role if it does not exist, or updates the password and privileges if it does. Runs on the coordinator only.

```yaml
whpg_db_roles:
  - name: reporting
    password: "r3port!"
  - name: etl_admin
    password: "etl123"
    createdb: true
  - name: wem_user
    password: "admin"
    superuser: true
```

| Key | Required | Default | Description |
|---|---|---|---|
| `name` | yes | — | Role name. |
| `password` | no | — | Login password. Omit for passwordless roles. |
| `superuser` | no | `false` | Grant `SUPERUSER` privilege. |
| `createdb` | no | `false` | Grant `CREATEDB` privilege. |
| `createrole` | no | `false` | Grant `CREATEROLE` privilege. |

Run standalone (skips cluster deploy, SSH key exchange, etc.):

```bash
ansible-playbook playbook.yml --tags roles
```

---

## Customizing `pg_hba.conf`

After cluster initialization, the role can extend the coordinator's `pg_hba.conf` with additional entries. Two mechanisms are available, and both can be used together. If either changes `pg_hba.conf`, the database is reloaded automatically.

### Template file per cluster

Drop a file named `pg_hba_<cluster_name>.conf` into the `warehouse-pg` role's `templates/` directory. When present, its contents are appended to the coordinator's `pg_hba.conf` inside an Ansible-managed block. The file is processed through Jinja2, so inventory variables can be referenced.

### `additional_pg_hba_conf` variable

Define `additional_pg_hba_conf` in the inventory as a list of entries. Each entry produces one `pg_hba.conf` line and the resulting block is added to the coordinator's `pg_hba.conf`.

Each entry is a hash with the following keys:

| Key | Required | Description |
|---|---|---|
| `conntype` | Yes | One of `local`, `host`, `hostssl`, `hostnossl`, `hostgssenc`, `hostnogssenc`. |
| `database` | Yes | Database name (e.g. `all`, `postgres`, `mydb`). |
| `user` | Yes | User or role name (e.g. `all`, `gpadmin`, `+admins`). |
| `address` | No | IPv4 or IPv6 address with hostmask (e.g. `10.0.0.0/8`, `2001:db8::/32`). Omit for `conntype: local`. |
| `method` | Yes | Authentication method (e.g. `trust`, `md5`, `scram-sha-256`, `peer`). |
| `options` | No | Free-form trailing options string. |

Example:

```yaml
additional_pg_hba_conf:
  - conntype: local
    database: all
    user: gpadmin
    method: peer
  - conntype: host
    database: all
    user: all
    address: 10.0.0.0/8
    method: scram-sha-256
  - conntype: hostssl
    database: mydb
    user: app_user
    address: 2001:db8::/32
    method: cert
    options: clientcert=verify-full
```

---

## Standby coordinator

The standby coordinator's lifecycle is driven by the inventory. Any host inside a cluster group that has `whpg_role: standby` is treated as the desired standby (at most one per cluster). On each run, the role reconciles inventory against the live cluster.

### Reconciliation

1. `03-hostfiles.yml` writes `~/cluster_hostfile_standby_<cluster_name>` on the coordinator when a standby host is in inventory, and deletes the file when no standby host is in inventory.
2. `07-initialize-database.yml`, on the coordinator, reads that hostfile, runs `gpstate -f`, and parses the `Standby address = …` line to find the currently configured standby.
3. The desired state (from the hostfile) and configured state (from `gpstate`) are compared:

| Inventory standby | `gpstate -f` standby | Action |
|---|---|---|
| none | none | no-op |
| `hostX` | none | `gpinitstandby -s hostX -S <whpg_standby_data_directory>/<seg_prefix>-1 -a` |
| none | `hostX` | `gpinitstandby -r -a` (remove) |
| `hostX` | `hostX` | no-op |
| `hostX` | `hostY` | fail — changing the standby hostname in place is not allowed |

### Data directory layout

The standby's data directory lives under `whpg_standby_data_directory` (default `<whpg_data_directory>/v<major_version>/standby`), with the standby data placed at `<whpg_standby_data_directory>/<seg_prefix>-1` (e.g. `/data/whpg/v7/standby/gpseg-1`). `gpinitstandby` is invoked with `-S` pointing at that path, so the standby host does **not** need the coordinator's directory path (e.g. `/data/whpg/v7/coordinator`) to exist. The parent (`whpg_standby_data_directory`) is created on the standby host by `06-install.yml`.

### Swapping the standby

To move the standby to a different host, change the inventory in two steps:

1. Remove `whpg_role: standby` from the current standby host and run the playbook. The old standby is removed via `gpinitstandby -r`.
2. Set `whpg_role: standby` on the new host and run the playbook again. The new standby is created via `gpinitstandby -s`.

Attempting both in a single inventory change (changing which host carries `whpg_role: standby` while the old standby is still configured in the database) is rejected with an explicit failure, since `gpinitstandby` does not support an in-place hostname change.

## Workload management

WarehousePG supports two mutually exclusive workload managers, selected via the variable `whpg_resource_management`. Valid values:

- `queue` — Resource Queues. Enforces concurrency, memory, and priority inside the database. No OS-level prep required.
- `group` — Resource Groups, with **auto-detection** of cgroup v1 vs v2 based on what is mounted on the host.
- `group-v2` — Resource Groups, **force cgroup v2**. Pre-flight rejects this on hosts that do not have `/sys/fs/cgroup/cgroup.controllers` (i.e. that aren't running cgroup v2 unified).

The value maps to the `gp_resource_manager` GUC. If the database currently runs under a different manager than the inventory specifies, the playbook switches it via `gpconfig` and restarts the cluster with `gpstop -r` before applying any group/queue definitions.

For Resource Groups, Greenplum 7 distinguishes between cgroup v1 (`gp_resource_manager=group`, paths under `/sys/fs/cgroup/cpu/gpdb/...`) and cgroup v2 (`gp_resource_manager=group-v2`, paths under `/sys/fs/cgroup/gpdb/` and `/sys/fs/cgroup/gpdb.service/`). With `whpg_resource_management: group`, the playbook detects which hierarchy is mounted and chooses the right GUC value for you; with `whpg_resource_management: group-v2`, it always uses `group-v2`. The OS prep in `04-os.yml` installs `libcgroup-tools` and writes `/etc/cgconfig.d/gpdb.conf` on cgroup v1 hosts, or installs a `gpdb-cgroup.service` systemd unit that creates both `/sys/fs/cgroup/gpdb` and `/sys/fs/cgroup/gpdb.service` on cgroup v2 hosts.

The pre-flight check rejects any value other than `queue`, `group`, or `group-v2`. When `group-v2` is set, pre-flight also verifies that every host has `/sys/fs/cgroup/cgroup.controllers`.

### `whpg_resource_groups`

A list of resource group definitions, applied only when `whpg_resource_management` is `group` or `group-v2`. Each entry is a hash with a unique `name` and the following attributes:

| Attribute | Type | Range / values | Notes |
|---|---|---|---|
| `CONCURRENCY` | int | 0--1000 | maximum concurrent transactions |
| `CPU_MAX_PERCENT` | int | -1, or 1--100 | mutually exclusive with `CPUSET` |
| `CPU_WEIGHT` | int | 1--500 | relative CPU priority |
| `CPUSET` | int | -1, or 0--256 | mutually exclusive with `CPU_MAX_PERCENT` |
| `IO_LIMIT` | int / str | -1, 2--4294967295, or `max` | I/O bytes per second, or `max` for unbounded |
| `MEMORY_QUOTA` | int | -1, or 0..INT_MAX (MB) | -1 = queries use `statement_mem` |
| `MIN_COST` | int | >= 0 | minimum query cost handled by the group |

If `CPUSET >= 0`, the playbook emits `CPUSET='<n>'` in the CREATE/ALTER statement (and Greenplum forces `CPU_MAX_PERCENT` to -1). If `CPUSET == -1`, the playbook emits `CPU_MAX_PERCENT=<n>` instead.

The built-in groups `default_group`, `admin_group`, and `system_group` are never removed, even if absent from inventory.

Example:

```yaml
whpg_resource_groups:
  - name: rgroup_etl              # balanced ETL workload
    CONCURRENCY: 20
    CPU_MAX_PERCENT: 20
    CPU_WEIGHT: 500
    CPUSET: -1
    IO_LIMIT: -1
    MEMORY_QUOTA: 250
    MIN_COST: 50
  - name: rgroup_reporting        # many concurrent, low-cost report queries
    CONCURRENCY: 50
    CPU_MAX_PERCENT: 30
    CPU_WEIGHT: 200
    CPUSET: -1
    IO_LIMIT: -1
    MEMORY_QUOTA: 100
    MIN_COST: 0
  - name: rgroup_admin            # dedicated CPU core for admin/maintenance
    CONCURRENCY: 5
    CPU_MAX_PERCENT: -1
    CPU_WEIGHT: 500
    CPUSET: 0
    IO_LIMIT: max
    MEMORY_QUOTA: 500
    MIN_COST: 100
```

### `whpg_resource_queues`

A list of resource queue definitions, applied only when `whpg_resource_management: queue`. Each entry is a hash with a unique `name` and:

| Attribute | Type | Values |
|---|---|---|
| `MEMORY_LIMIT` | string | PostgreSQL memory string (`'2GB'`, `'512MB'`, ...) |
| `ACTIVE_STATEMENTS` | int | > 0 |
| `PRIORITY` | string | `LOW`, `MEDIUM`, `HIGH`, `MAX` |
| `MAX_COST` | number | >= 0 |

The built-in `pg_default` queue is never removed, even if absent from inventory.

Example:

```yaml
whpg_resource_queues:
  - name: etl                     # heavy ETL: low concurrency, large memory
    MEMORY_LIMIT: '2GB'
    ACTIVE_STATEMENTS: 3
    PRIORITY: MEDIUM
    MAX_COST: 0
  - name: reporting               # many concurrent, prioritized reports
    MEMORY_LIMIT: '4GB'
    ACTIVE_STATEMENTS: 10
    PRIORITY: HIGH
    MAX_COST: 0
```

### Sync behavior

At the end of `07-initialize-database.yml`, on the coordinator host:

1. Query the current `gp_resource_manager`. If it differs from `whpg_resource_management`, run `gpconfig` and `gpstop -r` to switch.
2. Fetch the current resource groups (or queues) and their settings from the database.
3. `CREATE` any entry that appears in inventory but not in the database.
4. `ALTER` an existing entry only when at least one attribute differs from inventory — no statements are sent if everything already matches (keeps the audit log clean).
5. `DROP` any database entry not present in inventory, skipping the built-in defaults listed above.

Leaving the relevant list (`whpg_resource_groups` or `whpg_resource_queues`) at its empty default is supported. No entries will be created or updated, but the cleanup step will still drop any custom (non-default) entries that exist in the database.

See `roles/warehouse-pg/defaults/main.yml` for fully commented examples.

## Extensions

The `warehouse-pg` role can install optional WarehousePG extensions. Each extension has an associated `whpg_extension_<name>` variable that defaults to `false`. The variable accepts three kinds of values:

- `true` — install the latest package matching `$whpg_major_version`. The package state honours `whpg_upgrade` (i.e. it is upgraded to the latest minor version when `whpg_upgrade` is `always-upgrade-to-latest-minor`, otherwise only installed if missing).
- `false` — uninstall the extension package if it is currently installed.
- a string — install the exact RPM identified by the string. The string is passed verbatim to `dnf` with `state: present`, so it may be a bare package name, an `name-version-release` (NVR), a full `name-version-release.arch` (NEVRA), or a path to a local `.rpm` file. The requested version is installed, upgraded, or downgraded as needed.

The role automatically selects the package matching `$whpg_major_version` for the `true`/`false` cases; extensions that are not available for the configured major version are skipped.

| Extension | Variable |
|---|---|
| diskquota | `whpg_extension_diskquota` |
| fts-enhanced | `whpg_extension_fts_enhanced` |
| hll | `whpg_extension_hll` |
| madlib | `whpg_extension_madlib` |
| madlib1 | `whpg_extension_madlib1` |
| pgfs | `whpg_extension_pgfs` |
| pgvector | `whpg_extension_pgvector` |
| plr | `whpg_extension_plr` |
| postgis | `whpg_extension_postgis` |
| pxf | `whpg_extension_pxf` |
| q3c | `whpg_extension_q3c` |
| user_profile | `whpg_extension_user_profile` |
| vectorchord | `whpg_extension_vectorchord` |
| whpg_fdw | `whpg_extension_whpg_fdw` |

Examples:

```yaml
# Install the latest pgvector package for the configured major version
whpg_extension_pgvector: true

# Uninstall the diskquota package if currently installed
whpg_extension_diskquota: false

# Install a specific pgvector RPM by NVR
whpg_extension_pgvector: "edb-whpg7-pgvector-0.7.4-1.el8"

# Install a specific postgis RPM by full NEVRA
whpg_extension_postgis: "edb-whpg6-postgis-3.3.2-1.el8.x86_64"

# Install madlib from a local RPM file
whpg_extension_madlib: "/var/tmp/edb-whpg7-madlib-2.1.0-1.el8.x86_64.rpm"
```

When `whpg_install_packages` is `false`, the role does not install or remove any packages and instead verifies that every extension marked with either `true` or a string value is already installed on the host (the base package name is checked, not a specific version).

This step only installs the extension package. The extension must still be activated in the database, using the `CREATE EXTENSION` command.

## All Variables

This section lists every non-internal variable referenced by the roles. Variables whose names begin with an underscore (`_`) are role-internal and are intentionally omitted.

### Shared variables (used across all roles)

| Variable | Default | Description |
|---|---|---|

### Role: `edb-repository`

| Variable | Default | Description |
|---|---|---|
| `edb_token` | `""` | EDB repository authentication token. Must be defined and non-empty. |
| `edb_repository` | `"dev"` | EDB repository name (e.g. `dev`, `gsupp`). |
| `serial_operations` | `1` | Throttle for concurrent download/install operations. |

### Role: `warehouse-pg`

#### Inventory / cluster variables (no defaults, must be provided)

| Variable | Scope | Description |
|---|---|---|
| `cluster_name` | group | Cluster identifier used to build the inventory group name (`cluster_<cluster_name>`), hostfile names, and `gpinitsystem_config_<cluster_name>`. |
| `whpg_cluster_name` | group | Cluster name identifier. Non-empty string; validated in pre-flight checks. |
| `whpg_data_directory` | group/host | Base data directory path. Must exist as a directory on every host. |
| `whpg_role` | host | Role of each host within its cluster. One of `coordinator`, `standby`, `segment`. |
| `hostname` | host | Expected system hostname for the host. Compared against `ansible_facts['hostname']`; set via `hostname` module. |
| `ip_address` | host | IPv4/IPv6 address used to populate `/etc/hosts` entries for all cluster hosts. |

#### Variables with defaults

| Variable | Default | Description |
|---|---|---|
| `WHPG_UNIX_USER` | `"gpadmin"` | Unix user that owns the WarehousePG installation. |
| `WHPG_UNIX_GROUP` | `"gpadmin"` | Unix group for the WarehousePG user. |
| `whpg_major_version` | `7` | WarehousePG major version. Must be `6` or `7`. |
| `whpg_seg_prefix` | `"default"` | Segment prefix. Non-empty string. When equal to `"default"`, the template emits `gpseg`. |
| `whpg_seg_num` | `2` | Number of primary segments per host. Integer, 1--64. |
| `whpg_port` | `5432` | Coordinator listen port. Integer, 1024--65535. |
| `whpg_seg_base_port` | `6000` | Base port for primary segments (written to `PORT_BASE` in `gpinitsystem` config). |
| `whpg_upgrade` | `"install-but-no-automatic-upgrades"` | Upgrade policy for warehouse-pg packages. The default `"install-but-no-automatic-upgrades"` is the safe setting: it installs the packages if missing but will not upgrade an already-installed version (maps to the `dnf` state `present`). `"always-upgrade-to-latest-minor"` auto-upgrades to the latest available minor version on every run (maps to the `dnf` state `latest`). |
| `whpg_use_versioned_directories` | `true` | When `true`, data directories include a `v<major_version>/` subdirectory. |
| `whpg_seg_enable_mirrors` | `true` | Enable mirror segments in the `gpinitsystem` configuration. |
| `whpg_seg_mirror_base_port` | `7000` | Base port for mirror segments. |
| `whpg_default_database` | `""` | Default database name. Must be a valid PostgreSQL identifier (1--63 chars, starts with letter or underscore) when non-empty. |
| `whpg_install_packages` | `true` | Whether the role should install packages. When `false`, the user is expected to install them. |
| `whpg_install_backup` | `false` | Set to `true` to install the `whpg-backup` package. |
| `whpg_resource_management` | `"group"` | Workload manager: `queue`, `group`, or `group-v2`. Maps to the `gp_resource_manager` GUC. See the "Workload management" section. |
| `whpg_resource_groups` | `[]` | List of resource group definitions applied when `whpg_resource_management` is `group` or `group-v2`. Empty by default. |
| `whpg_resource_queues` | `[]` | List of resource queue definitions applied when `whpg_resource_management` is `queue`. Empty by default. |
| `DNF_TIMEOUT` | `120` | Maximum time (seconds) to wait for the RPM repository. |
| `whpg_remove_old_gpAdminLogs` | `false` | Add maintenance jobs to remove old `gpAdminLogs` logfiles on coordinator/standby. |
| `whpg_remove_old_logs` | `false` | Add maintenance job to remove old data-directory logfiles. |
| `greenplum_path` | `"/usr/edb/whpg7/greenplum_path.sh"` | Path to the `greenplum_path.sh` environment script. |

#### Optional variables (no default; validated only when defined)

| Variable | Description |
|---|---|
| `whpg_seg_base_port` | Base port for segments. Integer, 1024--65535. Validated only when defined and non-empty. |
| `additional_pg_hba_conf` | List of additional `pg_hba.conf` entries to apply on the coordinator. See the "Customizing `pg_hba.conf`" section above for the entry schema. |
| `whpg_db_roles` | List of database roles to create or update on the coordinator. See the "Database roles" section for the entry schema. Empty by default. |

### Role: `warehouse-pg-wem-dashboard`

See [roles/warehouse-pg-wem-dashboard/README.md](roles/warehouse-pg-wem-dashboard/README.md) for the full variable reference.

### Role: `warehouse-pg-wem-workflow-manager`

See [roles/warehouse-pg-wem-workflow-manager/README.md](roles/warehouse-pg-wem-workflow-manager/README.md) for the full variable reference.

### Role: `acp-host-agent`

See [roles/acp-host-agent/README.md](roles/acp-host-agent/README.md) for the full variable reference.

#### Running upgrades

The `whpg_upgrade` variable defaults to `install-but-no-automatic-upgrades`, which is a safe choice. The playbook can run again, without package changes applied and potentially forcing database restarts.

This variable can be overridden on the command line. Example:

```
ansible-playbook playbook.yml -e "whpg_upgrade=always-upgrade-to-latest-minor"
```

This will apply the new setting only for this run of the playbook, making it safe to apply potential updates when convenient.

## Performance

### AWS Instance Types

- Instance Type: `m7a.medium`: `1x vCPU`, `4 GB RAM`
- Instance Type: `m7a.large`: `2x vCPU`, `8 GB RAM`
- Instance Type: `m7a.xlarge`: `4x vCPU`, `16 GB RAM`
- Instance Type: `m7a.2xlarge`: `8x vCPU`, `32 GB RAM`

### Tests

Tests run with:

```
time ansible-playbook --extra-vars "whpg_seg_num=2" --extra-vars "whpg_seg_enable_mirrors=false" --extra-vars "whpg_resource_management=group-v2" playbook.yml
```

The `whpg_seg_num` parameter is adapted, see the output below.

The `forks` parameter is set to `10` (default: `10`), ssh options used:

```
ssh_args = -o ControlMaster=auto -o ControlPersist=10s -o StrictHostKeyChecking=no
```

All test did only run once. All tests ran on the code version 2026-05-21, deploying the EDB repository (role `edb-repository`), deploying the cluster (role `warehouse-pg`), creating cluster stats (role `warehouse-pg-cluster-data`), no expansions configured, no WEM (role `warehouse-pg-enterprise-manager`).

Note: runtine for tests are not linear, several factors can affect the runtime:

- network roundtrip time
- general ssh delays and retries
- repository delay (all EDB repository operations are serialized)
- load on the AWS hosts, influencing the virtual machine performance
- ssh jumphost delays (EDB is using a jumphost to access AWS infrastructure)

#### m7a.medium

| Segment Hosts | Segments | Run 1 | Run 2 |
|:---:|:---:|---:|---:|
| 2  | 2 |  8m0s |  4m56s |
| 2  | 4 | 8m3s |  5m 55s |
| 4  | 2 |  9m 58s |  5m 57s |
| 4  | 4 | 10m 29s |  5m 55s |
| 10 | 2 | 17m 51s | 10m 27s |
| 10 | 4 | 19m 26s | 10m 19s |
| 20 | 2 | 28m 53s | 15m  0s |
| 20 | 4 | 30m 44s | 15m  5s |

4 GB RAM is not enough for the `coordinator` to manage 40 segments, operation times out.

In general, `m7a.medium` instances are too small to run 2 WarehousePG instances. Deployment will occasionally time out.

#### m7a.large

| Segment Hosts | Segments | Run 1 | Run 2 |
|:---:|:---:|---:|---:|
|  2 | 2 |  7m 48s |  5m  0s |
|  2 | 4 |  7m 49s |  5m  2s |
|  4 | 2 |  9m 48s |  5m 48s |
|  4 | 4 |  9m 17s |  5m 56s |
| 10 | 2 | 16m 48s |  9m 30s |
| 10 | 4 | 17m 21s |  9m 27s |
| 20 | 2 | 31m 23s | 15m  8s |
| 20 | 4 | 30m 54s | 15m 34s |
| 40 | 2 | 56m 45s | 27m 31s |
| 40 | 4 | 50m 40s | 21m 13s |

#### m7a.xlarge

| Segment Hosts | Segments | Run 1 | Run 2 |
|:---:|:---:|---:|---:|
|  2 | 2 |  7m 12s |  4m 26s |
|  2 | 4 |  7m 28s |  4m 30s |
|  4 | 2 |  8m 10s |  4m 56s |
|  4 | 4 |  8m 50s |  4m 58s |
| 10 | 2 | 15m  3s |  7m 56s |
| 10 | 4 | 16m 35s |  7m 58s |
| 20 | 2 | 26m 40s | 12m 13s |
| 20 | 4 | 27m 36s | 13m  0s |
| 40 | 2 | 50m 52s | 24m 22s |
| 40 | 4 | 52m  8s | 21m 20s |

#### m7a.2xlarge

| Segment Hosts | Segments | Run 1 | Run 2 |
|:---:|:---:|---:|---:|
|  2 | 2 |  7m  3s |  4m 52s |
|  2 | 4 |  7m 48s |  4m 32s |
|  4 | 2 |  8m 32s |  5m  1s |
|  4 | 4 |  8m 39s |  4m 59s |
| 10 | 2 | 15m 17s |  8m 16s |
| 10 | 4 | 16m 27s |  8m  5s |
| 10 | 6 | 17m 38s |  8m  7s |
| 10 | 8 | 17m 10s |  8m  6s |
| 20 | 2 | 27m 21s | 12m 23s |
| 20 | 4 | 31m 50s | 16m 46s |
| 20 | 6 | 29m 34s | 13m 12s |
| 20 | 8 | 34m  2s | 15m 46s |
| 40 | 2 | 62m 13s | 34m 21s |
| 40 | 4 | 66m 42s | 38m 54s |
| 40 | 6 | 65m 25s | 36m 48s |
| 40 | 8 | 68m 53s | 33m 59s |

##### Increase forks parameter

The following tests were run with an increased `forks` parameter, allowing for more parallel operations on the Ansible controller host (Apple M1 Max) while consuming more resources on the controller.

| Segment Hosts | Segments | Forks | Parallelism | Run 1 | Run 2 |
|:---:|:---:|:---:|---|---:|---:|
|40  |8| 21 | 20 segments + coordinator in parallel | 71m 22s | 37m 33s |
|40  |8| 41 | 40 segments + coordinator in parallel | 72m 40s | 31m 11s |
