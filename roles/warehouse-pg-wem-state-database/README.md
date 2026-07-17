# warehouse-pg-wem-state-database

Installs PostgreSQL 15 on the `wem_stack` host via the DNF AppStream module. After installation, the role creates the WEM state database and user, the Temporal persistence databases and user, and configures `pg_hba.conf` to allow all users to connect from the local loopback address via trust auth.

Runs on port `5433` by default to avoid conflicting with WHPG on port `5432`.

## Run

```bash
ansible-playbook -i inventory-with-standby-and-monitoring.yml playbook.yml --tags warehouse-pg-wem-state-database
```

## Key variables

| Variable | Default | Description |
|---|---|---|
| `pg_version` | `15` | PostgreSQL major version to install via DNF AppStream. |
| `pg_port` | `5433` | Port PostgreSQL listens on. Uses 5433 to avoid conflict with WHPG on 5432. |
| `pg_data_dir` | `/var/lib/pgsql/data` | PostgreSQL data directory. |
| `pg_service` | `postgresql` | systemd service name. |
| `wem_state_db` | `wem` | Name of the WEM state database to create. |
| `wem_state_db_user` | `wem_admin` | PostgreSQL role to create for WEM state access. |
| `wem_state_db_password` | `""` | Password for `wem_state_db_user`. Leave empty to use trust auth when the database is local. Required when `wem_state_db_host` is not on localhost. |
| `temporal_db_name` | `temporal` | Temporal main persistence database name. |
| `temporal_visibility_db_name` | `temporal_visibility` | Temporal visibility database name. |
| `temporal_db_user` | `temporal` | PostgreSQL role to create for Temporal persistence. |
| `temporal_db_password` | `""` | Password for `temporal_db_user`. Leave empty to use trust auth when the database is local. Required when `temporal_db_host` is not on localhost. |
