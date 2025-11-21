# Remnanode Ansible Bootstrap

Playbooks and a common role to bootstrap and harden new remnanode hosts: users and SSH keys, hardened SSHD, Tailscale, UFW, Docker, and Grafana Alloy. All sensitive data and caches live outside the repo so it’s safe to publish under `remnanode-ansible-bootstrap`.

## Requirements
- Ansible 9+ (tested with ansible-core 2.16 / ansible 12.2).
- Python 3.x on the control host.
- SSH access to targets with the provider’s initial credentials for the first run.

## Layout
- `inventory/hosts.example.yml`: inventory template (format doc: `inventory/README.md`).
- `group_vars/*.example.yml`: variable templates; real files live in `~/.ansible/common/inventory/group_vars/`.
- `playbooks/`: entrypoint `playbooks/remnanode-bootstrap.yml`.
- `roles/common/`: the primary configuration role.
- Local controller state (outside the repo): `~/.ansible/common/`
  - `local_vars/`: cached generated usernames/passwords per host.
  - `generated_keys/`: SSH keypairs generated for managed users.
  - `.vault_password`: Vault password file.

## Local setup (copy examples)
- Inventory lives at `~/.ansible/common/inventory/hosts.yml` (see `ansible.cfg`). Start with:
  - `cp inventory/hosts.example.yml ~/.ansible/common/inventory/hosts.yml`
  - `cp -R group_vars ~/.ansible/common/inventory/` and rename the examples (e.g., `group_vars/project_remnanode.example.yml` -> `group_vars/project_remnanode.yml`; `group_vars/all/vault.example.yml` -> `group_vars/all/vault.yml`, then encrypt with `ansible-vault encrypt`).
- Vault password file: create `~/.ansible/common/.vault_password` (0600) with your password.
- SSH config: optional starter at `ssh_config.example`; merge relevant blocks into `~/.ssh/config`.

## Secrets & Vault
- Vault file: `group_vars/all/vault.yml` (encrypted).
- Password file: `~/.ansible/common/.vault_password` (referenced in `ansible.cfg`).
- View/edit:
  - `ansible-vault view group_vars/all/vault.yml`
  - `ansible-vault edit group_vars/all/vault.yml`
- Add new secrets as `vault_common_*` and map them in `roles/common/defaults/main.yml` (or relevant vars files) before use.

## Running
- All hosts:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml`
- Single host/group:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml -l <host_or_group>`
- Check mode:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml -l <host> --check`
- First-time password-based bootstrap (provider creds):  
  ```
  ansible-playbook playbooks/remnanode-bootstrap.yml \
    -l <host> \
    --ask-pass \
    --ask-become-pass \
    -e "ansible_user=<provider_user> ansible_port=<provider_port> ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
  ```
  Use `--ask-become-pass` only if sudo prompts for a password; drop `ansible_ssh_common_args` after the first trust. Subsequent runs can use the generated user/key from `~/.ssh/config`.

## Inventory
- Host data under `all.hosts` plus grouping by `project`, `environment`, `region`, `role`.
- Put shared settings in `group_vars/<group>.yml`; keep secrets only in Vault.
- Quick validation:  
  `ansible-inventory --list -i inventory/hosts.yml`

## SSH & Users
- The role generates a random username/password and an ed25519 key locally, creates the user, deploys the key, and:
  - caches creds in `~/.ansible/common/local_vars/<host>.yml`;
  - copies the private key to `~/.ssh/<host>_<user>.key`;
  - updates `~/.ssh/config` for Host `<inventory_hostname>` (User, IdentityFile, Port, HostName) with comments.
- Subsequent runs reuse the cached username/password/key.

## Tailscale & Firewall
- Uses `vault_common_tailscale_authkey`.
- After setting SSH port and Tailscale, updates HostName/Port in local `~/.ssh/config`.
- UFW is configured for the custom SSH port and 443.

## Alloy
- `roles/common/templates/alloy/config.alloy.j2` reads settings from env vars set in `env.conf.j2` via `common_alloy_*` (sourced from Vault).
- Required values are validated by asserts in the `alloy` tasks.

## Adding a new host
1. Add the host to `inventory/hosts.yml` and include it in the needed groups (`project_*`, `env_*`, `region_*`, `role_*`).
2. Ensure required secrets are present in Vault.
3. For the first SSH access, use provider credentials (add a temporary Host block to `~/.ssh/config` if needed).
4. Run: `ansible-playbook playbooks/remnanode-bootstrap.yml -l <host>`.
5. Check the updated block in `~/.ssh/config` and the local cache in `~/.ansible/common/`.

## Useful commands
- Inspect SSH entries: `grep -A4 "Host <host>" ~/.ssh/config`
- View Vault contents: `ansible-vault view group_vars/all/vault.yml`
- Inventory graph: `ansible-inventory --graph -i inventory/hosts.yml`

## Security
- Do not commit `~/.ansible/common/` or any cached keys/creds (they’re outside the repo).
- To rotate the Vault password, update `~/.ansible/common/.vault_password` and rekey `group_vars/all/vault.yml`.
- Periodically clean old keys/caches if hosts are removed.
