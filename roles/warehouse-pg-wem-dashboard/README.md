# warehouse-pg-wem-dashboard

Installs the WarehousePG Enterprise Manager (WEM) — a monitoring and administration dashboard for WarehousePG. The role installs the `whpg-enterprise-manager` RPM, edits the RPM-shipped `/etc/wem/wem.conf` in place, and enables the `wem` systemd service.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-dashboard
```

## Where the role runs

The role targets `wem_stack` — which can be a dedicated standby node or a separate monitoring instance. Set via inventory.
The role is assigned per-host, **outside** of the `whpg_clusters` inventory group. The target host can be:

- A dedicated standalone host (no WHPG installed locally) — WEM connects to the WarehousePG cluster over the network.
- A WHPG coordinator.
- A WHPG standby.

The operator chooses by adding `warehouse-pg-wem-dashboard` to a play that targets the desired host(s).


## Prerequisites

- *EDB repository* configured on the target host. The role does not install it; on an air-gapped host the operator must arrange this manually. On a coordinator/standby it's already configured via `edb-repository`.

- *A WHPG database user* with superuser privileges. Create one via:

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags roles \
  -e '{"whpg_db_roles": [{"name": "wem_user", "password": "{{ whpg_password }}", "superuser": true}]}'
```

> **Important:** when WEM is co-located with WarehousePG (coordinator or standby), `WHPG_UNIX_USER` and `WHPG_UNIX_GROUP` **must hold the same values in both plays**. Both roles create or chown files for that user, and a mismatch leaves the host with two different identities for the same files. Set the variables once in `group_vars` / inventory rather than per-play to guarantee this.

## Key variables

| Variable | Default | Description |
|---|---|---|
| `whpg_host` | `127.0.0.1` | **Required.** Hostname or IP of the WarehousePG cluster WEM monitors. |
| `whpg_port` | `5432` | WarehousePG coordinator port. |
| `whpg_user` | `wem_user` | **Required.** Database user WEM connects as. |
| `whpg_password` | `""` | Password for `whpg_user`. Required when `whpg_host` is not on localhost. |
| `whpg_database` | `""` | WarehousePG database WEM connects to. Leave empty to use the default. |
| `wem_state_db_host` | `127.0.0.1` | **Required.** Host of the WEM state PostgreSQL instance. |
| `wem_state_db_port` | `5433` | WEM state database port. |
| `wem_state_db_user` | `wem_admin` | **Required.** PostgreSQL user for WEM state access. |
| `wem_state_db_password` | `""` | Password for `wem_state_db_user`. Required when `wem_state_db_host` is not on localhost. |
| `wem_state_db` | `""` | WEM state database name. Leave empty to use the default. |
| `wem_port` | `8080` | Port the WEM web interface listens on. |
| `wem_admin_password` | *(auto-generated)* | Initial admin password. Leave unset to have the RPM generate one on first run. Set explicitly to manage it via Ansible — provide via vault. |
| `wem_cookie_secret` | *(auto-generated)* | Session cookie secret. Leave unset to have the RPM generate one on first run. |
| `wem_insecure_cookies` | `1` | Set to `"1"` when the dashboard is not served over HTTPS. |
| `wem_prometheus_url` | `http://localhost:9090` | Prometheus endpoint URL. |
| `wem_loki_url` | `http://localhost:3100` | Loki endpoint URL. |
| `wem_alertmanager_url` | `http://localhost:9093` | Alertmanager endpoint URL. |
| `temporal_address` | `localhost:7233` | Temporal frontend address for WEM. |
| `wem_upgrade` | `install-but-no-automatic-upgrades` | Install/upgrade/remove policy. |

## Package state

| Value | Effect |
|---|---|
| `install-but-no-automatic-upgrades` (default) | Install if missing, don't auto-upgrade |
| `always-upgrade-to-latest-minor` | Upgrade to latest on every run |
| `absent` | Remove the RPM |

## Other knobs

See `defaults/main.yml` for the full list including: exporter tuning (`wem_exporter_*`), remote write receiver (`wem_remote_write_receiver_*`), PXF integration, and service control.
