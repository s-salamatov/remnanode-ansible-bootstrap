# Inventory Guidelines (Remnanode Ansible Bootstrap)

> Inventory is stored locally at `~/.ansible/common/inventory/hosts.yml`. This directory mirrors `inventory/` and `group_vars/` examples from the repo—copy and adapt before running playbooks.

## Format
- Use the YAML inventory stored at `inventory/hosts.yml`. INI files are no longer supported here.
- The root layout is `all -> hosts` (hostvars) plus `all -> children -> <category> -> children -> <group> -> hosts`.
- Keep host definitions alphabetized to reduce merge conflicts.

## Groups & Variables
- We maintain four category trees under `all.children`:
  - `project.*` (e.g., `project_remnanode`)
  - `environment.*` (`env_prod`, `env_dev`, …)
  - `region.*` (`region_us`, `region_ru`, …)
  - `role.*` (`role_proxy`, etc.)
- A host should appear inside each relevant category group. Example:
  ```yaml
  all:
    hosts:
      remnanode-proxy-de-1-prod:
        project: remnanode
        role: proxy
        env: prod
        region: de
    children:
      region:
        children:
          region_de:
            hosts:
              remnanode-proxy-de-1-prod: {}
  ```
- Store shared settings in `group_vars/<group>.yml`; only host-specific overrides (e.g., `region`, `env`) should live under `all.hosts`.

## Editing Workflow
1. Add or update host metadata inside `all.hosts` in `inventory/hosts.yml`.
2. Ensure the host is referenced inside every applicable category group (project, role, environment, region).
3. Document new shared settings in the matching `group_vars/<group>.yml`.
4. Use descriptive hostnames that encode role, region, and environment (`<stack>-<region>-<env>`).
5. Avoid embedding secrets; use `group_vars`, `host_vars`, or `ansible-vault` instead.

## Validation
- Run `ansible-inventory --list -i inventory/hosts.yml` after every edit to catch syntax or hierarchy mistakes.
- Use `ansible-inventory --graph -i inventory/hosts.yml` to visualize group membership if unsure where a host ends up.
