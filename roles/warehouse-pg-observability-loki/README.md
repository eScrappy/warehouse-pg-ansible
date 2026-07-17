# warehouse-pg-observability-loki

Downloads and installs a Loki binary directly from GitHub releases. Runs in single-binary filesystem mode with a TSDB v13 schema. Intended to be co-located with Prometheus on the monitoring instance.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-observability
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `loki_version` | `3.5.0` | Loki release version to download from GitHub. |
| `loki_port` | `3100` | Port Loki listens on. |
| `loki_config_dir` | `/etc/loki` | Directory for the Loki configuration file. |
| `loki_data_dir` | `/var/lib/loki` | Directory where Loki stores chunks and the index. |
| `loki_retention_period` | `744h` | Log retention period (e.g. `744h` = 31 days). |
