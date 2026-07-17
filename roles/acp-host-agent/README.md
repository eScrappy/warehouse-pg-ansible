# acp-host-agent

Installs the `acp-host-agent` RPM from the EDB repository on **coordinator nodes only** (`whpg_role: coordinator`). The agent connects to the Temporal service on the `wem_stack` host and registers `whpg.cluster.*` activity handlers that WEM uses to execute cluster management operations remotely.

The agent's configuration file `/etc/edb/acp-host-agent/acp-host-agent.conf` is edited in place — only the keys controlled by Ansible variables are written; all RPM-shipped commented examples remain untouched.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags acp-host-agent
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `acp_agent_package_name` | `acp-host-agent` | RPM package name from the EDB repository. |
| `acp_agent_version` | `""` | Pin a specific RPM version (e.g. `1.2.3`). Leave empty to install whatever the EDB repository provides. |
| `acp_agent_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy. Accepted values: `install-but-no-automatic-upgrades`, `always-upgrade-to-latest-minor`, `absent`. |
| `acp_agent_temporal_address` | `{{ temporal_address \| default('127.0.0.1:7233') }}` | Temporal frontend address (`host:port`). Derives from `temporal_address` which is set in inventory or resolved from `wem_host` in the playbook. |
| `acp_agent_temporal_namespace` | `wem` | Temporal namespace the agent connects to. |
| `acp_agent_temporal_task_queue` | `remote-tasks` | Task queue the agent polls. |
| `acp_agent_temporal_api_key` | `""` | Temporal Cloud API key. Only written when non-empty. |
| `acp_agent_temporal_tls_cert` | `""` | Path to mTLS client certificate. Only written when non-empty. |
| `acp_agent_temporal_tls_key` | `""` | Path to mTLS client key. Only written when non-empty. |
| `acp_agent_temporal_tls_server_ca_cert` | `""` | Path to CA cert for server verification. Only written when non-empty. |
| `acp_agent_temporal_tls_server_name` | `""` | TLS SNI server name override. Only written when non-empty. |
| `acp_agent_id` | `""` | Stable per-host identifier. Leave empty to auto-derive from DMI UUID / MAC. |
| `acp_agent_state_dir` | `""` | Directory where the Ed25519 enrollment keypair is stored. Leave empty to use the RPM default. |
| `acp_agent_gphome` | `""` | Path to WHPG installation (e.g. `/usr/edb/whpg7`). Enables `whpg.cluster.*` handlers when set. |
| `greenplum_path` | `/usr/local/greenplum-db/greenplum_path.sh` | Path to `greenplum_path.sh`, sourced before running WHPG commands. |
| `acp_agent_rust_log` | `info` | Log level filter (`error`, `warn`, `info`, `debug`, `trace`). |

## Notes

- Only runs on hosts with `whpg_role: coordinator`.
- `temporal_address` is resolved automatically from `wem_host` in the playbook — it points to the WEM host's private IP and Temporal gRPC port. Override via inventory or `-e temporal_address=<host>:<port>` to point at a Temporal instance running elsewhere.
- To enable WHPG cluster management handlers, set `acp_agent_gphome: "/usr/edb/whpg7"`.
