# Vaultwarden

####  This role was previously known as `dmaes.bitwardenrs` and `dmaes.vaultwarden`

Since the Bitwarden_RS project changed names to Vaultwarden, so did this role (see #12 for more info).
Force this change, we changed everything from `bitwardenrs` to `vaultwarden` (variables used in the ansible code, but also directories, user, systemd service, etc.)
When making the switch:
* Stop old `bitwarden_rs` service
* Make a backup of both files and database for good measure
* Update your ansible code to work with new role
* Either point `vaultwarden_directory` to the old directory, or move stuff to the new default (`/opt/vaultwarden`). Also pay attention to `vaultwarden_datadir` if using a custom one.
* The new vaultwarden user should get the same rights on the database as your previous bitwardenrs user
  * for postgres:
     * su - postgres
     * psql
     * `postgres-# GRANT bitwardenrs TO vaultwarden;`
* Run ansible, this will create everything under the new name (user and service, not directory)
* Cleanup old user, service (and possibly (data)directory)


[![Build Status](https://app.travis-ci.com/JensTimmerman/ansible-role-vaultwarden.svg?branch=main)](https://app.travis-ci.com/JensTimmerman/ansible-role-vaultwarden)

Builds, installs and configures [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (without Docker).

*Only tested on Rocky linux 9*

## Requirements
* Requirements for the [unarchive](https://docs.ansible.com/ansible/latest/modules/unarchive_module.html)-module
* Requirements for the [package](https://docs.ansible.com/ansible/latest/modules/package_module.html)-module
* wget or curl
* jinja => v2.11
* Systemd (optional)

At least 1 GB of RAM (2GB recommended), the rustc compiler will compile bitwarden on your host which takes a lot of ram. Possible symptoms of not enought RAM are:
`Memory cgroup out of memory: Killed process 709453 (rustc) total-vm:2668356kB, anon-rss:955680kB, file-rss:0kB, shmem-rss:0kB, UID:996 pgtables:4516kB oom_score_adj:0`

## Role Variables
| Variable | Description | Default value |
| --- | --- | --- |
| `vaultwarden_directory` | Where to install Vaultwarden | `/opt/vaultwarden` |
| `vaultwarden_version` | Which version to install | `latest` |
| `vaultwarden_webvault` | Install the patched webvault | `true` |
| `vaultwarden_webvault_version` | Version of the webvault to install (This will be ignored if vaultwarden version is set to latest, the corresponding latest webvault version will be installed)  This is the version without the 'v' prefix | `2022.12.0` |
| `vaultwarden_build_backend` | The database-type to compile for | *vaultwarden_version-specific(\*)* |
| `vaultwarden_force_recompile` | Force recompile binary, (e.g. you switched backends on same server | `false` |
| `vaultwarden_config` | Key-value environment variables for the Vaultwarden `.env` file | `{ DOMAIN: "https://{{ ansible_fqdn }}/" }` |
| `vaultwarden_datadir` | Vaultwarden data directory (does not configure, only create and used for e.g. keys) | `{{ vaultwarden_directory }}/data` |
| `vaultwarden_encryption_key` | RSA key to use for encryption (content, not file), empty string to not copy, Vaultwarden should generate one if non-existing | `""` |
| `vaultwarden_force_encryption_key` | Force changing encryption key if it already exists (DANGEROUS!) | `false` |
| `vaultwarden_systemd` | Manage systemd service | `{{ ansible_service_mgr == 'systemd' }}` |
| `vaultwarden_required_service` | Make Systemd wait for other service before starting vaultwarden | `undefined` |
| `vaultwarden_proc_limit` | Specify the LimitNPROC for the systemd service file | `128` |
*(\*)Starting from `vaultwarden_version: 1.17.0`: defaults to `sqlite,mysql,postgresql`, before: defaults to `sqlite`*

## Example Playbook
```yaml
- hosts: servers
  vars:
    vaultwarden_configure: yes
    vaultwarden_port: "443"
    vaultwarden_build_backend: "sqlite,postgresql"
    vaultwarden_required_service: "postgresql.service"  # add this if postgresql starts to slow, this will make systemd wait
    admin_token: !vault |
      $ANSIBLE_VAULT;1.1;AES256
      ...
    vaultwarden_config:
      DOMAIN: "https://example.com/"
      DOMAIN_PATH: "/vaultwarden"  # results in a domain of https://example.com/vaultwarden/, needs to start with a '/'
      ADMIN_TOKEN: "{{ admin_token }}"
      DATABASE_URL: "postgresql:///vaultwarden?host=/run/postgresql/"
      SIGNUPS_ALLOWED: 'false'
      SIGNUPS_VERIFY: 'true'
      SIGNUPS_DOMAINS_WHITELIST: 'example.com'
      INVITATIONS_ALLOWED: 'true'
      SMTP_HOST: 'mail.example.com'
      SMTP_FROM: 'vaultwarden@example.com'
      SMTP_FROM_NAME: 'vaultwarden'
      ROCKET_ADDRESS: '0.0.0.0'
  roles:
    - jenstimmerman.vaultwarden
```

## Changelog

### v1.5.0 (unreleased)
* **Set `VW_VERSION` during build** ([#40](https://github.com/JensTimmerman/ansible-role-vaultwarden/issues/40)): recent Bitwarden clients require a version string baked into the binary. The `VW_VERSION` environment variable is now passed to cargo during compilation, using the resolved target version.
* **Systemd service hardening** ([#27](https://github.com/JensTimmerman/ansible-role-vaultwarden/issues/27)): the generated systemd unit now includes additional restrictions: `NoNewPrivileges`, `CapabilityBoundingSet=`, `LockPersonality`, `KeyringMode=private`, `RemoveIPC`, `DevicePolicy=closed`, `ProtectClock`, `ProtectControlGroups`, `ProtectHostname`, `ProtectKernelLogs`, `ProtectKernelModules`, `ProtectKernelTunables`, `ProtectProc=invisible`, `RestrictNamespaces`, `RestrictRealtime`, `RestrictSUIDSGID`, `RestrictAddressFamilies`, `SystemCallArchitectures=native`, and `SystemCallFilter=@system-service`.
* **`ReadWriteDirectories` replaced by `ReadWritePaths`**: the deprecated directive is now replaced with `ReadWritePaths`, which also covers `vaultwarden_datadir`. If you set a custom `vaultwarden_datadir` outside `vaultwarden_directory`, verify that vaultwarden can still write to it after upgrading.

### v1.4.1
* **Bugfix: vaultwarden fails to start**: the default `DOMAIN` config value contained a stray trailing quote, producing an invalid `.env`. Vaultwarden would exit immediately with `Failed parsing environment file: .env` and crash-loop under systemd.
* **Fix builds on OpenSSL < 3**: newer vaultwarden dependencies (webauthn-rs) require OpenSSL >= 3; molecule tests now run on Debian 12 instead of Debian 11.
* Molecule test hardening: fake previous-build fixture moved to prepare, removed always-changed `daemon_reload` from the start task, and verification now asserts the service is active and answering on `:8000/alive`.

### v1.4.0
* **Clean up previous build directories**: before installing rust and compiling, all previous `vaultwarden-*` build trees under `vaultwarden_directory` are removed (each keeps a multi-GB `target/` dir). This prevents updates from failing with `No space left on device` on smaller disks.
* Molecule test coverage for the cleanup.
* ansible-lint: exclude the test playbook; move Dependabot config to `.github/dependabot.yml`.

## License
MIT
