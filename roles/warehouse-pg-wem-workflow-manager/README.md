# warehouse-pg-wem-workflow-manager

Installs the Temporal.io workflow service on the `wem_stack` host. WEM uses Temporal for background job scheduling and durable workflow execution. The role downloads the Temporal server binary from GitHub releases, bootstraps the PostgreSQL persistence databases and schema using `temporal-sql-tool`, deploys a configuration file and systemd unit, and enables the `temporal` service.

The role runs after `warehouse-pg-wem-state-database` in the `wem_stack` play so the PostgreSQL instance is available before schema bootstrap runs.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-workflow-manager
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `temporal_version` | `1.31.1` | Temporal server version downloaded from GitHub releases. |
| `temporal_db_host` | `127.0.0.1` | PostgreSQL host for Temporal persistence. |
| `temporal_db_port` | `5431` | PostgreSQL port (must not conflict with WHPG on 5432). |
| `temporal_db_name` | `temporal` | Main persistence database name. |
| `temporal_visibility_db_name` | `temporal_visibility` | Visibility database name. |
| `temporal_db_user` | `temporal` | PostgreSQL role Temporal connects as. |
| `temporal_db_password` | `""` | Password for `temporal_db_user`. Leave empty to use trust auth when the database is local. Required when `temporal_db_host` is not on localhost. |
| `temporal_num_history_shards` | `4` | History shard count. Must be a power of 2. Use 4–8 for small deployments. |
| `temporal_frontend_grpc_port` | `7233` | Temporal gRPC frontend port. |
| `temporal_manage_db` | `true` | Create PostgreSQL user/databases and apply schema when `true`. |
| `temporal_default_namespace` | `default` | Name of the default namespace to register. |
| `temporal_default_namespace_retention` | `72h` | Retention period for the default namespace. |
| `temporal_install_dir` | `/usr/local/temporal` | Directory where the Temporal binary is extracted. |
| `temporal_config_env` | `temporal-server` | Config env name — maps to `/etc/temporal/temporal-server.yaml`. |
| `temporal_service_name` | `temporal` | systemd service name. |
| `temporal_service_enabled` | `true` | Enable the service at boot. |
| `temporal_service_started` | `true` | Start the service after deployment. |
| `temporal_cli_version` | `1.7.2` | Temporal CLI version for namespace management. |

## Notes

- `configure.yml` deploys a Jinja2 template to `{{ temporal_config_dir }}/{{ temporal_config_env }}.yaml` — PostgreSQL port, broadcast address, `maxJoinDuration`, and `bindOnIP` are all rendered from Ansible variables.
- `temporal_db_password` is only required when `temporal_db_host` is a remote host. For local co-located deployments (default), leave it empty and configure `pg_hba.conf` for trust auth — the `warehouse-pg-wem-state-database` role does this automatically.
- The default namespace (`temporal_default_namespace`) is created automatically after the service starts.
