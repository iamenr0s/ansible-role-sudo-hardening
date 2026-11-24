# Changelog

## Unreleased
- Change: dropped the `linux_hardening_mode` assert that repeated `meta/argument_specs.yml` (which already rejects other values).
- Fix: `remediate` with `--check` no longer fails (the re-audit is skipped in check mode).
- Change: task names in sub-task files are prefixed.
- Docs: README adds a Role Variables table, the example uses the standalone role name with `become: true`, idempotency and check-mode notes.
- Test: the default Molecule scenario runs a `check` (dry-run remediate) step first.
- CI: every platform in `meta/main.yml` gates the release (no `continue-on-error` legs); lint runs the pinned pre-commit hooks; pre-commit adds gitleaks; actions are pinned by commit SHA.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
