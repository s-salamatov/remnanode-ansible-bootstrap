# Contributing to Remnanode Ansible Bootstrap

## Principles
- Keep changes idempotent: rerunning playbooks must not break state.
- No plaintext secrets in code. All sensitive values belong in `group_vars/all/vault.yml` as `vault_*`.
- Maintain inventory structure (`inventory/README.md`) and documentation alongside code changes.
- Local artifacts (`~/.ansible/common/...`) must never be committed.

## Before sending changes
1. Update docs if structure, variables, or role behavior changes.
2. Run sanity checks:
   - `ansible-inventory --list -i inventory/hosts.yml`
   - Optionally `ansible-playbook playbooks/remnanode-bootstrap.yml -l <host> --check`
3. Ensure new vars have no secret defaults; use placeholders + Vault mappings.
4. When adding roles/tasks, follow the existing layout (`roles/common/...`) and prefer Ansible modules over shell.

## Vault workflow
- Vault password is outside the repo: `~/.ansible/common/.vault_password`.
- Edit secrets: `ansible-vault edit group_vars/all/vault.yml`.
- Add secrets as `vault_<namespace>_<name>` and expose them via defaults/vars before use.

## SSH and local paths
- The role manages SSH keys/config automatically. Don’t hardcode paths; use `common_control_*` vars.
- If you change env-var–driven templates (Alloy), keep them in sync with `env.conf.j2`.

## YAML/Ansible style
- 2-space indents, no tabs.
- Use `register:` only when needed.
- Keep `debug` output minimal and relevant.

## Testing
- Minimum: `--check` on one host/group.
- When touching SSH/networking, verify handlers and idempotency of port/hostname updates in `~/.ssh/config`.
