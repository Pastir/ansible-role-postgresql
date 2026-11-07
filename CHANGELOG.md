# Changelog

## 2.2.1 — 2026-11

### Fixed
- `ansible-galaxy role install` no longer fails with "Expected role dependencies to be a list": the collection
  list moved from `meta/requirements.yml` (galaxy reads that file as role dependencies) to `requirements.yml` in the
  role root. A role cannot install collections; the project lists them in its own `requirements.yml`
  (see Installation in the README).

## 2.2.0 — 2026-11

### Added
- Users, databases and database privileges: `postgresql_users`, `postgresql_users_update_password`,
  `postgresql_databases`, `postgresql_privileges` (`community.postgresql`, now in `meta/requirements.yml`).
- Group memberships: `groups` in `postgresql_users` items (`postgresql_membership`), additive by default,
  `postgresql_users_exclusive_groups: true` revokes unlisted ones. Group roles are plain items with `NOLOGIN`.
  Users without `password` and databases without `owner` keep their current password / owner.
- Changes are made only on the primary (`SELECT pg_is_in_recovery()`), so the same play works on a standalone
  server and on every node of a Patroni cluster (`postgresql_objects_primary_only`).
- `postgresql_objects_wait_timeout`: wait for the instance; skip with a message on nodes where the role does not
  manage the service (Patroni) and in `--check`.
- Dropping a database requires `postgresql_allow_drop_database: true`; users are dropped without CASCADE.
- `tasks_from: objects.yml` entry point and the `postgresql_objects` tag.
- `python3-psycopg2` is installed when any of the lists is not empty.
- Non-breaking: all lists are empty by default, nothing changes for existing playbooks.

## 2.1.0 — 2026-11

### Changed
- `start.conf`: only the mode line (`auto`/`manual`/`disabled`) is managed (`lineinfile`), the rest of the file
  is kept. Variable and values are unchanged. Files written by 2.0.0 keep their old
  "Managed by Ansible" comment lines; remove them by hand if needed.

### Fixed
- The stock comments of `start.conf` were replaced with the role's own text (non-zero diff on existing servers).
- `start.conf` was rewritten on every run when `pastir.patroni` ran on the same host (different file text).
  Both roles now converge to the same file; use `postgresql_start_conf: manual` on Patroni nodes.
- The PGDG key download failed on a single TLS timeout; it now retries (5 times, 30 s timeout).

## 2.0.0 — 2026-11

### BREAKING
- The default major version is 15 (was 17), set it explicitly with `postgresql_version`.
- `postgresql-doc-*` and `libpq-dev` are no longer installed by default. Use `postgresql_extra_packages`.
- Removed the `__postgresql_*` override style (`postgresql_repo`, `repo_signing_key`, `postgresql_daemon`,
  `postgresql_restarted_state`). Use the public variables from `defaults/main.yml`.
- The repository is added as `/etc/apt/sources.list.d/pgdg.sources` with the key in `/etc/apt/keyrings`.
  The old `pgdg.list.list` and `/usr/share/keyrings/ACCC4CF8.asc` files are not removed automatically.

### Added
- Exact version pinning (`postgresql_minor_version`), `apt-mark hold`, and a guard against implicit upgrades.
- `postgresql_create_main_cluster`, `postgresql_start_conf`, `postgresql_manage_service` for Patroni nodes.
- Standalone configuration: `postgresql_conf` (conf.d), `postgresql_hba_entries`, reload with a
  pending-restart report.
- Molecule tests for Ubuntu 24.04 and 26.04.

### Fixed
- The handler used an undefined variable.
