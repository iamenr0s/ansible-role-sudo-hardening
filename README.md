[![Molecule](https://github.com/iamenr0s/ansible-role-sudo-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-sudo-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/sudo_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-sudo-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-sudo-hardening)

# Ansible Role: Sudo Hardening

This role audits and hardens sudo on Debian/Ubuntu, RHEL/AlmaLinux/RockyLinux and Fedora servers. It writes one validated drop-in file, `/etc/sudoers.d/99-linux-hardening`, and never edits existing sudoers rules, so nobody loses privileges.

Features
--------
- Audit mode by default: read-only, and reports every deviation. Changes need `linux_hardening_confirm_remediation: true`.
- Sets only four `Defaults`: `use_pty`, `logfile`, `timestamp_timeout` and `!visiblepw`.
- Validates the drop-in with `visudo -cf` before it reaches the live file.
- Reports `NOPASSWD` / `!authenticate` rules but never removes them.
- Installs sudo and adds the `#includedir` line when they are missing.
- Backs up the drop-in file before changing it.
- The only role in the `iamenr0s.linux_hardening` collection that writes sudoers settings.

## What it enforces
| Setting | Default |
|---|---|
| `Defaults use_pty` | on |
| `Defaults logfile="/var/log/sudo.log"` | on (`null` = unset) |
| `Defaults timestamp_timeout=15` | 15 min (`null` = unset) |
| `Defaults !visiblepw` | on |
| `#includedir /etc/sudoers.d` present in `/etc/sudoers` | added if missing (validated with `visudo -cf`) |

Reported only (never changed): `NOPASSWD` / `!authenticate` rules anywhere in sudoers, and a configuration that `visudo -c` rejects. Set `linux_hardening_sudo_allow_nopasswd: true` to accept passwordless rules (cloud images and automation accounts often need them).

## Variable names
Variables use the collection-wide `linux_hardening_*` prefix (shared with sibling roles) rather than `sudo_hardening_*`. Full list: `defaults/main.yml` and `meta/argument_specs.yml`. `linux_hardening_profile` is reserved here: this role applies the same Defaults for every profile.

## Modes
`audit` (default) is read-only and sets `linux_hardening_sudo_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`. After remediation only fixable findings fail the run.

## Safety
- The drop-in is written with `validate: visudo -cf %s`, so a syntax error never reaches the live file; `visudo -c` checks the whole configuration afterwards.
- Existing rules are untouched, so remediation cannot remove an admin's sudo rights. `use_pty` and `!visiblepw` can still affect unusual automation that runs sudo without a TTY or passes passwords visibly.
- The drop-in file name has no dot (sudo ignores names with dots or ending in `~`).

## Role Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_hardening_mode` | str | `audit` | audit is read-only. Choices: `audit`, `remediate`. |
| `linux_hardening_confirm_remediation` | bool | `false` | Must be true before remediate changes anything. |
| `linux_hardening_profile` | str | `cis_level1` | Shared collection variable. Reserved; this role applies the same Defaults for every profile. Choices: `baseline`, `cis_level1`, `cis_level2`. |
| `linux_hardening_backup` | bool | `true` | Back up files edited in place. |
| `linux_hardening_verify` | bool | `true` | Re-audit after remediation. |
| `linux_hardening_fail_on_error` | bool | `true` | Fail the play when audit finds deviations. |
| `linux_hardening_sudo_enabled` | bool | `true` | Run the role. |
| `linux_hardening_sudo_main_file` | path | `/etc/sudoers` | Main sudoers file. |
| `linux_hardening_sudo_dropin_dir` | path | `/etc/sudoers.d` | Drop-in directory. |
| `linux_hardening_sudo_dropin_name` | str | `99-linux-hardening` | Drop-in file name; must not contain a dot or end with a tilde. |
| `linux_hardening_sudo_use_pty` | bool | `true` | Defaults use_pty. |
| `linux_hardening_sudo_logfile` | raw | `/var/log/sudo.log` | Defaults logfile (path); null leaves it unset. |
| `linux_hardening_sudo_timestamp_timeout` | raw | `15` | Defaults timestamp_timeout in minutes (int); null leaves it unset. |
| `linux_hardening_sudo_disable_visiblepw` | bool | `true` | Defaults !visiblepw. |
| `linux_hardening_sudo_allow_nopasswd` | bool | `false` | Do not report NOPASSWD and !authenticate rules. |

**Check mode:** `remediate` with `--check --diff` shows what would change without writing anything; the post-remediation re-audit is skipped in check mode.

**Idempotent:** yes. A second `remediate` run reports no changes (the Molecule `idempotence` step checks this).

## Example
```yaml
- hosts: all
  become: true
  roles:
    - role: iamenr0s.sudo_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_sudo_allow_nopasswd: true   # automation account needs it
```

## Verification
Molecule `default` (remediate, idempotence, re-audit, mode 0440, `sudo -n true`, planted NOPASSWD rule untouched), `audit` (reports, writes nothing), `refuse`, `invalid` (a value that breaks syntax is rejected and no drop-in is written). Tested: AlmaLinux 9, Ubuntu 24.04, Debian 13. CI matrix: AlmaLinux 8–10, Rocky 8–10, Fedora 42–44, Debian 12–13, Ubuntu 22.04/24.04.

## Rollback
Delete `/etc/sudoers.d/99-linux-hardening` (a backup of any earlier version is kept when `linux_hardening_backup` is true).

## Known gaps
- Does not restrict who may use sudo (`sudo` group, `su` restriction, `pam_wheel`); that overlaps with `pam_hardening`.
- Passwordless rules are only reported; there is no automatic removal.
- `Defaults` set later in another file (sorting after `99-`) can still override these.
- Tested only with sudo 1.9.x from these three images; very old sudo (before 1.8) is untested. CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-sudo-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-sudo-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.sudo_hardening`
