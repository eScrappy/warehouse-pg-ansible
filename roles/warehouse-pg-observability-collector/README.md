# warehouse-pg-observability-collector

Installs the `edb-whpg-observability-collector` package from the EDB repository on **all cluster nodes**. Enables the Grafana Alloy service on every node. On the coordinator only, configures `collector.conf` with the WHPG DSN, Loki endpoint, and Prometheus endpoint, then runs the `./deploy-observability` script as `gpadmin`.

The role also grants `gpadmin` passwordless `sudo` on the coordinator to allow the deployment script to run without a password prompt.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-observability-collector
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `obs_collector_package_name` | `edb-whpg-observability-collector` | RPM package name from the EDB repository. |
| `obs_collector_version` | `""` | Pin a specific RPM version (e.g. `1.2.3`). Leave empty to install whatever the EDB repository provides. |
| `collector_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy. Accepted values: `install-but-no-automatic-upgrades`, `always-upgrade-to-latest-minor`, `absent`. |
| `obs_collector_config_file` | `/var/lib/whpg-observability-collector/collector.conf` | Path to `collector.conf`. |
| `obs_collector_loki_endpoint` | `http://localhost:3100/loki/api/v1/push` | Full Loki push endpoint. Set via `loki_endpoint` in group_vars or inventory. |
| `obs_collector_prometheus_endpoint` | `http://localhost:9090/api/v1/write` | Full Prometheus remote-write endpoint. Set via `prometheus_endpoint` in group_vars or inventory. |
| `obs_collector_whpg_host` | `127.0.0.1` | Host or IP of the WarehousePG coordinator. |
| `obs_collector_whpg_port` | *(from `whpg_port`)* | Port the WarehousePG coordinator listens on. |
| `obs_collector_whpg_dbname` | *(from `whpg_dbname`, default `postgres`)* | Database name the collector connects to. |
| `obs_collector_whpg_user` | *(from `whpg_user`)* | Database user the collector connects as. |
| `obs_collector_whpg_password` | *(from `whpg_password`)* | Password for `obs_collector_whpg_user`. |
| `obs_collector_whpg_sslmode` | `disable` | SSL mode for the WHPG DSN connection. |

## Notes

- Configuration is applied **coordinator only** — segment nodes only get the package and Alloy service.
- `obs_collector_loki_endpoint` and `obs_collector_prometheus_endpoint` derive from `loki_endpoint` / `prometheus_endpoint` which are set in `group_vars/whpg_clusters/vars.yml` using the monitoring host IP.
- `obs_collector_whpg_*` variables derive from the corresponding `whpg_*` group vars — override the `obs_collector_whpg_*` variables directly only when the collector DSN must differ from the cluster-wide defaults.
