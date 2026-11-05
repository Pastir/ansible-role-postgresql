# pastir.postgresql

Installs PostgreSQL from the official PGDG repository with an exact, held version. For standalone servers
it can also manage configuration parameters, `pg_hba.conf` and the cluster start mode.

What the role guarantees:
- **No implicit upgrades.** If the installed minor version differs from `postgresql_minor_version`, the
  role stops. A minor upgrade restarts the server, so it is done only with
  `-e postgresql_allow_upgrade=true`.
- **No implicit restarts.** Changed parameters are applied with a reload. If some of them need a restart,
  the role lists them and leaves the restart to you, unless `postgresql_restart_allowed: true`.
- **`postgresql.conf` is not rewritten.** Parameters go to `conf.d/90-ansible.conf`, which overrides it.

## Patroni nodes

On nodes managed by Patroni, the role only installs the binaries:

```yaml
postgresql_create_main_cluster: false  # do not let the package run initdb (fresh install)
postgresql_start_conf: manual          # never start the cluster behind Patroni's back
postgresql_manage_service: false
```

Leave `postgresql_conf` and `postgresql_hba_entries` empty there: Patroni owns that configuration.

### start.conf and pastir.patroni

The role changes only the mode line of `start.conf` (`auto`, `manual` or `disabled`) and keeps the rest of the
file, including the stock comments written by `pg_createcluster`. If the file does not exist yet (but the cluster
config directory does), it is created with just the mode line.

`pastir.patroni` (`patroni_pg_start_conf_manual: true`, the default) manages the same line the same way and sets it
to `manual`. When both roles run on one host, use `postgresql_start_conf: manual`: then both roles produce the same
file and a repeated run reports no changes. With any other value the roles would rewrite the line on every run,
so `pastir.patroni` stops with an error when it sees `postgresql_start_conf` other than `manual`.

## Variables

| Variable | Default | Description |
|---|---|---|
| `postgresql_version` | `"15"` | Major version |
| `postgresql_minor_version` | `""` | Exact version to install and keep, e.g. `"15.19"`. If empty, the latest minor of the major version is installed |
| `postgresql_packages` | server + client | Packages to install |
| `postgresql_extra_packages` | `[]` | Extensions etc., not pinned |
| `postgresql_allow_upgrade` | `false` | Allow changing the installed minor version (restarts the server) |
| `postgresql_hold` | `true` | `apt-mark hold` the server and client packages |
| `postgresql_repo_enabled` | `true` | Add the PGDG repository |
| `postgresql_repo_url`, `postgresql_repo_key_url` | PGDG | Mirror can be used |
| `postgresql_cluster_name` | `main` | |
| `postgresql_data_dir`, `postgresql_config_dir` | Debian layout | |
| `postgresql_create_main_cluster` | `true` | `create_main_cluster` in `createcluster.conf` |
| `postgresql_start_conf` | `auto` | `auto` \| `manual` \| `disabled`. Only the mode line of `start.conf` is managed. Use `manual` on Patroni nodes |
| `postgresql_manage_service` | `true` | Enable and start `postgresql@<ver>-<cluster>` |
| `postgresql_conf` | `{}` | Parameters for `conf.d/<postgresql_conf_file>`. String values that need quotes must include them: `listen_addresses: "'*'"` |
| `postgresql_conf_file` | `90-ansible.conf` | |
| `postgresql_hba_entries` | `[]` | If not empty, `pg_hba.conf` is fully managed. Items: `type`, `database`, `user`, `address`, `method`, `options`, `comment` |
| `postgresql_restart_allowed` | `false` | Restart automatically when a changed parameter needs it |

## Example

```yaml
- hosts: db
  become: true
  roles:
    - role: pastir.postgresql
      vars:
        postgresql_version: "15"
        postgresql_minor_version: "15.19"
        postgresql_conf:
          max_connections: 200
          shared_buffers: 4GB
          listen_addresses: "'*'"
        postgresql_hba_entries:
          - {type: local, database: all, user: postgres, method: peer}
          - {type: hostssl, database: app, user: app, address: 10.0.0.0/8, method: scram-sha-256}
```

### Taking over an existing server

Put the parameters you want to control into `postgresql_conf`. The existing `postgresql.conf` stays as it is.
`start.conf` keeps its comments, only the mode line is set.
For `pg_hba.conf`, copy the current rules into `postgresql_hba_entries` first and check with `--check --diff`.

## Testing

`molecule test`: standalone servers on 24.04 and 26.04, plus a Patroni-style node. The test checks
idempotence, reload without restart, the pending-restart report and the version guard.

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.postgresql
    src: git+git@github.com:Pastir/ansible-role-postgresql.git
    version: main  # or a tag
```

```bash
ansible-galaxy install -r requirements.yml
```

## License

Apache-2.0
