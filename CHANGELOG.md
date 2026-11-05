# Changelog

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
