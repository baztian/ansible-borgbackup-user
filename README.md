# baztian.borgbackup_user

Configure borgmatic backups for one existing non-root account. This role can run alongside
`baztian.borgbackup`: each role has its own configs, timer, and Uptime Kuma helper.

The role installs `borgbackup`, `borgmatic`, `acl`, and `dbus-user-session` with apt. Package installation may enable the
distribution's system-wide `borgmatic.timer`; this role does not manage that timer. The target needs systemd,
logind, and a working user D-Bus. Linger is enabled by default to keep the user manager available.

## Installation

After the role has been published to Ansible Galaxy:

```sh
ansible-galaxy role install baztian.borgbackup_user
```

Before publication, place a copy of this repository in a playbook's `roles/borgbackup_user` directory and refer
to the role as `borgbackup_user`.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `borgbackup_user_name` | `""` | Required existing non-root account. |
| `borgbackup_user_config_directory` | `""` | Optional absolute directory inside the home; defaults to `~/.config/borgmatic.d`. |
| `borgbackup_user_configs` | `{}` | Mapping of config names to borgmatic configurations. |
| `borgbackup_user_timer` | `Weekly` | Systemd `OnCalendar` value. |
| `borgbackup_user_init_repos` | `true` | Initialize repositories before starting the timer. |
| `borgbackup_user_init_encryption` | `""` | Required explicit Borg encryption mode when initialization is enabled. |
| `borgbackup_user_enable_linger` | `true` | Enable linger; set `false` when another role manages the user manager. |
| `borgbackup_user_kuma` | `{}` | Optional Uptime Kuma URL and repository matchers. |

The role targets Ubuntu 24.04 and Debian 13. For example:

```yaml
borgbackup_user_name: backup-job
borgbackup_user_init_encryption: repokey
borgbackup_user_timer: daily
borgbackup_user_configs:
  backup-home:
    source_directories:
      - /srv/backup-source
    repositories:
      - path: /srv/backup-repository
    encryption_passphrase: "{{ vault_borgmatic_passphrase }}"
```

Add the role to a playbook, using `borgbackup_user` as the role name when installed locally:

```yaml
- hosts: servers
  become: true
  roles:
    - role: baztian.borgbackup_user
```

The role explicitly selects its user config directory for initialization and scheduled backups. Configs and Kuma
tokens are user-owned and mode `0600`; keep secrets in Ansible Vault. The service account needs read and traversal
access to sources and write access to local repositories. Configure SSH credentials and remote access separately.

The role does not remove old configs when a config name is removed. Remove obsolete files explicitly. The user timer
does not inherit the system unit's hardening or `systemd-inhibit` settings. Use separate repositories where possible.
When sharing a repository, coordinate schedules and archive retention: Borg locks do not prevent one job from pruning
another job's archives.

## Uptime Kuma

Kuma support requires borgmatic 1.8 or 1.9. Borgmatic 2 changes after-backup hook behavior and is not supported
for Kuma notifications. Set `borgbackup_user_kuma` as follows:

```yaml
borgbackup_user_kuma:
  url: "https://monitor.example.com"
  repositories:
    HOME:
      matcher: /srv/backup-repository
      token: "{{ vault_kuma_token }}"
```

Matchers use Bash glob patterns. The root-owned `/usr/local/sbin/borgmatic-user-kuma-push` is separate from the
system role's helper. The role leaves this shared user helper installed when Kuma is disabled for an account, so
other user jobs can continue to use it. Successful `create` actions notify Kuma, including manual backups.
Initialization and listing do not send notifications. The `on_error` hook also reports failures during prune,
compact, and check actions. A backup success can therefore be followed by a failure notification if a later check fails.

## Tests

Install Molecule, its Docker driver, and the required Ansible collections in a Python environment:

```sh
python3 -m pip install molecule molecule-docker
ansible-galaxy collection install community.docker ansible.posix
```

Run `molecule test` from this repository's root with Docker available. The default scenario tests Ubuntu 24.04 and Debian 13,
including configuration isolation, repository initialization, idempotence, backup creation, and Kuma hooks. It skips
live user-manager tasks because the containers do not run a user D-Bus.

Run `ansible-playbook -i INVENTORY tests/live-systemd.yml` against a disposable systemd VM to test linger, timer
activation, schedule changes, and a systemd-triggered backup. The playbook creates an account, directories, and a user
timer. Do not run it on a production host.

## Publishing

Create an Ansible Galaxy role import for the new GitHub repository and configure its `galaxy_api_key` repository
secret. The included publish workflow imports the role when a `v*.*.*` tag is pushed to the repository.

## License

MIT
