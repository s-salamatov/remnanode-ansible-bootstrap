# Remnanode Ansible Bootstrap

Ansible playbook and common role to bootstrap and harden remnanode hosts. It patches Debian servers, creates a dedicated admin user with cached credentials, hardens SSH, joins Tailscale (with a manual path for geo-blocked regions), configures UFW, installs Docker, and deploys Grafana Alloy. Secrets and caches live outside the repo so it’s safe to publish on GitHub.

## What you get
- OS hardening: dist-upgrade, unattended-upgrades (no auto reboot), reboot flag, timezone + chrony, hostname alignment, MOTD noise disabled.
- Access management: per-host admin user with random username/password and ed25519 key generated locally; credentials cached under `~/.ansible/common/` and injected into `~/.ssh/config`.
- SSH hardening: moves SSH to `common_ssh_port` (default 2000), disables root/password logins, tests config before restart with rollback.
- Networking: Tailscale install (official script or manual tarball for `region_ru`), exit-node + LAN access, IP forwarding/GRO tweaks, SSH config updated with new host/port; UFW allows Tailscale SSH and 443 only.
- Platform bits: Docker Engine with user in `docker` group; Grafana Alloy installed via official script, configured for Grafana Cloud metrics/logs/remote config using vaulted values.
- Local controller state: inventory, generated credentials, keys, downloads, and vault password live in `~/.ansible/common/` (kept out of git).

## Requirements
- Control host: Python 3, Ansible 9+ (tested with ansible-core 2.16 / ansible 12.2), SSH access to targets.
- Collections: `community.general` and `community.crypto` (install with `ansible-galaxy collection install community.general community.crypto`).
- Targets: Debian-based host with sudo (password or passwordless) and initial provider SSH credentials for the first run.
- Secrets: Tailscale auth key plus Grafana Cloud IDs/URLs/API token (`vault_common_*`).

## Repository layout
- `playbooks/remnanode-bootstrap.yml`: entrypoint playbook applying `roles/common`.
- `roles/common/`: updates, benchmarks placeholder, timezone/chrony, hostname, user + SSH + local cache, Tailscale, MOTD, UFW, Docker, Alloy (config + systemd drop-ins).
- `inventory/README.md` and `inventory/hosts.example.yml`: inventory shape and examples.
- `group_vars/*.example.yml`: variable templates; real files live under `~/.ansible/common/inventory/`.
- `ansible.cfg`: points to `~/.ansible/common/inventory/hosts.yml` and vault password at `~/.ansible/common/.vault_password`.

## Prepare your control host (one time)
1. Install Ansible and collections:
   ```sh
   python3 -m pip install "ansible>=9"
   ansible-galaxy collection install community.general community.crypto
   ```
2. Clone this repository and `cd` into it.
3. Create local inventory and vars from examples:
   ```sh
   mkdir -p ~/.ansible/common/inventory
   cp inventory/hosts.example.yml ~/.ansible/common/inventory/hosts.yml
   cp -R group_vars ~/.ansible/common/inventory/
   mv ~/.ansible/common/inventory/group_vars/project_remnanode.example.yml ~/.ansible/common/inventory/group_vars/project_remnanode.yml
   mv ~/.ansible/common/inventory/group_vars/all/vault.example.yml ~/.ansible/common/inventory/group_vars/all/vault.yml
   ```
4. Create the vault password file referenced by `ansible.cfg`:
   ```sh
   echo "your-vault-password" > ~/.ansible/common/.vault_password
   chmod 600 ~/.ansible/common/.vault_password
   ```
5. Add secrets to the vault and encrypt:
   ```sh
   ansible-vault encrypt ~/.ansible/common/inventory/group_vars/all/vault.yml
   ansible-vault edit ~/.ansible/common/inventory/group_vars/all/vault.yml
   ```
   Populate `vault_common_tailscale_authkey` and all `vault_common_alloy_*` values (metrics/logs/fleet URLs, IDs, and API token).
6. (Optional) Merge snippets from `ssh_config.example` into `~/.ssh/config` if you want defaults for the very first login.

## Running the playbook
- All hosts:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml`
- Limit to a host or group:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml -l <host_or_group>`
- Check mode:  
  `ansible-playbook playbooks/remnanode-bootstrap.yml --check`
- First bootstrap using provider password/port:  
  ```sh
  ansible-playbook playbooks/remnanode-bootstrap.yml \
    -l <host> \
    --ask-pass \
    --ask-become-pass \
    -e "ansible_user=<provider_user> ansible_port=<provider_port> ansible_ssh_common_args='-o StrictHostKeyChecking=no'"
  ```
  Drop `--ask-become-pass` if sudo is passwordless. Remove `ansible_ssh_common_args` after the host key is trusted. Subsequent runs use the generated user/key stored locally.

## Inventory and variables
- Real inventory lives at `~/.ansible/common/inventory/hosts.yml`; follow `inventory/README.md` for the category layout (`project_*`, `env_*`, `region_*`, `role_*`).
- Store shared settings in `~/.ansible/common/inventory/group_vars/<group>.yml`; keep secrets only in the vaulted `group_vars/all/vault.yml`.
- Quick validation: `ansible-inventory --list -i ~/.ansible/common/inventory/hosts.yml` (or `--graph`).

## Generated access and local state
- Random admin username/password and an ed25519 key are generated per host and cached in `~/.ansible/common/local_vars/<host>.yml` and `~/.ansible/common/generated_keys/`.
- Private keys are also copied to `~/.ssh/<host>_<user>.key`, and `~/.ssh/config` is updated with host/port/user/identity and a password comment.
- SSH is moved to `common_ssh_port` (default 2000). Adjust that variable in your vars if you need a different port.
- Tailscale setup may adjust HostName/Port in your local SSH config to match the new address/port after the first run.

## Notes on services
- Tailscale: installs via official script or manual tarball flow for `region_ru` hosts, enables exit-node + LAN access, and ensures IP forwarding/GRO settings.
- Firewall: UFW allows the custom SSH port from the Tailscale range (100.64.0.0/10) and HTTPS on 443; removes the default OpenSSH rule.
- Docker: installs Docker Engine packages and adds the generated admin user to the `docker` group.
- Grafana Alloy: installs via Grafana script, writes `/etc/alloy/config.alloy` and a systemd env drop-in from vault values, and enables the service.

## Useful commands
- View inventory: `ansible-inventory --list -i ~/.ansible/common/inventory/hosts.yml`
- Inspect SSH entry: `grep -A4 "Host <host>" ~/.ssh/config`
- View/edit vault: `ansible-vault view ~/.ansible/common/inventory/group_vars/all/vault.yml`

## Security housekeeping
- Keep `~/.ansible/common/` and `~/.ssh/*` out of version control—they contain generated credentials and keys.
- Rotate the vault password by updating `~/.ansible/common/.vault_password` and rekeying `~/.ansible/common/inventory/group_vars/all/vault.yml`.
- Clean up cached keys/vars if you decommission hosts.
