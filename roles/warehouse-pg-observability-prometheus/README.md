# warehouse-pg-observability-prometheus

Downloads and installs a Prometheus binary directly from GitHub releases. Configures `prometheus.yml` with scrape jobs for Prometheus self-monitoring and WEM metrics. Deploys a systemd service with `--web.enable-lifecycle` and `--web.enable-remote-write-receiver` flags enabled, making Prometheus ready to receive remote-write data from WEM exporters.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-observability
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `prometheus_version` | `3.5.1` | Prometheus release version to download from GitHub. |
| `prometheus_port` | `9090` | Port Prometheus listens on. |
| `prometheus_scrape_interval` | `15s` | Scrape interval for all jobs. |
| `prometheus_evaluation_interval` | `15s` | Rule evaluation interval. |
| `prometheus_wem_port` | `8080` | Port of the WEM metrics exporter that Prometheus scrapes. |
| `prometheus_alertmanager_port` | `9093` | Alertmanager port referenced in the Prometheus configuration. |
| `prometheus_config_dir` | `/etc/prometheus` | Directory for the `prometheus.yml` configuration file. |
| `prometheus_data_dir` | `/var/lib/prometheus` | Directory where Prometheus stores its TSDB data. |
