# ansible-percona

Installs and configures [Percona Server for MySQL](https://www.percona.com/software/mysql-database/percona-server) 8.4 LTS from the Percona APT repository. Optionally creates databases and users with generated or supplied passwords.

## Requirements

- Debian or Ubuntu with APT and systemd, on a release for which Percona provides the `ps-84-lts` repository.
- Root privileges on the managed host, either directly or with `become: true`.
- Python 3 with `python3-apt` on the managed host for gathering installed package facts; the role gathers these facts itself.
- The `ansible.mysql` collection when managing databases or users (`ansible-galaxy collection install ansible.mysql`). The role uses `ansible.mysql.mysql_db` and `ansible.mysql.mysql_user` and installs `python3-pymysql` on the managed host when needed.
- A writable password directory on the Ansible controller when using generated passwords.

## Example playbook

```yaml
---
- name: Configure Percona Server
  hosts: myserver
  become: true
  roles:
    - role: sunfoxcz.percona
      # Omit to generate and persist the root password on the controller.
      # percona_root_password: "{{ vault_percona_root_password }}"
      percona_databases:
        - mydatabase
      percona_users:
        - name: user1
          host: 127.0.0.1
          database: mydatabase
          # Omit to generate and persist the user password on the controller.
          # password: "{{ vault_user1_password }}"
```

Each user requires `name`, `host`, and `database`. The role grants `ALL` privileges on `database.*` to that user at the specified host. With name resolution disabled, use IP addresses or MySQL host patterns for remote accounts rather than DNS names. Local socket accounts can use `localhost`.

## Role variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `percona_root_password` | Generated with the password lookup | Root password used for initial installation and `/root/.my.cnf`. |
| `percona_local_passdb` | `local_passdb` if defined, otherwise `~/.ansible_percona_passwords` | Directory on the controller for generated passwords. |
| `percona_pass_len` | `24` | Length of newly generated passwords. |
| `percona_pass_complexity` | `ascii_letters,digits` | Character sets passed to the Ansible password lookup. |
| `percona_db_collation` | `utf8mb4_general_ci` | Collation for newly created databases. |
| `percona_databases` | Undefined; creation skipped | List of database names to create. |
| `percona_users` | Undefined; creation skipped | List of user definitions, each with an optional `password`. |

Generated passwords are stored in plaintext on the controller and reused on subsequent runs:

- Root: `<percona_local_passdb>/<inventory_hostname>/percona.root`
- User: `<percona_local_passdb>/<inventory_hostname>/percona.<name>`

Keep this directory private and retain it between runs. Generated user passwords are keyed by username, so accounts with the same name at different hosts share a generated password. Changing the password length or character sets does not regenerate existing passwords. Supplied passwords can be stored in Ansible Vault.

The role writes root client credentials to `/root/.my.cnf` with mode `0600`. Changing `percona_root_password` after installation updates this file but does not change the existing MySQL root account password; coordinate root password rotation separately.

## Installation and configuration

The role installs the generic `percona-release` package if absent and enables the Percona 8.4 LTS repository with `percona-release setup -y ps-84-lts`. This setup command can disable other Percona product repositories; see the [Percona repository documentation](https://docs.percona.com/percona-software-repositories/percona-release.html). It installs the unversioned `percona-server-server` and `percona-server-client` packages, then enables and starts the `mysql` service. On first installation, it preseeds the `percona-server-server/root-pass` and `percona-server-server/re-root-pass` debconf questions with the root password.

The role manages these files in `/etc/mysql/conf.d/` and restarts MySQL when they change:

| File | Configuration |
| --- | --- |
| `binlog.cnf` | Disables binary logging with `skip_log_bin`. |
| `client.cnf` | Sets the client character set to `utf8mb4`. |
| `mysqldump.cnf` | Enables quick dumps and quoted names, with a 64 MB packet limit. |
| `server.cnf` | Server character set, authentication, telemetry, logging, and memory settings. |

Pending configuration handlers run before databases and users are managed so the authentication plugin is available on the first run.

The previous role-managed `/etc/mysql/conf.d/mysqld.cnf` is removed because it includes query-cache options unsupported by MySQL 8.4.

The shipped server configuration includes:

- `utf8mb4` as the server character set and `mysql_native_password` as the authentication policy. The role explicitly enables this legacy plugin with `mysql_native_password = ON` because MySQL 8.4 disables it by default. This preserves compatibility with the existing user-management tasks and older application clients; the upstream default for new 8.4 installations is `caching_sha2_password`.
- `sql_mode = NO_ENGINE_SUBSTITUTION` and disabled hostname resolution.
- Error and slow-query logs under `/var/log/mysql`, with slow-query logging enabled.
- A fixed 2 GB InnoDB buffer pool, two buffer-pool instances, and `O_DIRECT` flushing.
- `innodb_doublewrite = 0` and `innodb_flush_log_at_trx_commit = 0`. These settings trade crash durability for performance: committed transactions can be lost and pages can be corrupted after a crash. Review them for the target workload.

Binary logging is disabled, so this configuration does not provide binlog-based replication or point-in-time recovery. Server tuning is supplied as static files under `files/`; it is not exposed through role variables.

Installation-time telemetry is disabled with `PERCONA_TELEMETRY_DISABLE=1`. The role also stops and disables `percona-telemetry-agent` and sets `percona_telemetry_disable = 1` in the server configuration. The telemetry package and its system group are retained for package dependencies and upgrades. See [Percona telemetry documentation](https://docs.percona.com/percona-server/8.4/telemetry.html).

The systemd drop-in `/lib/systemd/system/mysql.service.d/file-limit.conf` sets `LimitNOFILE=infinity`. MySQL separately requests `open_files_limit = 10000` in `server.cnf`.

## Changes from the previous role

- Percona Server 8.4 LTS replaces 8.0 and the earlier configurable 5.6/5.7 packages. `percona_version` is no longer used.
- `percona_nofile` is no longer used; the systemd and MySQL limits are configured as described above.
- Client and server character sets change from `utf8` to `utf8mb4`, and newly created databases default to `utf8mb4_general_ci`. Existing databases and tables are not converted.
- Repository configuration uses `percona-release` instead of directly managing an APT key and repository entry.

This role targets new installations; it does not implement a database migration or a validated major-version upgrade procedure. Back up existing data and follow the [Percona 8.4 upgrade documentation](https://docs.percona.com/percona-server/8.4/upgrade-procedures.html) before upgrading an existing deployment. Existing `mysql_native_password` accounts require the plugin to remain enabled. Remove any old `default_authentication_plugin` option (including package-generated `default-auth-override.cnf`) before starting 8.4 because that variable was removed.

## License

Licensed under the MIT license. See [LICENSE](LICENSE.md) for details.
