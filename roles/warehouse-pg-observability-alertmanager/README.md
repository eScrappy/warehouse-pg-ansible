# warehouse-pg-observability-alertmanager

Downloads and installs an Alertmanager binary directly from GitHub releases. Deploys a minimal configuration with a default null receiver. Intended to be co-located with the WEM workflow manager on the `wem_stack` host.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags wem
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `alertmanager_version` | `0.33.0` | Alertmanager release version to download from GitHub. |
| `alertmanager_port` | `9093` | Port Alertmanager listens on. |
| `alertmanager_config_dir` | `/etc/alertmanager` | Directory for the Alertmanager configuration file. |
| `alertmanager_data_dir` | `/var/lib/alertmanager` | Directory where Alertmanager stores its data. |
| `alertmanager_resolve_timeout` | `5m` | How long to wait before declaring an alert resolved if no new alerts arrive. |
| `alertmanager_slack_api_url` | `""` | Slack incoming webhook URL. Leave empty to disable Slack notifications. |
| `alertmanager_pagerduty_url` | `""` | PagerDuty events API URL. Leave empty to disable PagerDuty notifications. |
| `alertmanager_route_group_by` | `[alertname, cluster, service]` | Labels to group alerts by. |
| `alertmanager_route_group_wait` | `30s` | How long to wait before sending the first notification for a new group. |
| `alertmanager_route_group_interval` | `5m` | How long to wait before sending a notification for new alerts in an existing group. |
| `alertmanager_route_repeat_interval` | `4h` | How long to wait before re-sending a notification if it has not been resolved. |
| `alertmanager_route_receiver` | `null-receiver` | Default receiver for all alerts. |
| `alertmanager_routes` | `[]` | List of child route objects for custom alert routing. |
| `alertmanager_receivers` | `[{name: null-receiver}]` | List of receiver configurations (Slack, PagerDuty, etc.). |
| `alertmanager_inhibit_rules` | `[]` | List of inhibition rules to mute alerts when others are firing. |
