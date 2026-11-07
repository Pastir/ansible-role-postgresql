# pastir.postgresql

Installs PostgreSQL from the official PGDG repository with an exact, held version. For standalone servers
it can also manage configuration parameters, `pg_hba.conf` and the cluster start mode. On any server
(standalone or a Patroni cluster) it can manage users, databases and database privileges on the primary.

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
| `postgresql_users` | `[]` | Users (roles). Items: `name` (required), `password` (plaintext, optional), `role_attr_flags` (default `LOGIN`, e.g. `LOGIN,CREATEDB`), `conn_limit`, `update_password` (`always` \| `on_create`), `groups` (list), `state` (`present` \| `absent`). Group roles are items with `role_attr_flags: NOLOGIN` and no password |
| `postgresql_users_update_password` | `always` | `always`: set the password when it differs (hashes are compared, no change otherwise). `on_create`: only when the user is created |
| `postgresql_users_exclusive_groups` | `false` | `false`: memberships are only added. `true`: users with `groups` become members of exactly these groups, others are revoked |
| `postgresql_databases` | `[]` | Items: `name` (required), `owner`, `encoding`, `lc_collate`, `lc_ctype`, `template`, `state` (`present` \| `absent`) |
| `postgresql_allow_drop_database` | `false` | Required to drop a database (`state: absent`); without it the role stops before any change |
| `postgresql_privileges` | `[]` | `community.postgresql.postgresql_privs` items: `database`, `roles`, `privs` (required), `type` (default `database`), `objs`, `schema`, `state` |
| `postgresql_objects_primary_only` | `true` | Manage users/databases/privileges only where `pg_is_in_recovery()` is false |
| `postgresql_objects_wait_timeout` | `30` | Seconds to wait for the local instance before managing objects |
| `postgresql_objects_login_unix_socket` | `/var/run/postgresql` | Socket directory used by the modules |
| `postgresql_objects_port` | `5432` | |
| `postgresql_objects_login_user` | `postgres` | Database user; the role connects as OS user `postgres` (peer authentication) |

## Users and databases

```yaml
postgresql_users:
  - {name: readonly, role_attr_flags: NOLOGIN}   # group role, its privileges may be managed by hand
  - name: report
    groups: [readonly]                            # no password: an existing password is not touched
  - name: app
    password: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=kv/data/db:app_password') }}"
    role_attr_flags: LOGIN
    conn_limit: 50
postgresql_databases:
  - {name: app, owner: app, encoding: UTF8, lc_collate: C.UTF-8, lc_ctype: C.UTF-8, template: template0}
postgresql_privileges:
  - {database: app, roles: report, privs: CONNECT}
```

- **Primary only.** Before any change the role runs `SELECT pg_is_in_recovery()` on the local instance
  (read-only, also in `--check`). Objects are managed only where it returns `false`: on a standalone server
  and on the leader of a cluster. Replicas are skipped with a message and receive the objects by replication.
  The role does not talk to Patroni. `postgresql_objects_primary_only: false` disables the check.
- **Instance not running.** The role waits up to `postgresql_objects_wait_timeout` seconds for the instance
  (`pg_isready`). If it is still down, the objects are skipped with a message when
  `postgresql_manage_service: false` (Patroni nodes) or in `--check`; otherwise the role fails.
- **Patroni.** On a fresh cluster `pastir.postgresql` runs before `pastir.patroni`, so PostgreSQL is not running
  yet and the objects are skipped. Run the objects part again after Patroni, on all cluster nodes:

  ```yaml
  - role: pastir.patroni
  - role: pastir.postgresql
    tasks_from: objects.yml   # or the whole role again
  ```

- **Connection.** As OS user `postgres` over the unix socket (peer authentication). `python3-psycopg2` is
  installed only when one of the lists is not empty.
- **Passwords** are plaintext in the variables and hashed by PostgreSQL according to `password_encryption`
  (scram-sha-256 by default). Take them from a secret store (Vault lookup, ansible-vault), never commit them.
  The user task runs with `no_log` (Ansible then hides even the loop label); the next task lists the names of
  created/updated users. With `community.postgresql` >= 4.0 an unchanged
  password is detected against the stored SCRAM hash, so `always` is idempotent; use `on_create` if the
  password is rotated outside Ansible (note: `on_create` also ignores `conn_limit` changes of existing users).
- **Not touched.** A user without `password` keeps its current password (e.g. certificate authentication or a
  password set by hand). A database without `owner` keeps its current owner.
- **Groups.** All roles of `postgresql_users` are created first, then memberships (`groups`) are granted, so group
  roles can be listed in any order. Memberships are additive; with `postgresql_users_exclusive_groups: true` the
  memberships not listed are revoked for users that have `groups` (`groups: []` revokes all).
- **Dropping.** `state: absent` for a database requires `postgresql_allow_drop_database: true`. `state: absent`
  for a user runs `DROP ROLE` without `CASCADE`: it fails if the user still owns objects or has privileges.
- Tag `postgresql_objects` runs only this part: `--tags postgresql_objects`.

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
idempotence, reload without restart, the pending-restart report, the version guard, users/databases/privileges
(login with the password over TCP, the drop guard, no CASCADE on user drop, `--check`), and that the objects are
skipped on the Patroni-style node. The cluster case is covered by the `pastir.patroni` molecule test.

Requires the `community.postgresql` collection (`requirements.yml` in the role root).

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.postgresql
    src: git+git@github.com:Pastir/ansible-role-postgresql.git
    version: main  # or a tag
# A role cannot install collections by itself: the project lists them (see requirements.yml in the role root).
collections:
  - name: community.postgresql
    version: ">=4.0.0"
```

```bash
ansible-galaxy install -r requirements.yml   # roles and collections
```

## License

Apache-2.0
